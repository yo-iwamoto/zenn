---
title: '最小限の gssp → App Router 移行'
emoji: '🦥'
type: 'tech'
topics: ['nextjs', 'approuter']
published: true
---

# はじめに

- この記事で触れる「移行」はサーバー側のデータ取得処理、具体的には getServerSideProps、getStaticProps のみで、それ以外は省略しています。
  `next/router` to `next/navigation` や File Conventions 等その他の移行については、公式の Migration Guide で十分そうという認識です。
  https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration

- コード例は簡潔さのため詳細が省略されていることがあります。

- 誤った記述があれば [@yoiwamoto](https://twitter.com/yoiwamoto) に教えてください！

- 個人的には本編は最後の App Router との格闘で、前半はただの現状整理です。

# 最小限の変更で移行したい

Next.js の App Router が Stable になっていますが、コンポーネントレベルのデータ依存・キャッシュ管理など、pages とは根本からモデルが違う部分も多いです。

今後、初めから App Router で実装を始めるアプリケーションであれば、App Router の特徴を活かして Server Components や Server Actions を活用するような設計を行うことも考えられますが、pages をベースに実装されている既存の大きなアプリケーションについて、
**「App Router 移行ついでに、データ依存周りの処理からガラッと設計を変えましょうか ！💪」**
とは、なかなかならないですよね。

仮に設計を更新することには肯定的だったとしても、膨大な影響範囲を鑑みると、App Router への移行とは分けて取り組みたいと考えられます。

この記事では、App Router に移行した際のサーバー側でのデータ取得処理の変更について、**トップレベルから props を流し込むというアプリケーション構造を崩さない範囲での**、現実解的な移行例を挙げます。おそらく一般解でもありそうという認識です。

それと、SWR や TanStack Query で fallback を流し込む構成も一般的だと思うのでそれについても触れます。

# 1. 普通の getServerSideProps

```ts:pages/posts/[id]/index.tsx
export const getServerSideProps = (async (ctx) => {
  const post = await fetch(`https://api.example.com/posts/${ctx.query.id}`).then(r => r.json());
  return { props: { post } };
}) satisfies GetServerSideProps;

export default function Page({ post }: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return <PostPage post={post} />;
}
```

データ取得をしているだけなので以下のように書き換えて、`PostPage` のファイル先頭に `'use client'` を追記で動きます。
ただし、App Router では `fetch` に何も設定しないと SG と同じ挙動になるので、上記の挙動を再現するなら `cache: 'no-cache'` を指定する必要があります。

```ts:app/posts/[id]/page.tsx
export default async function Page({ params: { id }}: { params: { id: string } }) {
  const post = await fetch(`https://api.example.com/posts/${id}`, {
    cache: 'no-cache', // 指定しないと SG と同じ挙動になります。
  }).then(r => r.json());
  return <PostPage post={post} />;
}
```

それと、通常取得したデータは `<Head>` や `<NextSeo>` を介して SEO 周りの設定にも使用されていると思います。
App Router では動的なメタ情報取得は `generateMetadata` で行うので、これまで gssp から受け取った props をそのまま渡していた場合でも、別で記載する必要があります。

```ts:app/posts/[id]/page.tsx
export async function generateMetadata({ params: { id }}: { params: { id: string } }) {
   const post = await fetch(`https://api.example.com/posts/${id}`, {
    cache: 'no-cache',
  }).then(r => r.json());

  return {
    title: post.title,
  } satisfies Metadata;
}
```

この時、同じ URL に対して 2 回 `fetch` が発生しそうに見えますが、App Router では単一のページリクエストに対して同じ URL への `fetch` が複数回発生しないよう deduping が行われるので実際に行われる `fetch` は 1 回です。
https://nextjs.org/docs/app/building-your-application/data-fetching#automatic-fetch-request-deduping

ドキュメントにもある通り、データ取得がもし `fetch` ではなく DB へのクエリ等であれば React の `cache()` を使用してください。単一のリクエストに対して同じ引数での呼び出しが dedupe されます。
https://nextjs.org/docs/app/building-your-application/data-fetching/caching#react-cache

# 2. キャッシュが有効な getServerSideProps

gssp では `res.setHeader` で `Cache-Control` を触れるので、以下のようにキャッシュさせることも多いと思います。

```ts:pages/posts/[id]/index.tsx
export const getServerSideProps = (async (ctx) => {
  // Cache-Control を指定
  ctx.res.setHeader('Cache-Control', 'public, s-max-age=60, stale-while-revalidate');
  const post = await fetch(`https://api.example.com/posts/${ctx.query.id}`).then(r => r.json());
  return { props: { post } };
}) satisfies GetServerSideProps;

