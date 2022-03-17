---
title: "React.jsのSSRをTypeScriptで自前で実装してみた"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['React','SSR','TypeScript','reactrouter']
published: true
---

# この記事は？
- ReactのSSRの理解を深めるために自前で実装してみました。
  - せっかくなのでその記録を記事にまとめました。
- ReactRouterを使って複数ページのSSRをしている新しい日本語記事がなかったというのも記事化の理由の一つです。

この記事のソースコードは[こちら](https://github.com/hotsukai/SSR-practice)です。

## 技術スタック
- フロントエンド
  - React.js (v17)
  - React-Router (v6)
  - TypeScript 
  - Webpack 
- バックエンド
  - Express

## 作ったもの
![](/images/react_ssr_scratch/about.gif)
- 初回リクエスト時
  1. サーバーサイドレンダリング(SSR)して完成されたHTMLをクライアントに返却。
  2. クライアントサイドで完成されたHTMLにイベントリスナーを設定(hydrate)
- ページ遷移時
  1. 新しいページで必要な情報をクライアント側からREST APIを叩いて取得。
  2. クライアントサイドレンダリング(CSR)する。

Next.jsなどで使われている標準的なSSRの挙動になったと思います。


## 実装
### `src/routes.ts`
サーバーサイド・クライアントサイドで共通のRouteを記述するファイル。

```ts
import { Request } from 'express'
import { VFC } from "react";

import mockDB from "./server/mockDB";
import { IndexPage } from "./client/pages";
import DetailPage from "./client/pages/detail";

export type PageProps = {
  path: string,
  buildPath: (id?: string) => string
  component: VFC,
  getServerSideProps: (req: Request) => unknown, 
}

// Point①
const routes: { [key: string]: PageProps } = {
  '/todos': {
    path: '/todos',
    buildPath: () => '/todos',
    component: IndexPage,
    getServerSideProps: (req: Request) => mockDB.findAll(),
  },
  '/todos/:id': {
    path: '/todos/:id',
    buildPath: (id: string) => '/todos/' + id,
    component: DetailPage,
    getServerSideProps: (req: Request) => mockDB.find(req.params.id),
  },
}

export default routes
```
- Point①がルート定義です。
  - `path`: URLと対応
  - `buildPath`: Linkなどで使うためにパスを生成する。
  - `component`: ページと対応するコンポーネント
  - `getServerSideProps`: ページで必要な情報を取得する関数です。SSR時とCSRでページ遷移するときに呼ばれます。( Next.jsを真似ました )

## 実装(サーバーサイド編)
### `src/server/index.ts`
サーバーサイドのエンドポイントになるファイルです。
SSRのエンドポイントだけでなくCSRするときに情報取得で使うAPIエンドポイントもあります。
```ts
// APIサーバー
import express, { Request } from 'express'
import routes, { PageProps } from 'routes';

import renderHtml from './renderer';

const app = express()

// 後述のクライアントサイドのJSや画像などが入る
app.use('/public', express.static('dist/public'));

// Point②
Object.keys(routes).forEach(key => {
  const route = routes[key] as PageProps

  app.get(route.path, async (req, res) => {
    const pageData = await route.getServerSideProps(req)
    const pageHtml = await renderHtml({ url: req.url, pageData })
    res.send(pageHtml)
  })
  app.get(`/api${route.path}`, async (req, res) => {
    const pageData = await route.getServerSideProps(req)
    res.json(pageData)
  })
})

app.get('/*', async (req, res) => {
  res.send("Page NotFound")
})

app.listen(3000)
```
- Point②ではさっき定義したRouteそれぞれにSSRエンドポイントと、CSR時の情報取得用のAPIエンドポイントを生やしています。

### `renderer.ts`
URLやページのデータからHTMLを生成します。
```ts
import App from "../client/App";
import React from "react";
import ReactDOMServer from "react-dom/server";
import { StaticRouter } from "react-router-dom/server";

type Props = {
  url: string;
  pageData: unknown;
};
const createHtml = async ({ url, pageData }: Props) => {
  // Point③
  const pageElemHtml = ReactDOMServer.renderToString(
    <StaticRouter location={url}>
      <App serverData={pageData} />
    </StaticRouter>
  );

  // Point④
  return `
  <html>
    <head>
      <title>SSR practice</title>
    </head>
    <body>
      <div id="root" data-react='${JSON.stringify(
        pageData
      )}'>${pageElemHtml}</div>
      <script src="/public/client.js"></script>
    </body>
  </html>`;
};
export default createHtml;
```
- Point③ではページのReactコンポーネントをHTMLにレンダーして、さらに文字列にしています。 
  - この時、DBから取得した情報(`hoge`とか)がHTMLに埋め込まれていることが下の結果からわかる。
