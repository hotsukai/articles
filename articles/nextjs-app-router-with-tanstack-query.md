---
title: "Next.js App RouterとTanStack Queryの連携パターン比較 - SSRとクライアント間の効果的なデータ共有"
emoji: "🔄"
type: "tech"
topics: ["nextjs", "react", "tanstackquery", "typescript", "frontend"]
published: false
---

# Next.js App RouterとTanStack Queryの連携パターン比較

Next.js App RouterとTanStack Query（旧React Query）は、現代のReactアプリケーション開発において非常に強力なツールの組み合わせです。しかし、この2つを効果的に連携させるには、それぞれの仕組みを理解し、適切に統合する必要があります。

この記事では、Next.js App RouterとTanStack Queryを組み合わせる際の**複数の連携パターンを比較検討**し、それぞれの手法のメリット・デメリットを解説します。
基本的なセットアップ方法については公式ドキュメントのほか、日本語記事もいくつか存在するため、そちらを参考にしてください。
本記事では特に**サーバーサイドレンダリング（SSR）とクライアント間のデータ共有**に焦点を当て、実際のプロジェクトでどのパターンを選択すべきかの判断材料を提供します。

## そもそも SPA におけるTanstack Queryの挙動
TanStack Queryは元々、SPAでのデータ取得を効率化するために開発されました。SPAにおけるTanStack Queryの基本的な挙動を理解することが、Next.jsと組み合わせる際の理解の土台となります。

### SPAでのTanStack Queryの基本的な流れ

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

重要なポイントとして、Next.js App RouterでSSRを行う場合の挙動を理解する必要があります。'use client'ディレクティブで宣言されたクライアントコンポーネント内の`useQuery`は以下のように動作します：

