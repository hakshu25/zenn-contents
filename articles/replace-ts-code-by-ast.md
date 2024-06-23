---
title: "大量のコードを TypeScript の AST で一気に置き換えよう"
emoji: "🧙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "tsmorph", "ast"]
published: true
---

## はじめに

最近数百ファイルに同内容の変更を加える必要があったのですが、ファイルに応じて動的に変わる部分があることから、単純な置換では対応できずどうしようかと悩んでいました。
そんな時 TSKaigi の以下の発表を思い出し、TypeScript の AST を使って一気に置き換える方法を試してみました。

https://speakerdeck.com/yanaemon/typescript-nochou-xiang-gou-wen-mu-woyong-ita-shu-bai-wochao-eru-api-noda-gui-mo-rihuakutaringuzhan-lue

## 変換対象のコード

今回は以下のようにコンポーネント単位で用意している Storybook ファイルを対象としました。

```typescript:button.before.stories.tsx
import type { StoryObj } from '@storybook/react';
import Button from './Button';

export default {
  component: Button,
};
...
```

このコードへ以下のように型注釈を追加したものが完成形です。

```typescript:button.after.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import Button from './Button';

const meta: Meta<typeof Button> = {
  component: Button,
};
export default meta;
...
```

変換したい箇所と内容をまとめると以下の通りです。

1. `@storybook/react` の `import` 文に `Meta` を追加する
2. `export default` のオブジェクトを meta 変数に変更し、型注釈を設定する
    - 相対パスで `import` しているコンポーネントの型を指定する
3. `export default` を meta オブジェクトに変更する