- `ReactRouter`の`StaticRouter`にurlを渡すことで、正しいページのコンポーネントをレンダリングしてもらうことができる。
- クライアントサイドで実行されるJSは`dist/public/client.js`にビルドされるようになっている。

Point③の結果
```html
  <div><h1>index page</h1><a href="/todos/id1"><div style="border-radius:4px;border:1px solid black;padding:1rem;margin:0.5rem 0"><h2>hoge</h2><p>hogehoge</p></div></a><a href="/todos/id2"><div style="border-radius:4px;border:1px solid black;padding:1rem;margin:0.5rem 0"><h2>fuga</h2><p>fugafuga</p></div></a><form><label for="title">title<input type="text" id="title" value=""/></label><label for="detail">detail<input type="text" id="detail" value=""/></label><button>submit</button></form></div></div>
```

- Point④では2つのことをやっています。
  - クライアント側でReactをマウントできるようにする。
    - ページコンポーネントをReactをマウントするHTML Elementである`#root`配下に置く。
    - React側でページに埋め込んだ情報を扱えるように`#root`のdata attributesにページ情報のJSONを設定。
  - 完全なHTMLとしてレスポンスを返すこと。
    - メタ情報を追加など。
    - (HelmetとかでReact側でHeadを書き換えることはまだ考えていないです..)

Point④の結果
```html
  <html>
    <head>
      <title>SSR practice</title>
    </head>
    <body>
      <div id="root" data-react='[{"id":"id1","title":"hoge","detail":"hogehoge","isFinished":false},{"id":"id2","title":"fuga","detail":"fugafuga","isFinished":true}]'><div><h1>index page</h1><a href="/todos/id1"><div style="border-radius:4px;border:1px solid black;padding:1rem;margin:0.5rem 0"><h2>hoge</h2><p>hogehoge</p></div></a><a href="/todos/id2"><div style="border-radius:4px;border:1px solid black;padding:1rem;margin:0.5rem 0"><h2>fuga</h2><p>fugafuga</p></div></a><form><label for="title">title<input type="text" id="title" value=""/></label><label for="detail">detail<input type="text" id="detail" value=""/></label><button>submit</button></form></div></div>
      <script src="/public/client.js"></script>
    </body>
  </html>
```

## 実装(クライアントサイド編)
### `index.tsx`
クライアントサイドのエントリーポイントになるファイルです。
```tsx
import * as React from "react";
import ReactDOM from "react-dom";
import { BrowserRouter } from "react-router-dom";
import App from "./App";

// Point⑤
ReactDOM.hydrate(
  <BrowserRouter>
    <App />
  </BrowserRouter>,
  document.getElementById("root")
);
```
Point⑤では`ReactDOM.hydrate`により、root配下のHTML(SSRで作ったやつ)にイベントリスナーを設定してReactが動くようになります。

