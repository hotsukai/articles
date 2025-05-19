---
title: "Next.js App RouterとTanStack Queryの連携の実践"
emoji: "🔄"
type: "tech"
topics: ["nextjs", "react", "TanstackQuery", "typescript", "frontend"]
published: false
publication_name: sirok
---

Next.js App RouterとTanStack Queryの運用を続けてきて、ある程度知見が溜まったのでまとめます。

# Next.js App RouterとTanStack Queryの連携パターン比較

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
// データの取得関数
async function fetchUser(userId) {
  const res = await fetch(`https://api.example.com/users/${userId}`);
  return res.json();
}

// SPAでの基本的なTanStack Queryの使用例
function UserProfile({ userId }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return <div>{data.name}のプロフィール</div>;
}
```

このパターンでは、初回レンダリング時に「Loading...」が表示され、データが取得されると実際のコンテンツが表示されます。これは、クライアントサイドのみでレンダリングされる場合の標準的な挙動です。

重要なポイントとして、SSRを行う場合のuseQueryの挙動を理解する必要があります。

1. **SSR中（サーバー上）**: `useQuery`フックは処理されますが、`queryFn`は実行されません。これはTanStack Queryの内部実装において、データ取得の処理が`useEffect`フック内で行われているためです。
Reactの`useEffect`はクライアントサイドのライフサイクルフックであり、サーバーサイドレンダリング中には実行されません。

:::details より詳細な解説
| ステップ                                          | 何が起きるか                                                                                                                                                                                                                                 | どこで止まるか                                                |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| 1. **レンダー**（`<Component>` がサーバーで実行） | `useQuery` → `useBaseQuery` が `QueryObserver` を生成し、[`observer.getOptimisticResult()`](https://github.com/TanStack/query/blob/main/packages/react-query/src/useBaseQuery.ts#L93) で「キャッシュから取ってきたデータ(存在しない)」を返す | まだネットワーク処理なし                                      |
| 2. **エフェクト登録**                             | `useBaseQuery` の `React.useEffect` 内で `observer.setOptions()` を呼び出すコードが *予約* される                                                                                                                                            | **SSR では `useEffect` が実行されない**ため、ここが呼ばれない |
| 3. **フェッチ開始トリガ**                         | `observer.setOptions()` が呼ばれれば `executeFetch()` → `Query.fetch()` → `queryFn()` というチェーンが走る                                                                                                                                   | エフェクトが呼ばれないのでチェーンが開始しない                |
| 4. **結果**                                       | Query は `fetchStatus: 'paused'`（または `'idle'`）のまま。サーバー HTML にはデータが入らず、クライアント水和後に初めてフェッチが動く                                                                                                        | —                                                             |

:::
2. **ハイドレーション後（クライアント上）**: クライアントサイドでアプリケーションがハイドレーションされると、`useEffect`フックが実行され、`useQuery`が再び評価されます。このとき`initialData`が提供されていなければ、キャッシュヒットしないため `queryFn`が実行されてデータ取得が始まります。

この仕組みにより、useQueryを含むクライアントコンポーネントがSSRされた場合でも、サーバーから返却されるHTMLは「Loading...」状態のものになります。このためページのメインコンテンツがない状態でSSRが終わってしまうため、SSRの良さ(LCP, CLS)を享受しきれていません。
この問題は以降のセクションで解決していきます。


## [App Router x Tanstack Query] InitialData編

Next.js App RouterとTanStack Queryを組み合わせる最初のアプローチは、`initialData`を使用する方法です。この方法では、サーバーサイドで取得したデータをクライアントサイドのTanStack Queryに初期データとして渡します。

このアプローチは、App RouterとTanStack Queryを連携させる方法の中で**最もシンプルで実装が容易**です。
ユーザーは初回表示時から完全なコンテンツを見ることができ、クライアント側での「Loading...」表示を回避できます。

https://tanstack.com/query/latest/docs/framework/react/guides/ssr#get-started-fast-with-initialdata

### initialDataの基本的な使い方

```tsx
// Server Component
export default async function Page({ params }) {
  // サーバーサイドでデータを取得
  const initialUserData = await fetchUser(params.id);
  
  return <UserClient userId={params.id} initialData={initialUserData} />;
}

