---
title: "Next.js App RouterとTanStack Queryの組み合わせ方 - SSRとクライアントの連携"
emoji: "🔄"
type: "tech"
topics: ["nextjs", "react", "tanstackquery", "typescript", "frontend"]
published: true
---

# Next.js App RouterとTanStack Queryの組み合わせ方

Next.js App RouterとTanStack Query（旧React Query）は、現代のReactアプリケーション開発において非常に強力なツールの組み合わせです。しかし、この2つを効果的に連携させるには、それぞれの仕組みを理解し、適切に統合する必要があります。

この記事では、Next.js App RouterとTanStack Queryを組み合わせる方法について、サーバーサイドレンダリング（SSR）とクライアントサイドの連携に焦点を当てて解説します。

## モチベーション

なぜNext.js App RouterとTanStack Queryの組み合わせが重要なのでしょうか？

Next.jsのApp Routerは、Reactの最新機能であるServer ComponentsとClient Componentsを活用し、優れたパフォーマンスと柔軟なルーティングを提供します。一方、TanStack Queryは、サーバーデータの取得、キャッシュ、同期を効率的に管理するためのライブラリです。

これらを組み合わせることで、以下のようなメリットが得られます：

- **最適なユーザー体験**: SSRによる素早い初期表示とTanStack Queryによるデータの効率的な管理
- **開発効率の向上**: ボイラープレートコードの削減とデータ取得のロジックの簡素化
- **パフォーマンスの最適化**: 必要なデータのみを取得し、適切にキャッシュすることによる高速化

しかし、これらのツールを連携させるためには、それぞれの仕組みを理解し、適切に統合する必要があります。

## そもそも SPA における探索クエリの挙動

TanStack Queryは元々、SPAでのデータ取得を効率化するために開発されました。SPAにおけるTanStack Queryの基本的な挙動を理解することが、Next.jsと組み合わせる際の理解の土台となります。

### SPAでのTanStack Queryの基本的な流れ

SPAでTanStack Queryを使用する場合、基本的な流れは以下のようになります：

1. ユーザーがアプリケーションにアクセス
2. Reactコンポーネントがレンダリングされる
3. `useQuery`フックが実行され、キャッシュをチェック
4. キャッシュにデータがなければ、APIリクエストを送信
5. データが取得されると、コンポーネントが再レンダリングされる

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


## AppRouter x TanstackQuery InitialData編

Next.js App RouterとTanStack Queryを組み合わせる最初のアプローチは、`initialData`を使用する方法です。この方法では、サーバーサイドで取得したデータをクライアントサイドのTanStack Queryに初期データとして渡します。

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
- サーバーからのデータが即座に利用可能になるため、初期ロード時の「Loading...」状態を回避できる
- シンプルな実装で、Server ComponentsとClient Componentsの連携が容易

**注意点**:
- サーバーとクライアントで同じデータ取得ロジックを維持する必要がある
- すべてのデータをプロップスとして渡す必要があるため、大量のデータの場合は非効率
- 複数のコンポーネントで同じデータを使用する場合、重複したデータ取得が発生する可能性がある

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
- データがキャッシュに直接注入されるため、コンポーネント間でデータを共有できる
- クライアントコンポーネントがデータ取得ロジックのみを持つことで、関心の分離が明確になる
- 大量のデータでも効率的に処理できる

**注意点**:
- 実装がやや複雑
- サーバーで取得したデータとクライアントのキャッシュの整合性を保つための工夫が必要
- キャッシュの有効期限や再検証ポリシーを適切に設定する必要がある

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

1. **DRYな実装**: クエリの定義とデータ取得ロジックが一元管理されるため、重複が減少
2. **型安全性**: TypeScriptを使用することで、クエリキーとデータの型を厳密に定義できる
3. **テスト容易性**: ファクトリ関数は純粋関数として実装できるため、単体テストが容易
4. **保守性の向上**: クエリの変更が一箇所で済むため、メンテナンスが容易

### 実践的なヒント

1. **細分化されたキャッシュキー**: 適切な粒度でキャッシュキーを設計することで、キャッシュの無効化や更新を効率的に行える
2. **サーバーとクライアントの共通ロジック**: できるだけフェッチロジックを共有し、整合性を保つ
3. **エラーハンドリング**: サーバーサイドとクライアントサイドの両方で適切なエラーハンドリングを実装する
4. **ステータスの同期**: ロード状態やエラー状態をサーバーとクライアント間で適切に同期させる

## まとめ

Next.js App RouterとTanStack Queryを組み合わせることで、高パフォーマンスで開発効率の高いWebアプリケーションを構築できます。

要点をまとめると：

- **初期データアプローチ**: シンプルで導入が容易だが、大量のデータや複雑なアプリケーションでは非効率になる可能性がある
- **ハイドレーションアプローチ**: やや複雑だが、より効率的でスケーラブルな実装が可能
- **ファクトリパターン**: クエリの定義とデータ取得ロジックを一元管理することで、保守性と開発効率を向上させる

最終的には、アプリケーションの規模や要件に応じて適切なアプローチを選択することが重要です。小規模なアプリケーションでは初期データアプローチで十分かもしれませんが、大規模で複雑なアプリケーションではハイドレーションアプローチとファクトリパターンの組み合わせが効果的でしょう。

## 参考資料

- [Next.js 公式ドキュメント - App Router](https://nextjs.org/docs/app)
- [TanStack Query 公式ドキュメント](https://tanstack.com/query/latest)
- [TanStack Query - SSR](https://tanstack.com/query/latest/docs/react/guides/ssr)
- [TanStack Query - ハイドレーション](https://tanstack.com/query/latest/docs/react/guides/ssr#using-hydration)
- [React Server Components の理解](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)
