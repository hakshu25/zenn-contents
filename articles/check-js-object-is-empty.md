---
title: 'JavaScriptでObjectの空判定'
emoji: '🙆'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['JavaScript']
published: true
---

ふと Object の空判定ってどうやるんだっけと思ったので書き留めておきます。

# Object の空判定

空 Object と比較すればいいんじゃなかったっけと思ってたんですが、このやり方では比較結果は `false` になります。

```javascript
const obj = {};
console.log(obj === {}); // false
```

何故かというと、等価演算子は左辺と右辺両方のオペランドが Object である場合は同じオブジェクトを参照している場合にのみ `true` を返すようになっています。

> 両方のオペランドがオブジェクトである場合、同じオブジェクトを指している場合に限り true を返します。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Strict_equality

そのため以下のように `Object.keys` の `length` が 0 がどうか検証する必要があります。

```javascript
const emptyObj = {};
const isEmpty = Object.keys(emptyObj).length === 0 && emptyObj.constructor === Object;

console.log(isEmpty); // true
```

ここで `constructor` の Object 判定を合わせて行っている理由ですが、 `new Date()` などの場合も `keys` の `length` が 0 になり不十分な判定となるためです。
