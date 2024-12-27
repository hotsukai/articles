---
title: "AppRouterとTanstack Queryの併用のここまでの知見"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Next.js, Tanstack Query, React]
published: false
publication_name: sirok
---


# AppRouterとTanstack Queryの併用のここまでの知見


## [注意] ベストプラクティスはまだない
[Tanstack Queryのドキュメント](https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr)にもこう書かれており、
Next.jsとTanstack Queryの併用についてのベストプラクティスはTanstack Queryの公式も持ち合わせていません。各プロダクトの性質に合わせて使ってください。(どのような技術もそうですが。)
**本記事では、一意見として弊チームでのNext.jsプロジェクトへのTanstack Queryの導入理由とその結果を共有します。**

> It's hard to give general advice on when it makes sense to pair React Query with Server Components and not. If you are just starting out with a new Server Components app, we suggest you start out with any tools for data fetching your framework provides you with and avoid bringing in React Query until you actually need it. This might be never, and that's fine, use the right tool for the job!
:::details GPTによる翻訳
React QueryをServer Componentsと組み合わせるべきタイミングについて一般的なアドバイスをするのは難しいです。もし新しくServer Componentsアプリを始めたばかりなら、まずはフレームワークが提供しているデータ取得ツールを使い、React Queryを導入するのは実際に必要になった場合に限ることをお勧めします。その「必要」が訪れない場合もありますが、それでも問題ありません。適材適所でツールを使いましょう！
:::

## [おさらい] Next.jsのキャッシュ機構について

Next.js 13以降のAppRouterは、React Server Components（RSC）を前提に設計されており、
サーバーサイドでRequest Memoization, Data Cache, Full Route Cache という形で、レスポンス結果やレスポンス結果を使ったレンダリング結果を保持します。
キャッシュの破棄には`revalidateTag`, `revalidatePath`, fetchのrevalidateオプションなどを使います。

## [おさらい] Tanstack Queryのキャッシュ機構について
Tanstack Queryはご存知の通り、APIレスポンスを中心とした非同期の状態管理ライブラリです。
キャッシュはクライアントサイドで保持されます。

## モチベーション
Next.jsのキャッシュ機構もある中で、Tanstack Queryも選定した理由は下記です。

1. **より安全・簡単にキャッシュ管理したかった**
キャッシュの扱いはとてもセンシティブで、キャッシュの制御に問題があることを原因とした個人情報漏えい事故はニュースでも見かけると思います。
シロクショップではECサイトという特性上多くのお客様情報をお預かりしており、キャッシュを扱う際にも最大限の注意を払い実装しています。
また、弊チームではフロントエンドを専任としているエンジニアがおらず、またエンジニア自体も少人数であるため
これらのプロダクト背景から、管理が難しくなる多段キャッシュを避けること、学習・実装・レビューにおいて社内外に知見が溜まっておりコストが小さいこと、破壊的変更が少ないことなどの枯れた技術であることが重要となり、**サーバーサイドキャッシュはNext.jsでは持たずに、RailsのAPIサーバーのRedis、CDNなどで保持する**ことにしました。


またお客様ごとに表示する内容を変更することが多いです。


2. **よりきめ細やかな再レンダリング制御をしたかった**
Tanstack Queryではなく

この選択をすると、ユーザーの行動を受けとりServerActionsでrevalidateして、Revalidateされたキャッシュに依存するRSCが再レンダリングされる。
という、App Routerらしい挙動を捨ててしまうことにもなります。

現在、社内ツールなどより実験的な環境でApp Routerらしい挙動を検証しています。
知見が溜まり次第App Routerのキャッシュ機構を使うようことも検討していきたいと思います。

1. **柔軟なキャッシュ制御が必要な場合**  
   Next.jsでは、`revalidateTag`や`revalidatePath`を用いたキャッシュ管理が可能ですが、これが十分に細かく制御できない場面もあります。例えば、特定の条件でのみ再フェッチしたい、あるいはキャッシュの時間を動的に調整したい場合などでは、Tanstack Queryの詳細なキャッシュ制御機能が強みを発揮します。



## 実装例
実装方針は、initialDataを使う方法と、HydrationBoundaryを使う方法があります。
それぞれの
詳細な実装方法はTanstack Queryドキュメントの[Advanced Server Rendering](https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr)をご確認ください。

```bash
npm install @tanstack/react-query
```

### 2. QueryClientの初期設定

Tanstack Queryを利用するために、`QueryClient`をアプリケーション全体で利用できるよう設定します。

#### QueryProviderの作成

```typescript
// app/page.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import React from 'react';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = React.useState(() => new QueryClient());

  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}

// components/CartComponent.tsx
```

#### layout.tsxに組み込む

```typescript
// app/layout.tsx
import './globals.css';
import { QueryProvider } from './providers/QueryProvider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

### 3. デフォルト設定のカスタマイズ

```typescript
const [queryClient] = React.useState(
  () =>
    new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 1000 * 60 * 5, // データの再フェッチ間隔を5分に設定
          cacheTime: 1000 * 60 * 10, // キャッシュを保持する時間を10分に設定
          refetchOnWindowFocus: false, // フォーカス時の再フェッチを無効化
        },
      },
    })
);
```

## 実装例

### 1. サーバーサイドでの初期データ取得

```typescript
// app/posts/page.tsx
import React from 'react';

async function fetchPosts() {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts', { cache: 'force-cache' });
  return res.json();
}

export default async function PostsPage() {
  const posts = await fetchPosts();

  return (
    <div>
      <h1>Posts</h1>
      <ul>
        {posts.map((post: any) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 2. クライアントサイドでのデータ操作

```typescript
// app/posts/components/PostsList.tsx
'use client';

import { useQuery } from '@tanstack/react-query';

async function fetchPosts() {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts');
  return res.json();
}

export default function PostsList() {
  const { data: posts, isLoading } = useQuery(['posts'], fetchPosts);

  if (isLoading) {
    return <div>Loading...</div>;
  }

  return (
    <ul>
      {posts.map((post: any) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## 課題と解決策

### 1. キャッシュの競合
**解決策**: 明確な責務分担を設定し、サーバーサイドでは初期データ取得、クライアントサイドでは更新管理を行う。

### 2. クライアントサイドでのデータ取得のタイミング
**解決策**: `initialData`を活用してデータの二重フェッチを防ぐ。

### 3. エラー処理の一貫性
**解決策**: Tanstack Queryの`onError`ハンドラを利用し、エラーハンドリングを統一。

## まとめと次のステップ

1. **AppRouterとTanstack Queryの連携の意義**: サーバーとクライアントの役割分担を明確化し、パフォーマンスを向上。
2. **次のステップ**: サンプルアプリケーションの構築やリアルワールドのケースでの試行。

---

これで、Next.jsのAppRouterとTanstack Queryを併用する際の基本的なガイドが完成しました。ぜひ試してみてください！
