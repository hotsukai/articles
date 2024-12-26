---
title: ""
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


# Next.jsのAppRouterとTanstack Queryを併用するプラクティス

## モチベーション

## initialDataを使う方法

## Hydration Boundaryを使う方法
> It's hard to give general advice on when it makes sense to pair React Query with Server Components and not. If you are just starting out with a new Server Components app, we suggest you start out with any tools for data fetching your framework provides you with and avoid bringing in React Query until you actually need it. This might be never, and that's fine, use the right tool for the job!

### GPTによる翻訳
React QueryをServer Componentsと組み合わせるべきタイミングについて一般的なアドバイスをするのは難しいです。もし新しくServer Componentsアプリを始めたばかりなら、まずはフレームワークが提供しているデータ取得ツールを使い、React Queryを導入するのは実際に必要になった場合に限ることをお勧めします。その「必要」が訪れない場合もありますが、それでも問題ありません。適材適所でツールを使いましょう！



## 導入

Next.js 13以降のAppRouterは、React Server Components（RSC）を前提に設計されており、サーバーサイドレンダリング（SSR）とクライアントサイドレンダリング（CSR）を効率的に組み合わせる強力なフレームワークです。この新しい構造では、以下が基本的な推奨事項となります。

- **サーバーサイドでのデータフェッチング**: RSCを活用し、`fetch`や`cache`を利用してデータを取得します。
- **Next.jsのキャッシュ戦略**: サーバーサイドでのデータ取得結果をキャッシュすることで、レスポンス性能とスケーラビリティを向上。

しかし、これだけでは全てのユースケースをカバーできません。特に、以下のような状況ではクライアントサイドでのデータ管理が求められることがあります。

### Tanstack Queryを使うべき場面

1. **クライアントサイド操作が多いケース**  
   例えば、フォーム入力後のリアルタイム更新やユーザーがインタラクションを頻繁に行うコンポーネントでは、RSCを介したデータフェッチが適切でない場合があります。このような場面では、クライアントサイドで直接データを管理できるTanstack Queryが有用です。

2. **柔軟なキャッシュ制御が必要な場合**  
   Next.jsでは、`revalidateTag`や`revalidatePath`を用いたキャッシュ管理が可能ですが、これが十分に細かく制御できない場面もあります。例えば、特定の条件でのみ再フェッチしたい、あるいはキャッシュの時間を動的に調整したい場合などでは、Tanstack Queryの詳細なキャッシュ制御機能が強みを発揮します。

3. **サーバーサイドでの複雑な依存関係を回避したい場合**  
   複雑なデータ取得ロジックや、クライアントサイドで依存性が動的に変化するケース（例: フィルタリングやページング）は、サーバー側での処理よりもクライアントサイドで処理する方が合理的です。この場合もTanstack Queryの柔軟性が役立ちます。

## 基礎設定

### 1. 必要なパッケージのインストール

Tanstack QueryとそのReact向けライブラリをインストールします。

```bash
npm install @tanstack/react-query
```

### 2. QueryClientの初期設定

Tanstack Queryを利用するために、`QueryClient`をアプリケーション全体で利用できるよう設定します。

#### QueryProviderの作成

```typescript
// app/providers/QueryProvider.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import React from 'react';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = React.useState(() => new QueryClient());

  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
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
