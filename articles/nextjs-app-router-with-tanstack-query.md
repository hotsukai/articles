---
title: "Next.js App RouterとTanStack Queryの連携パターン比較 - SSRとクライアント間の効果的なデータ共有"
emoji: "🔄"
type: "tech"
topics: ["nextjs", "react", "tanstackquery", "typescript", "frontend"]
published: false
publication_name: sirok
---

# Next.js App RouterとTanStack Queryの連携パターン比較

Next.js App RouterとTanStack Query（旧React Query）は、現代のReactアプリケーション開発において非常に強力なツールの組み合わせです。しかし、この2つを効果的に連携させるには、それぞれの仕組みを理解し、適切に統合する必要があります。

この記事では、Next.js App RouterとTanStack Queryを組み合わせる際の**複数の連携パターンを比較検討**し、それぞれの手法のメリット・デメリットを解説します。
基本的なセットアップ方法については公式ドキュメントのほか、日本語記事もいくつか存在するため、そちらを参考にしてください。
本記事では特に**サーバーサイドレンダリング（SSR）とクライアント間のデータ共有**に焦点を当て、実際のプロジェクトでどのパターンを選択すべきかの判断材料を提供します。

## TanStack Queryの基本的な流れ と SSR時のデフォルトの挙動

SPAでTanStack Queryを使用する場合、基本的な流れは以下のようになります：

1. ユーザーがアプリケーションにアクセスします
2. Reactコンポーネントがレンダリングされます
3. `useQuery`フックが実行され、キャッシュをチェックします
4. キャッシュにデータがなければ、APIリクエストを送信します
5. データが取得されると、コンポーネントが再レンダリングされます

```tsx
// SPAでの基本的なTanStack Queryの使用例
function UserProfile({ userId }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUserData(userId),
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return <div>{data.name}のプロフィール</div>;
}
```

このパターンでは、初回レンダリング時に「Loading...」が表示され、データが取得されると実際のコンテンツが表示されます。これは、クライアントサイドのみでレンダリングされる場合の標準的な挙動です。

重要なポイントとして、SSRを行う場合のuseQueryの挙動を理解する必要があります。

1. **SSR中（サーバー上）**では`useQuery`フックは処理されますが、`queryFn`は実行されません。これはTanStack Queryの内部実装において、データ取得の処理が`useEffect`フック内で行われているためです。
2. Reactの`useEffect`はクライアントサイドのライフサイクルフックであり、サーバーサイドレンダリング中には実行されません。

:::details より詳細な解説
| ステップ                                | 何が起きるか                                                                                             | どこで止まるか                                   |
| ----------------------------------- | -------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 1. **レンダー**（`<Component>` がサーバーで実行） | `useQuery` → `useBaseQuery` が `QueryObserver` を生成し、[`observer.getOptimisticResult()`](https://github.com/TanStack/query/blob/main/packages/react-query/src/useBaseQuery.ts#L93) で「キャッシュから取ってきたデータ(存在しない)」を返す | まだネットワーク処理なし                              |
| 2. **エフェクト登録**                      | `useBaseQuery` の `React.useEffect` 内で `observer.setOptions()` を呼び出すコードが *予約* される                   | **SSR では `useEffect` が実行されない**ため、ここが呼ばれない |
| 3. **フェッチ開始トリガ**                    | `observer.setOptions()` が呼ばれれば `executeFetch()` → `Query.fetch()` → `queryFn()` というチェーンが走る         | エフェクトが呼ばれないのでチェーンが開始しない                   |
| 4. **結果**                           | Query は `fetchStatus: 'paused'`（または `'idle'`）のまま。サーバー HTML にはデータが入らず、クライアント水和後に初めてフェッチが動く          | —                                         |

:::
2. **ハイドレーション後（クライアント上）**: クライアントサイドでアプリケーションがハイドレーションされると、`useEffect`フックが実行され、`useQuery`が再び評価されます。このとき`initialData`が提供されていなければ、キャッシュヒットしないため `queryFn`が実行されてデータ取得が始まります。

この仕組みにより、useQueryを含むクライアントコンポーネントがSSRされた場合でも、サーバーから返却されるHTMLは「Loading...」状態のものになります。このためページのメインコンテンツがない状態でSSRが終わってしまうため、SSの良さ(LCP, CLS)を享受しきれていません。
この問題は以降のセクションで解決していきます。


## AppRouter x TanstackQuery InitialData編

Next.js App RouterとTanStack Queryを組み合わせる最初のアプローチは、`initialData`を使用する方法です。この方法では、サーバーサイドで取得したデータをクライアントサイドのTanStack Queryに初期データとして渡します。

このアプローチは、App RouterとTanStack Queryを連携させる方法の中で**最もシンプルで実装が容易**です。
ユーザーは初回表示時から完全なコンテンツを見ることができ、クライアント側での「Loading...」表示を回避できます。

### initialDataの基本的な使い方

```tsx
// Server Component
async function getUser(userId) {
  const res = await fetch(`https://api.example.com/users/${userId}`);
  return res.json();
}

export default async function Page({ params }) {
  // サーバーサイドでデータを取得
  const initialUserData = await getUser(params.id);
  
  return <UserClient userId={params.id} initialData={initialUserData} />;
}

// Client Component
'use client';

import { useQuery } from '@tanstack/react-query';