export default function Page({ post }: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return <PostPage post={post} />;
}
```

App Router では、厳密に言うと Next.js にキャッシュを握らせずに Cache-Control でクライアントまたは CDN 側でキャッシュをさせるこの振る舞いを実現する方法はありません。
このユースケースに対して、revalidate ありの getStaticProps で実現していたいわゆる ISR のようなものが `fetch` レベルで可能になっています。[4. revalidate を指定した getStaticProps](#4.-revalidate-を指定した-getstaticprops) を参照してください。

一応、特定のパスに対して常に一定のレスポンスヘッダを指定したいだけであれば、以下で動き**そうですが**、実際には middleware で指定した Cache-Control は Next.js により上書きされてしまいます。要件によって Surrogate-Control 等が使える可能性があるので検討するのが良さそうです。
https://github.com/vercel/next.js/issues/48480

```ts:middleware.ts
export const config = {
  matcher: '/:path*',
};

const handler = ((req) => {
  const { headers } = req;
  if (/^\/posts\/\d+$/.test(req.nextUrl.pathname)) {
    headers.set(
      'Cache-Control',
      'public, s-max-age=60, stale-while-revalidate',
    );
  }

  return NextResponse.next({ request: { headers } });
}) satisfies NextMiddleware;

export default handler;
```

ちなみに next.config.js の `headers` では、ページのキャッシュは制御できません。
https://nextjs.org/docs/app/api-reference/next-config-js/headers#cache-control

> You cannot set Cache-Control headers in next.config.js file as these will be overwritten in production to ensure that API Routes and static assets are cached effectively.

# 3. パスパラメータ無しの getStaticProps

```ts:pages/posts/index.tsx
export const getStaticProps = (async () => {
  const posts = await fetch('https://api.example.com/posts').then((r) => r.json());
  return { props: { posts } };
}) satisfies GetStaticProps;

export default function Page({ post }: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return <PostsPage posts={posts} />;
}
```

App Router では何も設定しない場合デフォルトで静的にデータ取得を行うので、ただ fetch するだけでよいです。

```ts:app/posts/page.tsx
export default async function Page() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());
  return <PostsPage posts={posts} />;
}
```

例えばデータ依存がこれのみの場合、レスポンスヘッダの `Cache-Control` は `s-maxage=31536000, stale-while-revalidate` となり、別のデータ依存があった場合の内部的なキャッシュ挙動もこれになります。

# 4. revalidate を指定した getStaticProps

```ts:pages/posts/index.tsx
export const getStaticProps = (async () => {
  const posts = await fetch('https://api.example.com/posts').then((r) => r.json());
  return {
    props: { posts },
    revalidate: 30,
  };
}) satisfies GetStaticProps;

export default function Page({ post }: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return <PostsPage posts={posts} />;
}
```

似たような挙動は以下の実装で実現できます。

```ts:app/posts/page.tsx
export default async function Page() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 30 }, // これ
  }).then(r => r.json());
  return <PostsPage posts={posts} />;
}
```

ただしこれは、ページリクエストに対するキャッシュ設定ではなく、`fetch` レベルの設定です。
リクエストに対して使用されている SC でのデータ依存がここだけであれば、Next.js はレスポンスヘッダで `Cache-Control: public, s-max-age=30, stale-while-revalidate`　を付けますが、他に `cache: 'no-cache'` のような `fetch` が存在すれば、レスポンスヘッダとしては `Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate` とかになり、クライアント側でキャッシュすることはできません。
Next.js 側で内部的に HIT はしますが、構成に影響があると思うので十分検証や、場合によっては部分的にリアーキが必要です。

# 6. SWR の fallback

サーバーステートの管理に SWR を使っている場合、key に対して以下のように fallback を当てることがあると思います。

```ts:pages/posts/[id]/index.tsx
export const getServerSideProps = (async (ctx) => {
  const { id } = ctx.query;
  if (typeof id !== 'string') return { notFound: true };

  const post = await fetch(`https://api.example.com/posts/${id}`).then((r) =>
    r.json(),
  );

  return {
    props: {
      fallback: {
        [unstable_serialize(['posts', id])]: post,
      },
    },
  };
}) satisfies GetServerSideProps;
```

```ts:pages/_app.tsx
import { SWRConfig } from 'swr';
import type { AppProps } from 'next/app';

