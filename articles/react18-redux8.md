---
title: "Reactを18に上げたらReduxで詰まった話"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['react', 'redux', 'npm']
published: true
---

## この記事は？
- Reactのバージョンを18にあげたら、React-Reduxが動かなくなったよ。
- それに対処したよ。

## 結論
- React-Reduxをv8に上げる必要がある。
  - > This major version release updates `useSelector`, `connect`, and `<Provider>` for compatibility with React 18
- それによる破壊的変更はほぼない。
  - `@types/react-redux`を消す必要があるくらい。
- 該当[リリースノート](https://github.com/reduxjs/react-redux/releases/tag/v8.0.0)

## 以下ポエム
### この記事を書いた理由
React 18 / React Redux 7の環境で動かないことがわかりにくくかった。誰かの役に立てば🤝

- 下記はエラーログ。見てもreduxのバージョンが原因とわからなかった。
```sh
react.development.js:1465 Uncaught Error: Invalid hook call. Hooks can only be called inside of the body of a function component. This could happen for one of the following reasons:
1. You might have mismatching versions of React and the renderer (such as React DOM)
2. You might be breaking the Rules of Hooks
3. You might have more than one copy of React in the same app
See https://fb.me/react-invalid-hook-call for tips about how to debug and fix this problem.
    at resolveDispatcher (react.development.js:1465:1)
    at useMemo (react.development.js:1520:1)
    at Provider (Provider.js:11:29)
    at renderWithHooks (react-dom.development.js:16305:1)
    at mountIndeterminateComponent (react-dom.development.js:20074:1)
    at beginWork (react-dom.development.js:21587:1)
    at HTMLUnknownElement.callCallback (react-dom.development.js:4164:1)
    at Object.invokeGuardedCallbackDev (react-dom.development.js:4213:1)
    at invokeGuardedCallback (react-dom.development.js:4277:1)
    at beginWork$1 (react-dom.development.js:27451:1)
```
- [NPMのReact Reduxのページ](https://www.npmjs.com/package/react-redux/v/7.2.8)にReact18で動かないとの記載がなかった。
  > `React Redux 7.1 requires React 16.8.3 or later.`

