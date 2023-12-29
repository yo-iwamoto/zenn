---
title: 'Vitest Workspace でテストごとに異なる実行環境を使う'
emoji: '🦔'
type: 'tech'
topics: ['vitest']
published: true
---

# Vitest の Workspace について

Vitest には v0.30.0 から Workspace 機能が追加されていて、共通設定などを一元管理できるようになっています。

https://vitest.dev/guide/workspace.html

これはドキュメントにもある通り、一応は monorepo のビルトインサポートという位置付けなのですが、実際には `Workspace` というのは特にディレクトリを指しているというわけではなく、独立したテスト実行単位のようなものです。

そのため、どちらかというと [Playwright の Projects](https://playwright.dev/docs/test-projects) に似たもので、同じ package 内であっても論理的にテスト実行の設定を切り分けるような使い方ができます。

:::message
主な使い方である monorepo での使用について、公式ドキュメント以外にはあまり記事が見つからなかったのでそちらに触れてもよかったんですが、この記事では割愛します。
:::

# 設定

デモリポジトリです。
https://github.com/you-5805/vitest-workspace

`vitest.config.ts` の代わりに `vitest.workspace.ts` を作成し、例えば以下のように設定すると、`*.node.test.tsx` では `node` が実行環境として使われるようになります。

```ts:vitest.workspace.ts
import { defineConfig, defineWorkspace, mergeConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

const sharedConfig = defineConfig({
  test: {
    setupFiles: ['src/__tests__/setup.ts'],
  },
});

const browserConfig = defineConfig({
  plugins: [react()],
  test: {
    name: "browser",
    environment: "jsdom",
    exclude: ["src/**/*.node.test.{ts,tsx}", "node_modules"],
  },
});

const nodeConfig = defineConfig({
  test: {
    name: ' node',
    environment: 'node',
    include: ['src/**/*.node.test.{ts,tsx}'],
  },
});

export default defineWorkspace([
  mergeConfig(sharedConfig, browserConfig),
  mergeConfig(sharedConfig, nodeConfig),
]);
```

公式ドキュメントにある設定方法とちょっと違いますが、自分の中ではこのあたりが分かりやすそうな書き方だと思いました。

:::message
もしビルドでも Vite を使用していて `vite.config.ts` がある場合は、それを impoirt して `mergeConfig` してもいいですし、`defineConfig` の `extends` で `./vite.config.ts` を指定してもいいです。
:::

# 使うことありますか？

正直、使わなくても良いケースが大半だとは思います。
というのも、これを見るだけだと、
「`jsdom` 環境だと Node.js の API やサーバーサイドの環境変数にアクセスできなかったりするから、そういうものに触るモジュールのテストでは `node` を使用するようにする必要があるんだな」
と思うかもしれませんが、実際は `jsdom` はブラウザ API にアクセスできるようにする拡張のようなもので、Vitest が Node.js で実行されているのであれば実装は普通に Node.js API にはアクセスできてしまいます。

# どんな時に使うか

じゃあどういう時に使うかというと、これは実際に自分が困った例なんですが、`jsdom` 環境で実行すると単純に以下のような関数がエラーになります。

```ts
function getSecret() {
  if (typeof window !== 'undefined') {
    throw new Error('This is not a browser!');
  }
  return 'SECRET';
}
```

正直これについては、例えば `isServer` の実装を外部化して、vi.mock() でサーバーであるかのように見せればいい話ですが、これがライブラリ側に書かれていたりすると、モックする以外では対応できなさそうです。
この関数について、実行環境を `node` にしてあげれば期待通り `typeof window` は `undefined` になるので、`'SECRET'` を返すことができます。

これの対応のために Vitest の環境を `jsdom` / `node` で切り替える仕組みを入れることが妥当であるかどうかはさておき、確かに「ブラウザでないこと」の分岐が発生する実装はたまによくありそうで、そういう時には Workspace が役に立つかもしれません。

# もっとカジュアルに変更できないのか？（できない）

ファイル名に変更が入るのはやや面倒ですよね。

:::message
2023.12.29 12:24 に上記の設定を修正して、`jsdom` を使う方の workspace について `include` ではなく `exclude` を使用することで、`*.browser.test.tsx` というファイル名を使用しなくても、`*.node.test.tsx` というファイルだけを例外的に `node` 環境で扱う、という設定にできたので、特に面倒ではなくなりました (?)。
:::

例えばテストファイルスコープの設定と言えば `vi.setConfig()` があり、こういうので設定できると楽だったんですが、そういう設定はできません。まあ実行環境なので、実行時に読み取られる個別の設定よりはもっと前段で分岐が必要そうです。

それと、例えば Storybook の tag や Playwright の Group tests annotation のように、ファイル名ではなくコード上の記述で何らかこれを分岐する余地があればもっと良かったんですが、今のところそういう設定も難しそうです。
