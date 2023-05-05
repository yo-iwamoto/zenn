---
title: 'Vercel はもっと薄く使える'
emoji: '🍦'
type: 'tech'
topics: ['vercel', 'typescript', 'serverless']
published: true
---

# はじめに

Vercel Ship、アツい発表が続いていますね。
特に初日の Storage は、KV、Postgres、Blob と 3 つのプロダクトが公開され、「とりあえず何か試したい！」「今の構成のあのコンポーネントは Vercel で済ませられるようになりそう！」となった方も多いんじゃないでしょうか。
https://vercel.com/blog/vercel-storage

Vercel で何かしら関数を動かすとなった時に一番最初に思い付かれがちな選択肢は、とりあえず `create-next-app` して `pages/api/hello.ts` を作成して... というあれですが、Vercel を使ってみたい人（またはプロジェクト）のうち全員が React を使っているわけではありませんし、そもそもフロントエンド無しで完結するユースケースも多いはずです。

Vercel の Serverless Functions は（ご存知の方も多いと思いますが）実際には Framework agnostic なもので、もっと薄く、Node.js だけでも[^1]使うことができます。
便利な反面、あまり一般的では無さそうなので、使い方を紹介します。

# Demo Repository

こちらの README にもある程度書いています。

https://github.com/you-5805/vanilla-vercel-functions

# Serverless Functions

https://vercel.com/docs/concepts/functions/serverless-functions

root に `api` ディレクトリ内に、以下のようにハンドラを default export するファイルを置くと、Serverless Functions として動作します。

デモ: https://vanilla-vercel-functions.vercel.app/api/hello

```ts:api/hello.ts
export default function handler(req: VercelRequest, res: VercelResponse) {
  res.status(200).json({
    message: 'Hello world',
    cookies: req.cookies,
  })
}
```

# Edge Functions

https://vercel.com/docs/concepts/functions/edge-functions

以下のような `config` オブジェクトを `export` すると、Edge Functions として動作させることができます。

Edge Functions のランタイムは軽量で、Node.js ではないので様々な制約を受けますが、その分パフォーマンスや物理的距離によるレイテンシの低さで様々な使い所があります。
また、以下のように [@vercel/edge](https://www.npmjs.com/package/@vercel/edge) という npm パッケージとして、リクエストの地理情報・IP アドレスの取得等は簡単に行うことができます。

デモ: https://vanilla-vercel-functions.vercel.app/api/edge

```ts:api/edge.ts
import { geolocation } from '@vercel/edge';

export const config = {
  runtime: 'edge',
};

export default function handler(req: Request) {
  const { city } = geolocation(req);

  return new Response(`Hello, from ${city} I'm now an Edge Function!`);
}
```

# ちなみに

## Vercel KV

https://vercel.com/docs/storage/vercel-kv

上記のような関数から、最近発表された Vercel KV などの Storage にもアクセスできます。
（確認してませんが Postgres や Edge Config、Blob も同様かと思います。）
例えば Node.js の場合、[@vercel/kv](https://www.npmjs.com/package/@vercel/kv) というクライアントライブラリが公開されていますが、こちらがありがたいことにエッジランタイムで動作可能になっているので、Edge Functions でも使えます。

以下は来訪者カウンター実装の例です。

デモ: https://vanilla-vercel-functions.vercel.app/api/kv

```ts:api/kv.ts
import type { RequestContext } from '@vercel/edge';
import kv from '@vercel/kv';

export const config = {
  runtime: 'edge',
};

export default async function handler(_: Request, ctx: RequestContext) {
  const currentCount = (await kv.get<number>('visitor-count')) ?? 0;
  const incremented = currentCount + 1;
  ctx.waitUntil(kv.set('visitor-count', incremented));

  return new Response(`You're No. ${incremented} visitor!`);
}
```

## express もデプロイできる

https://vercel.com/guides/using-express-with-vercel

express のインターフェースは Serverless Functions に互換なので、イベントハンドラとしてそのまま利用できます。

デモ: https://vanilla-vercel-functions.vercel.app/api/express/posts/with-dynamic-param

```ts:api/express/index.ts
import express, { Router } from 'express';

const app = express();
const router = Router();

router.get('/posts/:slug', (req, res) => {
  res.json({
    post: {
      title: 'Test Post',
      slug: req.params['slug'],
    },
  });
});

app.use('/api/express', router);

export default app;
```

:::message
素の Serverless Functions では、Next.js のように複数のダイナミックパスパラメータを受けとる関数は作成できないようなので、代わりに特定のパス以下のリクエストを全て express に流すために `vercel.json` で `rewrites` の設定が必要です。
（`/api/express/index.ts` にファイルを置いた場合、以下）

```json:vercel.json
{
  "rewrites": [
    {
      "source": "/api/express/(.*)",
      "destination": "/api/express"
    }
  ],
}
```

参考: https://vercel.com/guides/using-express-with-vercel#standalone-express
:::

~~（本当は [Hono](https://hono.dev) を使いたかったんですが、うまく動かせませんでした。分かる方いたら教えてください。）~~

## 🔥 Hono も使えた（追記 - 2023.05.05）

hono/nextjs で Serverless Functions でも動かせるようでした。（バージョンか書き方のミスだったかもしれません）
hono だと組み込みの BASIC 認証ミドルウェアで、簡単に認証がかけられたりもしていて圧倒的に便利ですね。

デモ: https://vanilla-vercel-functions.vercel.app/api/hono/auth
ID: `vercel` / PW: `hono`

ご対応ありがとうございます！

https://github.com/you-5805/vanilla-vercel-functions/pull/2

## Static Files

Vercel ならデフォルトで Static File Serving が有効なので、`public` ディレクトリに置くだけで雑に配信できます。

Web サイトが必要なわけではなくても、画像などちょっとしたファイルを配信したいことは多いと思うので、ついでに活用しましょう。

デモ: https://vanilla-vercel-functions.vercel.app/author.jpg

# まとめ

これでもう、API をデプロイしたいだけなのに `npm i next` や `npm i react` をして不思議な気持ちにならずにすみますね。

[^1]:
    Node.js の他に公式で Go、Python、Ruby、非公式で Bash、Deno、PHP、Rust のランタイムがサポートされています。
    https://vercel.com/docs/concepts/functions/serverless-functions/runtimes
