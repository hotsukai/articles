---
title: "Ruby on Rails + Vite + React でHMRをする方法"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vite", "rails", "react"]
published: false
---



## この記事は？
- Ruby on Rails + Vite でReactをHMRをするときにハマりまくったのでそのメモ。

:::message
わかりやすさのために記事中のコードは最低限に省略しています。
:::

## 環境
- RailsでERBを返却するMPA
- 諸事情により[Vite Rails](https://vite-ruby.netlify.app/guide/rails.html)は採用したくなかった。

## 結論
[公式Doc(バックエンドとの統合)](https://ja.vitejs.dev/guide/backend-integration.html)に記載されている。
HMRをするためには読み込みたいファイルの他に、
- `@vite/client`を読み込み
- (`@vitejs/plugin-react`を使っている場合には)`@react-refresh`を読み込んで初期化

する必要がある。

## 最終的な構成
```ts:vite.config.ts
import { Plugin, defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import * as fs from 'fs';

export default defineConfig(({ mode }) => ({
  plugins: [react({ jsxRuntime: 'classic' })],
  server: {
    host: 'dev.myapp.com',
    port: 5173,
    strictPort: true,
    https: {
      key: fs.readFileSync('path_to_cert/key.pem'),
      cert: fs.readFileSync('path_to_cert/cert.pem'),
    },
    hmr: {
      protocol: 'wss',
      port: 24678,
    }
  },
  build: {
    manifest: true,
    outDir: '../../public',
    rollupOptions: {
      input: {
        ...getJSEntryPoints(),
      },
    },
  },
}));
```


```rb:js_helper.rb(抜粋)
VITE_BASE_URL = 'https://dev.myapp.com:5173'

def render_vite_tag(file_path)
  tags = []
  tags << javascript_tag(
          "
            import RefreshRuntime from '#{VITE_BASE_URL}/@react-refresh'
            RefreshRuntime.injectIntoGlobalHook(window)
            window.$RefreshReg$ = () => {}
            window.$RefreshSig$ = () => (type) => type
            window.__vite_plugin_react_preamble_installed__ = true
            ",
          type: 'module'
        )
  tags << content_tag(:script, nil, type: 'module', src: "#{VITE_BASE_URL}/@vite/client")
  tags << content_tag(:script, nil, type: 'module', src: "#{VITE_BASE_URL}/#{file_path}")
  safe_join(tags)
end
```

```rb:sample.html.erb(抜粋)
<%= render_vite_tag('src/main.ts') %>
```


## ハマりどころ(備忘録)
### SSL化されたページからViteDevサーバにリクエストできない
```txt:ブラウザのコンソールのエラー
Mixed Content: The page at 'https://dev.myapp.jp/sample_page' was loaded over HTTPS, 
but requested an insecure script 'http://localhost:5173/@vite/client'. 
This request has been blocked; the content must be served over HTTPS.
```
RailsアプリがSSL化されていてページのプロトコルがHTTPSのとき(`https://dev.myapp.com`)、
viteサーバーのlocalhostはhttpのためスクリプトの読み込みができない。
ViteサーバーのHTTPSを有効にする([ドキュメント](https://ja.vitejs.dev/config/server-options.html#server-https))ことで解決。

### @vitejs/plugin-react がエラーになる
```txt:ブラウザのコンソールのエラー
hogehoge.tsx:7 Uncaught Error: @vitejs/plugin-react can't detect preamble. 
Something is wrong. See https://github.com/vitejs/vite-plugin-react/pull/11#discussion_r430879201
    at hogehoge.tsx:7:11
```
エラーメッセージで言及されている[プルリク](https://github.com/vitejs/vite-plugin-react/pull/11#discussion_r430879201)には、
クラスコンポーネントでは動かない・無名コンポーネントだと動かない・Vite4.1で解消される などと記載がある。
だが今回はいずれにも該当しなかった。


上述の[公式Doc(バックエンドとの統合)](https://ja.vitejs.dev/guide/backend-integration.html)に記載があるように、
```html
<script type="module">
  import RefreshRuntime from 'http://localhost:5173/@react-refresh'
  RefreshRuntime.injectIntoGlobalHook(window)
  window.$RefreshReg$ = () => {}
  window.$RefreshSig$ = () => (type) => type
  window.__vite_plugin_react_preamble_installed__ = true
</script>
```
を追加することでエラーを解消できた。

:::details 気づくまでの過程
```js:ビルドされたファイル(抜粋)
if (!window.__vite_plugin_react_preamble_installed__) {
    throw new Error("@vitejs/plugin-react can't detect preamble. Something is wrong. See https://github.com/vitejs/vite-plugin-react/pull/11#discussion_r430879201");
}
```
ここでエラーがthrowされていた。
`__vite_plugin_react_preamble_installed__`でググったらやっとドキュメントと出会えた...
だいぶ遠回りしてしまった感。
:::

