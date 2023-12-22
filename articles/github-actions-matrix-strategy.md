---
title: "GitHub Actions の matrix strategy で job を並列に実行する"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [githubactions]
published: true
---

## はじめに

最近 GitHub Actions の CI を高速化するために、job の並列実行を導入しました。そこで使った `jobs.<job_id>.strategy.matrix` について紹介します。

## matrix とは

`matrix` は、job を複数の変数の組み合わせで実行するための機能です。例えば、以下のように `matrix` に OS や Node.js のバージョンを指定すると、それぞれの組み合わせで処理が実行されます。

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20]
        env: [dev, prod]
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: ${{ matrix.node }}
      - run: pnpm install
      - run: pnpm build ${{ matrix.env }}
```

この場合の組み合わせは、以下になります。

- `{ node: 18, env: dev }`
- `{ node: 18, env: prod }`
- `{ node: 20, env: dev }`
- `{ node: 20, env: prod }`

指定したい組み合わせごとに job を分けて並列化しなくても、`matrix` を使うことで、1 つの job の記述で並列化を行うことができます。
なお、`strategy` は、`matrix` を使うための機能であるため、セットで使うと覚えておきましょう。
ちなみに、`matrix` ではワークフローの実行ごとに最大 256 個の job を生成できます。つまり、これが `matrix` で指定できる最大の組み合わせ数になります。

## 細かいところに手が届く便利オプション

### fail-fast

`fail-fast` は、`matrix` 内の job が失敗した時のエラー制御を行うためのオプションです。`matrix` と同様に `strategy` 直下に指定します。
デフォルト値は `true` であり、この場合は `matrix` 内のいずれかの job が失敗した時に同じ `matrix` 内の進行中もしくはキューに入っている job はキャンセルされます。 `fail-fast` を `false` にすると、job の成功・失敗に関わらず `matrix` 内のすべての job が完了するまで実行します。`false` でいずれかの job が失敗した場合でも、ワークフロー全体のステータスは失敗となるため、失敗に気づけないということはありません。

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node: [18, 20]
        env: [dev, prod]
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: ${{ matrix.node }}
      - run: pnpm install
      - run: pnpm build ${{ matrix.env }}
```

設定値ですが、変更を加えた時に、失敗する job を全て把握したい場合は `false` に、失敗時にすぐワークフローを止めたい場合は `true` で指定すると良さそうです。

### max-parallel

`max-parallel` は、`matrix` 内の job の最大実行数を指定するオプションです。`matrix`, `fail-fast` と同様に `strategy` 直下に指定します。指定しなければ、ランナーが実行可能な最大数まで job を並列実行します。何かの理由で、同時に実行する job の数を制限したい場合に指定できます。

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      max-parallel: 2
      matrix:
        node: [18, 20]
        env: [dev, prod]
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: ${{ matrix.node }}
      - run: pnpm install
      - run: pnpm build ${{ matrix.env }}
```

## 最後に

`matrix` を使うことで、簡潔な記述でお手軽に job を並列実行できました。なお、Private Repository で過剰に並列化しすぎると以下の記事のように課金で痛い目を見てしまうのでご利用は計画的に。(自分も一気に並列化してハマりかけました...)

https://khasegawa.hatenablog.com/entry/2022/11/14/100000

## 参考

https://docs.github.com/ja/actions/using-jobs/using-a-matrix-for-your-jobs