// Client Component
'use client';

import { useQuery } from '@tanstack/react-query';

function UserClient({ userId, initialData }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
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

## [App Router x Tanstack Query] Hydration編

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
    queryFn: () => fetchUser(userId),
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
    queryFn: () => fetchUser(userId),
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
- **サーバー側とクライアント側でキャッシュキーやフェッチの実装がずれる可能性があります**。たとえばqueryKeyとqueryFnの組み合わせが、サーバー側とクライアント側でずれる実装ミスがあると、重複したデータ取得や意図しないUI動作の原因となります。
- **この問題を避けるためには、次のセクションで説明するファクトリパターンが有効です**。サーバー側とクライアント側で共通のクエリ定義を使用することで、一貫性のある実装が可能になります

## うまく行うための工夫(ファクトリ)

前述のように、Hydrationパターンではサーバー側とクライアント側でキャッシュキーやフェッチの実装がずれる実装ミスを起こしやすい問題がありました。この問題を解消するために、クエリの定義とデータ取得ロジックを一元管理するファクトリパターンが有効です。

### クエリファクトリの実装

```tsx
// src/lib/queryFactory.ts
import {
  QueryClient,
  QueryKey,
  QueryFunction,
  useQuery,
  UseQueryOptions,
  UseQueryResult,
} from '@tanstack/react-query'


export function createQueryFactory<
  TData = unknown,
  TError = unknown,
  // 呼び出し側が任意個の引数を渡せるように
  TArgs extends readonly unknown[] = readonly unknown[]
>(
  keyFn: (...args: TArgs) => QueryKey,
  fn: (...args: TArgs) => Promise<TData>
) {
  // ↓ 呼び出し時に実体化されるクロージャ
  return (...args: TArgs) => {
    const queryKey = keyFn(...args)
    const queryFn: QueryFunction<TData> = () => fn(...args)

    /** サーバー用：prefetchQuery ラッパー */
    const prefetch = (queryClient: QueryClient) =>
      queryClient.prefetchQuery({ queryKey, queryFn })

    /** クライアント用：useQuery ラッパー */
    const use = (
      options?: Omit<
        UseQueryOptions<TData, TError>,
        'queryKey' | 'queryFn'
      >
    ): UseQueryResult<TData, TError> =>
      useQuery({ queryKey, queryFn, ...options })

    return { queryKey, prefetch, use }
  }
}

// src/features/user/queries.ts
import { createQueryFactory } from '@/lib/queryFactory'
import { fetchUser } from './api'

export const fetchUserQuery = createQueryFactory(
  (userId: string) => ['user', userId], // keyFn
  (userId: string) => fetchUser(userId) // queryFn
);

```

### ファクトリを使用したコンポーネント例

```tsx
// app/user/[id]/page.tsx  (Server Component)
import { QueryClient, dehydrate, HydrationBoundary } from '@tanstack/react-query'
import { fetchUserQuery } from '@/features/user/queries'
import UserClient from './UserClient'

export default async function Page({ params }) {
  const queryClient = new QueryClient()
  await fetchUserQuery(params.id).prefetch(queryClient)

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      {/* Client Component 側へレンダリング */}
      <UserClient userId={params.id} />
    </HydrationBoundary>
  )
}


// app/user/[id]/UserClient.tsx  ('use client')
'use client'
import { fetchUserQuery } from '@/features/user/queries'

export default function UserClient({ userId }: { userId: string }) {
  const { data, isLoading } = fetchUserQuery(userId).use({
    staleTime: 60 * 1000,
  })

  if (isLoading) return <p>Loading...</p>
  return <pre>{JSON.stringify(data, null, 2)}</pre>
}
```

### ファクトリパターンのメリット

このアプローチには、以下のようなメリットがあります：

1. **DRYな実装**: クエリの定義とデータ取得ロジックが一元管理されるため、重複が減少します
2. **型安全性**: TypeScriptを使用することで、クエリキーとデータの型を厳密に定義できます

## パフォーマンスと適材適所の重要な考慮点

実際のアプリケーション開発ではさらに以下の点を考慮することが重要です：

### prefetchのリフトアップによるパフォーマンス最適化

prefetchの`await`をどこで行うかは重要なパフォーマンス最適化ポイントです。

```tsx
// 非効率なパターン：page.tsx全体がブロックされる
export default async function Page() {
  await fetchUserQuery(userId).prefetch(queryClient)
  await fetchPostsQuery(userId).prefetch(queryClient)
  
  return <Layout>...</Layout>
}
```

以下のようにリフトアップすることでFCPが改善できます：

```tsx
// 効率的なパターン：必要な部分だけブロック
export default function Page() {
  return (
    <Layout>
      <Suspense fallback={<UserSkeleton />}>
        <UserContainer />  {/* ここでawait prefetchする */}
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <PostsContainer />  {/* ここでawait prefetchする */}
      </Suspense>
    </Layout>
  )
}
```

データを使用するできる限り末端のコンポーネントで`await prefetch`して、親コンポーネントでSuspenseでラップすることで、ページ全体のレスポンスを高速化できます。

### 静的データと動的データの区別

すべてのデータをTanStack Queryで管理すべきではありません：

- **静的なマスターデータ**（CMS記事、商品データなど）：TanStack Queryを通す意味は少なく、SSR時のfetch結果をそのまま使用する方が効率的です
- **動的データ**（ユーザー操作による状態変化など）：TanStack Queryを活用すべき領域です

例えば、ECサイトであれば商品データは通常の`fetch`+`cache: 'force-cache'`を使用し、カートやお気に入り情報などユーザー固有のデータにTanStack Queryを適用するのが適切です。

## まとめ

Next.js App RouterとTanStack Queryを組み合わせることで、高パフォーマンスで開発効率の高いWebアプリケーションを構築できます。

本記事では、サーバーサイドとクライアント間のデータ共有における主要な3つのアプローチを比較検討しました：

| 連携パターン       | 実装の複雑さ   | 保守性                               |
| ------------------ | -------------- | ------------------------------------ |
| initialData        | 低（シンプル） | 低〜中（バケツリレーが必要）         |
| Hydration          | 中〜高         | 中（キャッシュキー整合性管理が必要） |
| ファクトリパターン | 高（初期設定） | 高（一元管理・型安全性）             |

Next.js App RouterとTanStack Queryの組み合わせはまだ発展途上の分野であり、ベストプラクティスが確立されている段階ではありません。
:::details 公式ドキュメント
https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr#data-ownership-and-revalidation
> It's hard to give general advice on when it makes sense to pair React Query with Server Components and not. If you are just starting out with a new Server Components app, we suggest you start out with any tools for data fetching your framework provides you with and avoid bringing in React Query until you actually need it. This might be never, and that's fine, use the right tool for the job!

#### GPTによる翻訳
React QueryをServer Componentsと組み合わせるべきタイミングについて一般的なアドバイスをするのは難しいです。もし新しくServer Componentsアプリを始めたばかりなら、まずはフレームワークが提供しているデータ取得ツールを使い、React Queryを導入するのは実際に必要になった場合に限ることをお勧めします。その「必要」が訪れない場合もありますが、それでも問題ありません。適材適所でツールを使いましょう！

:::
streamingのサポートがexperimental機能として実装されるなど、今後も進化が続くことが期待されます。

この記事で紹介したパターンも一例に過ぎないので、もし他の優れたプラクティスをご存知でしたら、ぜひコメントやSNSでフィードバックをいただければ幸いです。

## 参考資料

- [Next.js 公式ドキュメント - App Router](https://nextjs.org/docs/app)
- [TanStack Query 公式ドキュメント](https://tanstack.com/query/latest)
- [TanStack Query - SSR](https://tanstack.com/query/latest/docs/react/guides/ssr)
- [TanStack Query - ハイドレーション](https://tanstack.com/query/latest/docs/react/guides/ssr#using-hydration)
- [React Server Components の理解](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)