export default function App({
  Component,
  pageProps: { fallback, ...pageProps },
}: AppProps) {
  return (
    <SWRConfig value={{ fallback }}>
      <Component {...pageProps} />
    </SWRConfig>
  );
}
```

同様のことは App Router でも実現可能ですが、pages の時のように `fallback` というオブジェクトに key-value で入れておけば fallback される、のような使い方はできず、各ページで詰める必要はあります。

```ts:app/posts/[id]/page.tsx
export default async function Page({ params: { id }}: { params: { id: string } }) {
  const post = await fetch(`https://api.example.com/posts/${id}`, { cache: 'no-cache' }).then(r => r.json());
  return (
    <FallbackProvider fallback={{ post }}>
      <PostPage />
    </FallbackProvider>
  );
}
```

`SWRConfig` は ReactContext などのブラウザで動く機能で実現されているので、例えば以下のように CC として別定義します。

```ts:components/FallbackProvider.tsx
'use client';

type Props = PropsWithChildren<{
  [key: string]: any;
}>;

export function FallbackProvider({ fallback, children }: Props) {
  return <SWRConfig value={{ fallback }}>{children}</SWRConfig>;
}
```

# 7. TanStack Query の fallback

やっていることはほとんど同じなので、コード例のみで説明は割愛します。

```ts:pages/posts/[id]/index.tsx
export const getServerSideProps = (async (ctx) => {
  const { id } = ctx.query;
  if (typeof id !== 'string') return { notFound: true };

  const queryClient = new QueryClient();
  await queryClient.prefetchQuery(
    ['posts', id],
    () => fetch(`https://api.example.com/posts/${id}`).then((r) => r.json()),
  );

  return {
    props: {
      dehydratedState: dehydrate(queryClient),
    },
  };
}) satisfies GetServerSideProps;
```

```ts:pages/_app.tsx
export default function App({
  Component,
  pageProps: { dehydratedState, ...pageProps },
}: AppProps) {
  const [queryClient] = useState(() => new QueryClient())

  return (
    <QueryClientProvider client={queryClient}>
      <Hydrate state={dehydratedState}>
        <Component {...pageProps} />
      </Hydrate>
    </QueryClientProvider>
  )
}
```

これを以下のように変えます。

queryClient が `cache()` でシングルトン的に使われていたり、別途 layout.tsx で `QueryClientProvider` を置く必要はあったりするので、ドキュメントを参照ください。（ところで現時点で App Router に公式が対応してドキュメントまであるの優秀すぎませんか？しかも全く問題なく動く）
https://tanstack.com/query/latest/docs/react/guides/ssr#using-the-app-directory-in-nextjs-13

```ts:app/posts/[id]/page.tsx
export default async function Page({ params: { id }}: { params: { id: string } }) {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery(
    ['posts', id],
    () => fetch(`https://api.example.com/posts/${id}`).then((r) => r.json()),
  );

  return (
    <Hydrate state={dehydrate(queryClient)}>
      <PostPage />
    </Hydrate>
  );
}
```

# そもそも移行が不可能な例

一部の要件について、App Router で、少なくとも Next.js で完結する形では実現が不可能と思われる処理がある場合、現時点で pages から移行する判断は取れないと考えられるので、先に確認しておきましょう。

## **getServerSideProps (gssp) で、API から取得したデータに応じて `Cache-Control` の値を切り替えている**

これは例えば以下のような実装のことを指します。

```ts:pages/index.tsx
export const getServerSideProps = (({ res, query }) => {
  const post = await fetch(`https://example.com/api/posts/${query.id}`).then(r => r.json());

  if (post.status === PostStatus.DRAFT) {
    // 下書きステータスの時、キャッシュさせない
    res.setHeader('Cache-Control', 'no-store');
  } else {
    // 通常、キャッシュさせる
    res.setHeader('Cache-Control', 's-maxage=60, stale-while-revalidate');
  }
  return { props: { post } };
}) satisfies GetServerSideProps;
```

この gssp では、API から取得した post の status の値に応じて `Cache-Control` を切り替えています。
比較的一般的なパターンと思われますが、App Router では実は現状これに相当する振る舞いを実現する方法がありません。
（下書きと公開を同じ URL で表示するなと言われそうですが、まあこの例に限らずキャッシュを動的に制御することは普通にあると思います）

実現できない理由として、App Router でのデータのキャッシュ制御は **fetch の引数または Segment Config** によって行うことができるのですが、fetch の引数を渡す時点ではもちろん、まだ post が取得されていないため参照することができませんし、Segment Config に至ってはビルド時点で分かっている必要があるため、動的なデータを扱う場合は使えません。

```ts:app/page.tsx
export default async function Page({ params }) {
  const post = await fetch(`https://example.com/api/posts/${query.id}`, {
    // 直感的なイメージとしては、ここと、
    cache: isDraft ? 'no-cache' : undefined,
    next: {
      // ここで isDraft を知りたい
      revalidate: isDraft ? 0 : 60,
    }
  }).then(r => r.json());

  return <PostPage post={post} />;
}
```

ここで、例えば status を取得する API だけ別立てするとか、API のレスポンスの Cache-Control で制御する、のように Next.js 外にまで影響を広げてしまえば解決できる場合もありますが、これが単にフレームワークの表現力の都合であることを考えると、正直悪手だと思われます。
ので、現状このような動的なキャッシュ制御はできないと思っています。
**もし上記が間違っていて普通にできる場合、今まさにすごく困っているのでぜひ教えて欲しいです 🙏🙏**

# (未解決) キャッシュ設定を動的に変更するページも移行したい

:::message
以下、脱線です。
タイトルと直接は関係ありませんが、解決した場合、# そもそも移行が不可能な例 セクションを消せるので書いています。
:::

既に試したことを書きます。

## SC は `'force-dynamic'` にして、前段の middleware でデータを取得、`Surrogate-Control` をセットする

そもそも self-host の場合 Next.js にキャッシュを握らせない構成が明らかに分かりやすいと思うので、常に `'force-dynamic'` にした上で、SC と違い、HTTP リクエストに対して直接ヘッダーを触れる唯一の場所である middleware でキャッシュ制御を行おうとしたアプローチです。

```ts:app/page.tsx
export const dynamic = 'force-dynamic';