コンポーネントの型が動的に変わりますが、変更内容は共通しているため、AST を使って一気に変換できそうです。
変換には [ts-morph](https://ts-morph.com/) というライブラリを使いました。

## ts-morph とは

TypeScript の compiler API をラップして AST 解析をできるライブラリです。解析の上、コードの変更や新規作成もできるので、今回のようなコードの変換に使えますね。

## ts-morph を使ってコードを書き換える

まずは ts-morph をインストールします。

```bash
# npm
npm i -D ts-morph

# yarn
yarn add -D ts-morph

# pnpm
pnpm add -D ts-morph
```

試行錯誤しながらですが以下のようなスクリプトを作成しました。

```typescript:transform.ts
import { ImportDeclaration, ObjectLiteralExpression, Project, SourceFile, SyntaxKind } from 'ts-morph';

const project = new Project({
  tsConfigFilePath: './tsconfig.json',
});

const unshiftMetaImport = (storyImport: ImportDeclaration): void => {
  const namedImports = storyImport.getNamedImports();
  const hasMetaImport = namedImports.some((namedImport) => namedImport.getName() === 'Meta');

  if (!hasMetaImport) {
    storyImport.insertNamedImport(0, 'Meta');
  }
};

const getDefaultExportObject = (sourceFile: SourceFile): ObjectLiteralExpression | undefined => {
  const defaultExport = sourceFile.getDefaultExportSymbol();
  return defaultExport?.getValueDeclaration()?.getFirstDescendantByKind(SyntaxKind.ObjectLiteralExpression);
};

const replaceDefaultExport = (
  defaultExportObject: ObjectLiteralExpression,
  componentImport: ImportDeclaration,
): void => {
  const componentName = componentImport.getDefaultImport()?.getText();
  const exportAssignment = defaultExportObject.getParentIfKind(SyntaxKind.ExportAssignment);
  if (exportAssignment) {
    exportAssignment.replaceWithText(
      `const meta: Meta<typeof ${componentName}> = ${defaultExportObject.getFullText()};\nexport default meta;`,
    );
  }
};

const sourceFiles = project.addSourceFilesAtPaths('src/Components/**/*.stories.tsx');
sourceFiles.forEach((sourceFile) => {
  const storyImport = sourceFile.getImportDeclaration('@storybook/react');
  if (storyImport) {
    unshiftMetaImport(storyImport);
  }

  const defaultExportObject = getDefaultExportObject(sourceFile);
  if (!defaultExportObject) {
    console.error(`${sourceFile.getFilePath()}: Default export object is replaced.`);
    return;
  }

  const componentImport = sourceFile.getImportDeclarations().find((importDeclaration) => {
    const moduleSpecifier = importDeclaration.getModuleSpecifierValue();
    return moduleSpecifier.startsWith('./') && importDeclaration.getDefaultImport();
  });
  if (!componentImport) {
    console.error(`${sourceFile.getFilePath()}: Component import not found.`);
    return;
  }

  replaceDefaultExport(defaultExportObject, componentImport);

  sourceFile.saveSync();
});
```

### スクリプトについて

#### 対象のファイルを指定する

```typescript
const project = new Project({
  tsConfigFilePath: './tsconfig.json',
});
```

ts-morph ではまず `Project` クラスを使って、指定した tsconfig.json を元に AST を解析できるようにします。

```typescript
const sourceFiles = project.addSourceFilesAtPaths('src/components/**/*.stories.tsx');
```

デフォルトでは tsconfig.json でコンパイル対象としたファイルを解析対象とするため、特定のファイルに絞りたい場合は `addSourceFilesAtPaths` メソッドで指定します。
今回は Storybook ファイルに限定しています。

#### 変換処理

対象のファイルを設定したら、それぞれのファイルに対して変換処理をかけていきます。
おさらいですが、今回の変換内容は以下の通りです。

>1. `@storybook/react` の `import` 文に `Meta` を追加する
>2. `export default` のオブジェクトを meta 変数に変更し、型注釈を設定する
    - 相対パスで `import` しているコンポーネントの型を指定する
>3. `export default` を meta オブジェクトに変更する

ここから AST で対象箇所を取得していく必要がありますが、ここで役立つのが TypeScript AST Viewer です。

https://ts-ast-viewer.com/

このツールに TypeScript のコードを入力すると以下のように AST が可視化されるので、対象箇所が AST 上でどのように表現されているかを確認できます。

![](/images/replace-ts-code-by-ast/ts-ast-viewer-sample.png)

まず、`@storybook/react` の `import` 文、AST 上では `ImportDeclaration` として表現されるものを取得します。
その `NamedImports` 部分に `Meta` を追加します。

```typescript
const unshiftMetaImport = (storyImport: ImportDeclaration): void => {
  const namedImports = storyImport.getNamedImports();
  const hasMetaImport = namedImports.some((namedImport) => namedImport.getName() === 'Meta');

  if (!hasMetaImport) {
    storyImport.insertNamedImport(0, 'Meta');
  }
};

...

sourceFiles.forEach((sourceFile) => {
  const storyImport = sourceFile.getImportDeclaration('@storybook/react');
  if (storyImport) {
    unshiftMetaImport(storyImport);
  }
...
```

次に、`export default` のオブジェクトを取得し、型注釈を追加した meta 変数に変換します。
コードの詳細は割愛しますが、やることは変わらず、AST 上での表現を確認しながら対象箇所を取得し、変換する...の繰り返しです。
ChatGPT や GitHub Copilot でサンプルコードを生成して試していくのが効率的ですが、AST に対応するメソッドを探す場合は以下の詳細ページが参考になります。
https://ts-morph.com/details/index

```typescript
const getDefaultExportObject = (sourceFile: SourceFile): ObjectLiteralExpression | undefined => {
  const defaultExport = sourceFile.getDefaultExportSymbol();
  return defaultExport?.getValueDeclaration()?.getFirstDescendantByKind(SyntaxKind.ObjectLiteralExpression);
};

const replaceDefaultExport = (
  defaultExportObject: ObjectLiteralExpression,
  componentImport: ImportDeclaration,
): void => {
  const componentName = componentImport.getDefaultImport()?.getText();
  const exportAssignment = defaultExportObject.getParentIfKind(SyntaxKind.ExportAssignment);
  if (exportAssignment) {
    exportAssignment.replaceWithText(
      `const meta: Meta<typeof ${componentName}> = ${defaultExportObject.getFullText()};\nexport default meta;`,
    );
  }
};

sourceFiles.forEach((sourceFile) => {
...
  const defaultExportObject = getDefaultExportObject(sourceFile);
  if (!defaultExportObject) {
    console.error(`${sourceFile.getFilePath()}: Default export object is replaced.`);
    return;
  }

  const componentImport = sourceFile.getImportDeclarations().find((importDeclaration) => {
    const moduleSpecifier = importDeclaration.getModuleSpecifierValue();
    return moduleSpecifier.startsWith('./') && importDeclaration.getDefaultImport();
  });
  if (!componentImport) {
    console.error(`${sourceFile.getFilePath()}: Component import not found.`);
    return;
  }

  replaceDefaultExport(defaultExportObject, componentImport);
...
```

最後に `sourceFile.saveSync()` で変更を保存します。

```typescript
...
  sourceFile.saveSync();
});
```

### 実行

スクリプトを実行すると、実際に変換されます。

```bash
npx tsx transform.ts
```

## まとめ

ChatGPT に聞きながら試していきましたが 30 分ほどでスクリプトを完成させ、一気に変換できました。
AST を使うことで効率的に対応できるので、今後も積極的に活用していきたいです。
