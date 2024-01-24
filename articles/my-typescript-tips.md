---
title: "TypeScriptのよく使うけどよく忘れるやつのメモ"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript"]
published: true
---

# この記事は？
自分がよく使うけど毎回忘れて調べているやつのメモです。
順次追加していきます

## オブジェクトを作るときNull, undefinedな値をキーごと除く
```ts
const a = 1
const b = 'Hello World!!'
const c: string | null = null

const obj = {
  a, b,
  ...(c && { c }),
}
/**
 * {
 *    a: 1,
 *    b: 'Hello World!!',
 * }
*/
```

## Array.prototype.filterで特定の型だけを残す
```ts
const result = ['a', 'b', 'c', undefined, null]
  .filter(
    (item): item is Extract<typeof item, string> => typeof item === 'string'
  )
```
![](/images/my-typescript-tips/result1.png)



## Object.keysに型をつける
```ts
const engineer = {
  name: 'hogehoge',
  age: 20,
  profile: 'hello world',
  skillSets: ['Typescript', 'React']
};

const result2 = (Object.keys(engineer) as (keyof typeof engineer)[])
```
![](/images/my-typescript-tips/result2.png)