function UserClient({ userId, initialData }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => getUser(userId),
    initialData,
  });
  
  return <div>{data.name}のプロフィール</div>;
}
```

この例では、サーバーコンポーネントでデータを取得し、それをクライアントコンポーネントに`initialData`として渡しています。
これにより、クライアントコンポーネントをSSRするときに、useQueryが初期値を持つ(=queryClientがキャッシュを持つ)状態で評価されるため、SSR結果のHTMLにもユーザー名が含まれます。
また、その後のクライアントサイドでのデータ更新はTanStack Queryによって管理されます。

### initialDataのメリットと注意点

**メリット**:
- サーバーからのデータが即座に利用可能になるため、初期ロード時の「Loading...」状態を回避できます
- シンプルな実装で、Server ComponentsとClient Componentsの連携が容易です

**注意点**:
- サーバーとクライアントで同じデータ取得ロジックを維持する必要があります
- **コンポーネントの階層が深い場合、データを下層コンポーネントに渡すための「バケツリレー」が必要になります**。

## AppRouter x TanstackQuery Hydration編

初期データをPropsとして渡す方法に加えて、より効率的なアプローチとして「ハイドレーション」があります。これは、サーバーサイドで取得したデータをTanStack Queryのキャッシュに直接注入し、Query Clientをダンプした文字列をHTMLに含めた(dehydrate)うえで、クライアントサイドでそれをパースし再利用する(hydrate)するものです。
https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr
### Hydrationの基本的な実装

```tsx
// page.tsx
import { dehydrate, HydrationBoundary } from '@tanstack/react-query'

export default async function Page({ params }) {
  // サーバーサイドでデータを取得し、queryClientに設定
  const queryClient = new QueryClient()
  await queryClient.prefetchQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUserData(userId),
  })
  
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserClient userId={params.id} />
    </HydrationBoundary>
  )
}

// UserClient.tsx
'use client';

import { useQuery } from '@tanstack/react-query';

export function UserClient({ userId }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUserData(userId),
  });
  
  return <div>{data?.name}のプロフィール</div>;
}
```

この実装でも、initialDataのときと同じく、SSR結果のHTMLにユーザー名が含まれます。

### Hydrationのメリットと注意点

**メリット**:
- データがキャッシュに直接注入されるため、コンポーネント間でデータを共有できます.
- バケツリレーが不要です。

**注意点**:
- 実装がやや複雑です
- サーバーで取得したデータとクライアントのキャッシュの整合性を保つための工夫が必要です
- **サーバー側とクライアント側でキャッシュキーやフェッチの実装がずれる可能性があります**。たとえば、サーバー側で使用したキャッシュキーとクライアント側で使用するキャッシュキーが異なると、同じデータに対して異なるキャッシュエントリが作成され、重複したデータ取得や意図しないUI動作の原因となります
- **この問題を避けるためには、次のセクションで説明するファクトリパターンが有効です**。サーバー側とクライアント側で共通のクエリ定義を使用することで、一貫性のある実装が可能になります

## うまく行うための工夫(ファクトリ)

Next.js App RouterとTanStack Queryを効果的に組み合わせるためには、いくつかの工夫が必要です。特に、クエリの定義とデータ取得ロジックを一元管理するファクトリパターンが有効です。

### クエリファクトリの実装

```tsx
// TODO
```

### ファクトリを使用したコンポーネント例

```tsx
// TODO
```

### ファクトリパターンのメリット

このアプローチには、以下のようなメリットがあります：

1. **DRYな実装**: クエリの定義とデータ取得ロジックが一元管理されるため、重複が減少します
2. **型安全性**: TypeScriptを使用することで、クエリキーとデータの型を厳密に定義できます


## まとめ

Next.js App RouterとTanStack Queryを組み合わせることで、高パフォーマンスで開発効率の高いWebアプリケーションを構築できます。

本記事では、サーバーサイドとクライアント間のデータ共有における主要な3つのアプローチを比較検討しました：

| 連携パターン | 実装の複雑さ | データの効率性 | コンポーネント間共有 | 適したプロジェクト規模 |
|-------------|------------|--------------|-----------------|-------------------|
| initialData | 低（シンプル） | 中（バケツリレー必要） | 難しい | 小〜中規模 |
| Hydration | 中〜高 | 高 | 容易 | 中〜大規模 |
| ファクトリパターン | 高（初期設定） | 高 | 最適 | 中〜大規模、長期保守 |

要点をまとめると：

- **初期データアプローチ**: シンプルで導入が容易ですが、大量のデータや複雑なアプリケーションでは非効率になる可能性があります。コンポーネント階層が深い場合にはバケツリレーの問題が発生します。
- **ハイドレーションアプローチ**: やや複雑ですが、より効率的でスケーラブルな実装が可能です。サーバー側とクライアント側でキャッシュキーの整合性管理が課題です。
- **ファクトリパターン**: クエリの定義とデータ取得ロジックを一元管理することで、保守性と開発効率を向上させます。特にハイドレーションアプローチと組み合わせることで効果を発揮します。

最終的には、アプリケーションの要件と制約に応じて適切なアプローチを選択することが重要です：

- 小規模なアプリケーションや、コンポーネント階層が浅い場合は、シンプルな初期データアプローチが適しています
- 大規模なアプリケーションや、多くのクエリを共有する必要がある場合は、ハイドレーションアプローチとファクトリパターンの組み合わせが効果的です
- チーム開発や長期保守が必要なプロジェクトでは、初期の複雑さがあってもファクトリパターンによる一貫性と型安全性のメリットが大きくなります

いずれのアプローチを選択する場合も、サーバーサイドとクライアントサイドの連携を意識し、データの整合性を保つことがNext.js App RouterとTanStack Queryを効果的に組み合わせる鍵となります。

## 参考資料

- [Next.js 公式ドキュメント - App Router](https://nextjs.org/docs/app)
- [TanStack Query 公式ドキュメント](https://tanstack.com/query/latest)
- [TanStack Query - SSR](https://tanstack.com/query/latest/docs/react/guides/ssr)
- [TanStack Query - ハイドレーション](https://tanstack.com/query/latest/docs/react/guides/ssr#using-hydration)
- [React Server Components の理解](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)