1. **SSR中（サーバー上）**: `useQuery`フックは処理されますが、`queryFn`は実行されません。これはTanStack Queryの内部実装において、データ取得の処理が`useEffect`フック内で行われているためです。Reactの`useEffect`はクライアントサイドのライフサイクルフックであり、サーバーサイドレンダリング中には実行されません。サーバーコンポーネントでは`async/await`を直接使用してデータ取得が可能ですが、`use client`のコンポーネント内では、サーバー側でレンダリングされる際にuseEffectは実行されないため、queryFnも実行されないのです。
:::details より詳細な解説
| ステップ                                | 何が起きるか                                                                                             | どこで止まるか                                   |
| ----------------------------------- | -------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 1. **レンダー**（`<Component>` がサーバーで実行） | `useQuery` → `useBaseQuery` が `QueryObserver` を生成し、[`observer.getOptimisticResult()`](https://github.com/TanStack/query/blob/main/packages/react-query/src/useBaseQuery.ts#L93) で「キャッシュから取ってきたデータ(存在しない)」を返す | まだネットワーク処理なし                              |
| 2. **エフェクト登録**                      | `useBaseQuery` の `React.useEffect` 内で `observer.setOptions()` を呼び出すコードが *予約* される                   | **SSR では `useEffect` が実行されない**ため、ここが呼ばれない |
| 3. **フェッチ開始トリガ**                    | `observer.setOptions()` が呼ばれれば `executeFetch()` → `Query.fetch()` → `queryFn()` というチェーンが走る         | エフェクトが呼ばれないのでチェーンが開始しない                   |
| 4. **結果**                           | Query は `fetchStatus: 'paused'`（または `'idle'`）のまま。サーバー HTML にはデータが入らず、クライアント水和後に初めてフェッチが動く          | —                                         |

:::
1. **ハイドレーション後（クライアント上）**: クライアントサイドでアプリケーションがハイドレーションされると、`useEffect`フックが実行され、`useQuery`が再び評価されます。このとき`initialData`が提供されていなければ、`queryFn`が実行されてデータ取得が始まります。

この仕組みにより、SSRでサーバーコンポーネントがデータを取得していたとしても、そのデータを`initialData`として明示的に渡さない限り、クライアント側のコンポーネントでは一時的に「Loading...」状態が表示されることになります。これは以降のセクションで説明する連携パターンで解決すべき重要な課題です。

### キャッシュの管理

TanStack Queryの大きな強みは、キャッシュ管理です。一度取得したデータは、指定された時間だけキャッシュされ、同じデータを要求する別のコンポーネントや後続のレンダリングで再利用されます。

```tsx
// 複数のコンポーネントで同じクエリを使用する例
function UserHeader({ userId }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUserData(userId),
  });
  
  return <header>{data?.name}</header>;
}

function UserDetails({ userId }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUserData(userId),
  });
  
  return <div>{data?.bio}</div>;
}
```

この例では、同じユーザーデータを要求する2つのコンポーネントがありますが、実際のAPIリクエストは1回だけ行われます。

## そもそもAppRouterにおけるSSRの挙動

Next.js App Routerは、React Server Components（RSC）に基づいた新しいレンダリングモデルを導入しています。このモデルを理解することは、TanStack Queryと組み合わせる上で非常に重要です。

### Server ComponentsとClient Components

App Routerでは、デフォルトでコンポーネントがServer Componentsとして扱われます。これは、コンポーネントがサーバー上でレンダリングされ、結果のHTMLがクライアントに送信されることを意味します。

```tsx
// Server Component（デフォルト）
export default async function UserProfile({ userId }) {
  // サーバー上で直接データを取得
  const userData = await fetchUserData(userId);
  
  return <div>{userData.name}のプロフィール</div>;
}
```

対照的に、Client Componentsは'use client'ディレクティブを使用して明示的に指定されます。

```tsx
'use client';

// Client Component
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

### サーバーサイドレンダリングのパフォーマンス特性

サーバーサイドでデータを取得してレンダリングすることには、いくつかの重要なパフォーマンス特性があります：

1. **FCP (First Contentful Paint)とLCP (Largest Contentful Paint)のトレードオフ**：
   - **SPAの場合**: 初期HTMLはデータがなくても素早く送信されるため、FCPは早くなる可能性があります。しかし、そのHTMLには「Loading...」表示しか含まれておらず、実際の有意義なコンテンツ（LCP）はクライアントでのデータ取得後になります
   - **SSRの場合**: サーバーでのデータ取得を待ってからHTMLが送信されるため、TTFBとFCPは遅くなる可能性があります。しかし、初回から完全なコンテンツを含むHTMLが送信されるため、FCPとLCPの間のギャップが小さくなります

   ```tsx
   // SPA（クライアントサイドフェッチ）の場合
   'use client';
   function UserProfileSPA({ userId }) {
     const { data, isLoading } = useQuery({
       queryKey: ['user', userId],
       queryFn: () => fetchUserData(userId),
     });
   
     // FCPは早いが、これは意味のあるコンテンツではない
     if (isLoading) return <div>Loading...</div>;
   
     // 実際のLCPはこのレンダリング時
     return <div>{data.name}のプロフィール</div>;
   }
   
   // SSR（サーバーサイドフェッチ）の場合
   // Server Component
   async function UserProfileSSR({ userId }) {
     // サーバー側でデータ取得を待機（TTFBとFCPに影響）
     const userData = await fetchUserData(userId);
     
     // FCPとLCPが近い（初回から意味のあるコンテンツ）
     return <div>{userData.name}のプロフィール</div>;
   }
   ```

   実際のパフォーマンス影響は、データ取得の速度やサーバーの処理速度によって変わります：
   - サーバーがデータを高速に取得できる場合（キャッシュやデータベースが近い場合）、SSRのLCPはSPAより優れる傾向があります
   - データ取得に時間がかかる場合、SSRのTTFBとFCPが遅延し、ユーザー体験に影響する可能性があります
   - Next.jsはストリーミングレスポンスやSuspenseを使って、これらのトレードオフを軽減するアプローチも提供しています

2. **CLS (Cumulative Layout Shift)の削減**：
   - SPAでは、データロード後にコンテンツが表示されるときにレイアウトが変わり、CLS（累積レイアウトシフト）が発生します
   - SSRではコンテンツが最初から完全な形で表示されるため、レイアウトの突然の変化が減少し、ユーザー体験が向上します
   - 特に画像やリストなど、サイズが動的に変わる要素を含むUIで効果的です

このようなパフォーマンス特性を理解した上で、Server ComponentsとClient Componentsを連携させる際には、データをどのようにクライアント側で再利用するかが重要になります。TanStack Queryのようなクライアントサイドのデータ管理ライブラリと効果的に連携させるには、以下で説明する複数のアプローチがあります。

## AppRouter x TanstackQuery InitialData編

Next.js App RouterとTanStack Queryを組み合わせる最初のアプローチは、`initialData`を使用する方法です。この方法では、サーバーサイドで取得したデータをクライアントサイドのTanStack Queryに初期データとして渡します。

このアプローチは、App RouterとTanStack Queryを連携させる方法の中で**最もシンプルで実装が容易**です。特筆すべき点として、サーバーコンポーネントでデータを取得した時点ですでに意味のあるコンテンツが用意されており、クライアントコンポーネントはそのデータをそのまま利用できるため、前述のパフォーマンス特性の良い点を活かすことができます。ユーザーは初回表示時から完全なコンテンツを見ることができ、クライアント側での「Loading...」表示を回避できます。

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
    queryFn: () => fetchUserData(userId),
    initialData,
  });
  
  return <div>{data.name}のプロフィール</div>;
}
```

この例では、サーバーコンポーネントでデータを取得し、それをクライアントコンポーネントに`initialData`として渡しています。これにより、初回レンダリング時にはサーバーから渡されたデータが使用され、その後のデータ更新はTanStack Queryによって管理されます。

### initialDataのメリットと注意点

**メリット**:
- サーバーからのデータが即座に利用可能になるため、初期ロード時の「Loading...」状態を回避できます
- シンプルな実装で、Server ComponentsとClient Componentsの連携が容易です

**注意点**:
- サーバーとクライアントで同じデータ取得ロジックを維持する必要があります
- すべてのデータをプロップスとして渡す必要があるため、大量のデータの場合は非効率です
- 複数のコンポーネントで同じデータを使用する場合、重複したデータ取得が発生する可能性があります
- **コンポーネントの階層が深い場合、データを下層コンポーネントに渡すための「バケツリレー」が必要になります**。これにより、中間コンポーネントが不必要にデータを受け渡すだけの役割を担うことになり、コードの可読性と保守性が低下します

## AppRouter x TanstackQuery Hydration編

初期データをプロップスとして渡す方法に加えて、より効率的なアプローチとして「ハイドレーション」があります。これは、サーバーサイドで取得したデータをTanStack Queryのキャッシュに直接注入する方法です。

### Hydrationの基本的な実装

```tsx
// Provider.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }) {
  const [queryClient] = useState(() => new QueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}

// layout.tsx
import { Providers } from './providers';

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}

// page.tsx
export default async function Page({ params }) {
  // サーバーサイドでデータを取得
  const userData = await fetchUserData(params.id);
  
  return (
    <>
      <HydrateClient queryKey={['user', params.id]} dehydratedState={userData} />
      <UserClient userId={params.id} />
    </>
  );
}

// HydrateClient.tsx
'use client';

import { Hydrate as RQHydrate } from '@tanstack/react-query';

export function HydrateClient({ queryKey, dehydratedState }) {
  // サーバーからのデータをキャッシュに注入
  const dehydratedStateFormatted = {
    queries: [
      {
        queryKey,
        state: {
          data: dehydratedState,
          dataUpdateCount: 1,
          dataUpdatedAt: Date.now(),
          error: null,
          errorUpdateCount: 0,
          errorUpdatedAt: 0,
          fetchFailureCount: 0,
          fetchMeta: null,
          isInvalidated: false,
          status: 'success',
          fetchStatus: 'idle',
        },
      },
    ],
  };

  return <RQHydrate state={dehydratedStateFormatted} />;
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

この実装では、サーバーサイドで取得したデータを適切なフォーマットに変換し、`Hydrate`コンポーネントを使用してTanStack Queryのキャッシュに注入しています。

### Hydrationのメリットと注意点

**メリット**:
- データがキャッシュに直接注入されるため、コンポーネント間でデータを共有できます
- クライアントコンポーネントがデータ取得ロジックのみを持つことで、関心の分離が明確になります
- 大量のデータでも効率的に処理できます

**注意点**:
- 実装がやや複雑です
- サーバーで取得したデータとクライアントのキャッシュの整合性を保つための工夫が必要です
- キャッシュの有効期限や再検証ポリシーを適切に設定する必要があります
- **サーバー側とクライアント側でキャッシュキーやフェッチの実装がずれる可能性があります**。たとえば、サーバー側で使用したキャッシュキーとクライアント側で使用するキャッシュキーが異なると、同じデータに対して異なるキャッシュエントリが作成され、重複したデータ取得や意図しないUI動作の原因となります
- **この問題を避けるためには、次のセクションで説明するファクトリパターンが有効です**。サーバー側とクライアント側で共通のクエリ定義を使用することで、一貫性のある実装が可能になります

## うまく行うための工夫(ファクトリ)

Next.js App RouterとTanStack Queryを効果的に組み合わせるためには、いくつかの工夫が必要です。特に、クエリの定義とデータ取得ロジックを一元管理するファクトリパターンが有効です。

### クエリファクトリの実装

```tsx
// src/queries/userQueries.ts
import { QueryClient } from '@tanstack/react-query';

// クエリキーの定義
export const userKeys = {
  all: ['users'] as const,
  details: (id: string) => [...userKeys.all, id] as const,
  preferences: (id: string) => [...userKeys.details(id), 'preferences'] as const,
};

// データ取得関数
export const fetchUser = async (id: string) => {
  const res = await fetch(`https://api.example.com/users/${id}`);
  return res.json();
};

// クエリ定義のファクトリ
export const userQueries = {
  detail: (id: string) => ({
    queryKey: userKeys.details(id),
    queryFn: () => fetchUser(id),
  }),
};

// サーバーコンポーネント用のプリフェッチ関数
export const prefetchUser = async (queryClient: QueryClient, id: string) => {
  await queryClient.prefetchQuery(userQueries.detail(id));
};

// サーバーからのデータをハイドレーション用に変換する関数
export const hydrateUser = async (id: string) => {
  const data = await fetchUser(id);
  return {
    queries: [
      {
        queryKey: userKeys.details(id),
        state: {
          data,
          dataUpdateCount: 1,
          dataUpdatedAt: Date.now(),
          error: null,
          errorUpdateCount: 0,
          errorUpdatedAt: 0,
          fetchFailureCount: 0,
          fetchMeta: null,
          isInvalidated: false,
          status: 'success',
          fetchStatus: 'idle',
        },
      },
    ],
  };
};
```

### ファクトリを使用したコンポーネント例

```tsx
// page.tsx
import { hydrateUser } from '@/queries/userQueries';
import { HydrateClient } from '@/components/HydrateClient';
import { UserClient } from '@/components/UserClient';

export default async function Page({ params }) {
  const dehydratedState = await hydrateUser(params.id);
  
  return (
    <>
      <HydrateClient state={dehydratedState} />
      <UserClient userId={params.id} />
    </>
  );
}

// UserClient.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { userQueries } from '@/queries/userQueries';

export function UserClient({ userId }) {
  const { data } = useQuery(userQueries.detail(userId));
  
  return <div>{data?.name}のプロフィール</div>;
}
```

### ファクトリパターンのメリット

このアプローチには、以下のようなメリットがあります：

1. **DRYな実装**: クエリの定義とデータ取得ロジックが一元管理されるため、重複が減少します
2. **型安全性**: TypeScriptを使用することで、クエリキーとデータの型を厳密に定義できます
3. **テスト容易性**: ファクトリ関数は純粋関数として実装できるため、単体テストが容易です
4. **保守性の向上**: クエリの変更が一箇所で済むため、メンテナンスが容易です

### 実践的なヒント

1. **細分化されたキャッシュキー**: 適切な粒度でキャッシュキーを設計することで、キャッシュの無効化や更新を効率的に行えます
2. **サーバーとクライアントの共通ロジック**: できるだけフェッチロジックを共有し、整合性を保つようにしましょう
3. **エラーハンドリング**: サーバーサイドとクライアントサイドの両方で適切なエラーハンドリングを実装しましょう
4. **ステータスの同期**: ロード状態やエラー状態をサーバーとクライアント間で適切に同期させましょう

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
