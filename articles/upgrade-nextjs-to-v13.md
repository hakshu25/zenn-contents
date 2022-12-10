---
title: '個人ブログのNext.jsをv13にアップグレードする'
emoji: '🛠'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['nextjs']
published: true
---

2022 年 10 月末に[Next.js 13](https://nextjs.org/blog/next-13)がリリースされましたが、個人開発に使っている Next.js をまだアップグレードしていなかったので、実際に行った作業について書き留めておきます。

## アップグレードしたリポジトリとサイト

https://github.com/flyhighair/hakshu.dev
https://hakshu-dev.vercel.app/

## Next.js v13 の変更サマリー[^1]

- IE のサポート終了
- Node.js の EOL に伴い、サポート最小バージョンを v12 から v14 に引き上げ
- React の最小バージョンを v17.0.2 から v18.2.0 に引き上げ
- デフォルトのコンパイラーを [SWC](https://swc.rs/) に変更(swcMinify オプションのデフォルト値を true に変更)
  - ※ v12 から段階的に取り入れられていた
- 従来の `next/image` が `next/legacy/image` に置き換えられ、 `next/future/image` が `next/image` に置き換えられる
- `next/link` の子要素に手動で `a` タグを追加する必要がなくなった
  - 内包されるようになったため
- `next.config.js` の `target` オプションが無くなり、[Output File Tracing](https://nextjs.org/docs/advanced-features/output-file-tracing) に置き換わった

## 変更元・変更先バージョン

- v12.3.4 → v13.0.6
  - v12.2 以前から引き上げる場合は適宜 Upgrade Guide を見ながら段階的にアップグレードしていくのが安全

## 今回やらないこと

- まだ stable でない機能は対応しない
  - app Directory, Turbopack, @next/font
- SWC への変更の対応
  - v12 段階で元々移行していたので、今回特に必要な作業は無かった

## やったこと

以下のアップグレードガイドを基に進めました。

https://nextjs.org/docs/upgrading#upgrading-from-12-to-13

### 依存ライブラリのアップグレード

まず `next` を含めた周辺ライブラリをアップグレードします。

```bash
# npm
npm i next@latest react@latest react-dom@latest eslint-config-next@latest

# yarn
yarn add next@latest react@latest react-dom@latest eslint-config-next@latest
```

他にも例えば `react` のバージョンに依存するライブラリを使っている場合は、そちらも適宜更新しましょう。
変更サマリーにも記載しましたが、Node.js と React の最小バージョンが引き上がっているので注意が必要です。

## Codemods で破壊的変更があったコードを Migrate する

Next.js は 非推奨や破壊的変更があった機能のコードを互換性を保ちながら自動でアップグレードしてくれる Codemods というものを提供しています。

https://nextjs.org/docs/advanced-features/codemods

実行コマンドは以下になります。transform に対象の変換名を指定し、path に変換対象のファイルやディレクトリを指定するようになっています。

```bash
npx @next/codemod@latest <transform> <path>
```

今回 v13 に Upgrade する上で必要な transform は以下になっています。

- `new-link`
- `next-image-to-legacy-image`

注意点としては、基本的には安全に従来の挙動を保つようにコードを変換してくれるものなので、その部分を新規機能に置き換えたい場合は、自分でさらに変更する必要があります。

自分のブログは規模が小さいこともあり、ここまでの作業でアップグレードが完了しました。

## 最後に

Next.js v13 が発表された時、 app Directory や Turbopack がかなり大きい変更に思えたので、それ以外の部分のアップグレードも大変なのではと思っていました。しかし、特にハマるところもなくアップグレードできました。app Directory や Turbopack も今後触っていきたいです。

[^1]: 出典: [https://nextjs.org/docs/upgrading#v13-summary](https://nextjs.org/docs/upgrading#v13-summary)
