---
title: "React.jsのSSRをTypeScriptで自前で実装してみた"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['React','SSR','TypeScript','reactrouter']
published: false
---

# この記事は？
- ReactのSSRを自前で実装して理解を深めようとした、せっかくなのでその記録を記事にもしておく。
- ReactRouterを使って複数ページのSSRをしている新しい日本語記事がなかった。

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
  1. 必要な情報をクライアント側からREST APIを叩いて取得。
  2. クライアントサイドレンダリング(CSR)する。

Next.jsなどで使われている標準的なSSRの挙動になったと思います。


## 実装(サーバーサイド編)
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
  - `getServerSideProps`: SSR時と, ページ遷移してCSR時に情報を取得する。( Next.jsを真似ました )

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
- Point③ではページのReactコンポーネントをHTMLにレンダーして、さらに文字列にしている。 
  - この時、DBから取得した情報(`hoge`とか)がHTMLに埋め込まれていることがわかる。
- `ReactRouter`の`StaticRouter`にurlを渡すことで、正しいページのコンポーネントをレンダリングしてもらうことができる。

Point③の結果
```html
  <div><h1>index page</h1><a href="/todos/id1"><div style="border-radius:4px;border:1px solid black;padding:1rem;margin:0.5rem 0"><h2>hoge</h2><p>hogehoge</p></div></a><a href="/todos/id2"><div style="border-radius:4px;border:1px solid black;padding:1rem;margin:0.5rem 0"><h2>fuga</h2><p>fugafuga</p></div></a><form><label for="title">title<input type="text" id="title" value=""/></label><label for="detail">detail<input type="text" id="detail" value=""/></label><button>submit</button></form></div></div>
```

- Point④では2つのことをやっている
  - クライアント側でReactをマウントできるようにする。
    - ページコンポーネントをReactをマウントするHTML Elementである`#root`配下に置く。
    - React側でページに埋め込んだ情報を扱えるように`#root`のdata attributesにページ情報のJSONを設定。
  - 完全なHTMLとしてレスポンスを返すこと。
    - メタ情報を追加など。
    - (HelmetとかでReact側でHeadを書き換えることはまだ考えていない..)

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
## 思ったこと
- `getServerSideProps`が重い場合、最初にユーザーに何かが表示されるまでの時間(FCP)が低下するので、スケルトンを表示してクライアント側でFetchするみたいに工夫したほうが良さそう。
- Link先を予めFetchしておくNext.jsすごい
- Next.jsしかり、RailsやPHPしかりSSRで毎回HTMLを生成するのリクエスト数が増えると大変だよね。
  - よく言われるように、内容が変わらないならば事前レンダリングしておいたほうがやっぱいいんだなぁ。

## 参考
https://ui.dev/react-router-server-rendering