### `App.tsx`
Reactアプリ本体です。
後述の`PageWrapper`でページコンポーネントをWrapすることで
- ページコンポーネント側(`pages/index.tsx`, `pages/detail.tsx`)がSSRかCSRかに関心を保つ必要を無くしています。
- `PageWrapper`にkeyをもたせることでクライアント側でルートが変わったときに強制的に`PageWrapper`を再レンダリングしています。(`PageWrapper`はそのままでページコンポーネントだけをレンダリングできると良さそう)
```tsx
import React, { VFC } from "react";
import { Route, Routes } from "react-router-dom";
import routes from "../routes";
import PageWrapper from "./PageWrapper";

const App: VFC<{ serverData?: any }> = ({ serverData = null }) => {
  return (
    <Routes>
      {Object.keys(routes).map((key) => {
        const { path, component } = routes[key];
        return (
          <Route
            key={path}
            path={path}
            element={
              <PageWrapper
                key={path}
                PageComponent={component}
                serverData={serverData}
              />
            }
          />
        );
      })}
    </Routes>
  );
};

export default App;
```

### `PageWrapper.tsx`
最後に紹介するファイルです。
これが肝かつ、リファクタリングのやりがいのあるファイルだと思いますw
前述の通り、
- SSRでHTMLを作ってる時
- SSR後にクライアント側でReactをhydrateしてイベントリスナーをつけた時
- クライアント側でページ遷移した時(CSR時)

の3つの状態を吸収して配下のページたちは、ページで必要なデータとローディング中か否かだけを意識すればいいようにします。


```tsx
import axios from "axios";
import React, { useEffect, useRef, useState, VFC } from "react";
import { useLocation } from "react-router-dom";

type Props = {
  serverData?: any;
  PageComponent: VFC<{ data?: any; isLoading: boolean }>;
};
const PageWrapper: VFC<Props> = ({ serverData, PageComponent }) => {
  const [data, setData] = useState(() => {
    if (typeof document !== "undefined") { // クライアントサイド
      const dataPool = (document.getElementById("root") as HTMLElement).dataset
        .react;
      const unsafeData = dataPool ? JSON.parse(dataPool) : null;
      (document.getElementById("root") as HTMLElement).dataset.react = "";
      return unsafeData;
    } else {// サーバーサイド
      return serverData; 
    }
  });
  const isDataExist = !!data;

  const [isError, setIsError] = useState(false);
  const [isLoading, setIsLoading] = useState(!isDataExist);

  const { pathname } = useLocation();

  useEffect(() => {
    if (isDataExist) return;
    const f = async () => {
      setIsLoading(true);
      const result = await axios
        .get(`/api${pathname}`)
        .then((data) => data.data)
        .catch((error) => {
          console.warn(error);
          setIsError(true);
          return null;
        });
      setData(result);
      setIsLoading(false);
    };
    f();
  }, [pathname, isDataExist]);
  if (isError) return <p>エラーが発生しました。</p>;
  return <PageComponent data={data} isLoading={isLoading} />;
};

export default PageWrapper;
```
- Point⑥ではページで使うデータの初期化をしています。データの取得は以下のとおりです。

|              | SSRでHTMLを作ってる時         | hydrate時                | CSR時     |
| ------------ | ----------------------------- | ------------------------ | --------- |
| データ取得元 | propsで受け取った`serverData` | `#root`のdata attributes | APIを叩く |


## 思ったこと(ポエム)
- Link先を予めFetchしておくNext.jsすごい！！
- なんとなくNext.jsやGatsby.jsを使っていたけど自分で作ってみると学びがあるしフレームワークのありがたさを再認識できますね。
- 自前実装に限った話ではなくSSR全般に言えることですが、`getServerSideProps`が重い場合、最初にユーザーに何かが表示されるまでの時間(FCP)が低下するので、スケルトンを表示してクライアント側でFetchするみたいに工夫したほうが良さそうです。
- Next.js / Nuxt.jsのようなSSR+CSRでも、RailsやPHPのような古典的SSR(なんか名前があった気がする)でも言えるけど、リクエストのたびにHTMLを生成するのは大変...
  - よく言われるように、内容が変わらないならば事前レンダリングしておいたほうがやっぱいいんだなぁ。
  - ビルド時に一回だけ`getServerSideProps`をして結果をHTMLファイルにしたらSSGも実装できそう。

## 参考
https://ui.dev/react-router-server-rendering
https://ja.reactjs.org/docs/react-dom.html#hydrate
https://reactrouter.com/docs/en/v6/guides/ssr