...
```

```ts:middleware.ts
const handler = ((req) => {
  const id = getIdParameterFromUrl(req.url);
  const post = await fetch(`https://api.example.com/posts/${id}`).then(r => r.json());

  const surrogateControl =
    post.status === PostStatus.DRAFT
      ? 'private, no-store'
      : 'public, s-maxage=60, stale-while-revalidate';
  const { headers } = req;
  headers.set('Surrogate-Control', surrogateControl);

  return NextResponse.next({ request: { headers } });
}) satisfies NextMiddleware;

export default handler;
```

このようにすると、middleware の時点でレスポンスヘッダを指定できるので、取得したデータの値に依存した設定で CDN でキャッシュさせ、SC が実行されるときは常に新鮮なデータを取りに行くという振る舞いを実現できます。

が、以下の問題点があります。

### データ取得の重複排除が有効でない

取得しているデータは SC でも取得したい値で、かつ [Automatic `fetch()` Request deduping ](https://nextjs.org/docs/app/building-your-application/data-fetching#automatic-fetch-request-deduping)は SC、`generateMetadata`、`generateStaticParams` のみで有効で、middleware では重複排除が行われません。
オリジンにリクエストが到達した場合は**常に API が直列で 2 回呼ばれることになり**、これは基本的にはパフォーマンス上許容できないと思います。

もちろん、別途 Redis 等の KV を置いて、取得したデータを自前で引き回すことは可能ですが、このためにミドルウェアを増やしたくはないですよね。
ちなみにもしこの方法を試す場合、リクエストごとに一意な ID は Next.js では振られていないようなので、自前で採番してカスタムヘッダや Cookie に入れ、SC から参照できる必要があります（さらに脱線）。
`next/headers` でアクセス可能な Cookie は middleware での変更の適用前であることに注意する必要があります。
現状、以下の Issue にあるような方法で fresh な Cookie にアクセスすることができるので、もし必要があればご参照ください。
https://github.com/vercel/next.js/discussions/49444

### `Cache-Control` や `Vary` などキャッシュ設定上重要なレスポンスヘッダは Next.js に管理されており触れない。

vercel/next.js に Issue も立っているんですが、middleware では、FW 側が制御したいこれらのヘッダを触れないことがあります。自分も試しましたが、どちらかというと**指定したのに next に上書きされる**というのが正しそうです。
https://github.com/vercel/next.js/issues/48480

例えばこのアプローチでは、Fastly に実装されている [Surrogate-Control](https://developer.fastly.com/reference/http/http-headers/Surrogate-Control/) という、`Cache-Control` とは別に CDN キャッシュを制御するためのヘッダを使っているので回避できていますが、それとは別にブラウザキャッシュもさせたい場合は使えません。

custom server などで回避できる可能性はありそうですが、そこまでレールから外れるのは基本的には避けたいですよね 😢

## Help me 🏳️

その他、Next.js にキャッシュを全く握らせないことは諦めた上で、データ取得後に分岐して `revalidateTag` してみたり（SC でも動いてるように見える時がある; 未検証）、`cache: force-cache` で取得後に `revalidateTag` して、再度同じ URL に `cache` と `next` の値を指定して `fetch` してみたりと、不毛な格闘をしています。助けてください。
