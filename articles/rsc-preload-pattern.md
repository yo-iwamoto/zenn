---
title: 'RSC での preload パターンの使い所'
emoji: '🐠'
type: 'tech'
topics: ['react', 'nextjs', 'approuter']
published: true
---

## preload パターンについて

preload パターンは、Next.js 公式ドキュメントの Data Fetching Patterns and Best Practices 内で紹介されている^[2024.05.04 現在]、Sequential data fetching によるウォーターフォールを回避するための 1 手法です。

https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#preloading-data

## 使い所

Sequential data fetching が発生するようなケースとその解決方法はいくつかあるのですが、preload パターンの典型的な使い所と言えるのは以下のようなケースです。

例は、認証されているユーザーに対してのみ、課金プランを表示するようなページです。

```tsx:app/billing-plan/page.tsx
import { redirect } from 'next/navigation';
import { getIsAuthenticated } from '@/lib/auth';

export default function Page() {
  const isAuthenticated = await getIsAuthenticated();
  if (!isAuthenticated) {
    redirect('/login');
  }

  const billingPlan = await fetch('https://api.example.com/api/billing-plan').then((r) =>
    r.json(),
  );

  return <h1>{billingPlan.displayName}</h1>
}
```

上記の route では、`https://api.example.com/api/billing-plan` から取得した課金プラン情報を表示しますが、この billingPlan のリクエストは、`getIsAuthenticated()` やその下の処理が実行された後に行われます。
`getIsAuthenticated()` が一瞬で完了することを期待するような関数であればこれでよいかもしれませんが、これが例えばネットワークを介するものであったり、その他非同期の重い計算を伴うような処理であったりすれば、2 つの処理が直列で実行されることはこの route のレスポンス速度に悪影響を及ぼします。

具体的には、試しにこのサンプルコードで `getIsAuthenticated()` と billingPlan を取得する処理にそれぞれ 1000ms のスリープを入れると、レスポンスの完了に 2s 以上がかかることが確かめられます。

![Chrome devtool 内の Network での Timing データ。Waiting for server response が 2.04s で、total が 2.05s。](/images/rsc-preload-pattern/bad-pattern.png =300x)
_devtool > Network > Timing_

## どのように解決するか

パターンといいつつ大したことではないのですが、以下のように、リクエストを先に非同期で開始しておいて、後から改めて `await getBillingPlan()` してレスポンスを受け取ります。

これが可能である理由は、現在の Next.js / React では、同一 URL への fetch は暗黙的に メモ化（route リクエストに対してそれぞれ 1 回しか発生しないようにキャッシュ）されるためです。

:::message
fetch のメモ化は現在暗黙的に行われますが、ここでは同様のことを行う react の `cache()` を敢えて明示的に使用しています。
理由としては、React は次バージョンの v19 で、これを実現するために行っていた fetch のパッチを切る予定^[[Remove automatic fetch cache instrumentation - facebook/react](https://github.com/facebook/react/pull/28896)]で、Next.js がこの役割を引き継ぐかはまだ分かっていないためです。
:::

```tsx:app/billing-plan/page.tsx
import { redirect } from 'next/navigation';
import { cache } from 'react';

const getBillingPlan = cache(async () => {
  return fetch('https://api.example.com/api/billing-plan').then((r) => r.json());
});

export default function Page() {
  void getBillingPlan(); // preload

  const isAuthenticated = await getIsAuthenticated();
  if (!isAuthenticated) {
    redirect('/login');
  }

  const billingPlan = await getBillingPlan();

  return <h1>{billingPlan.displayName}</h1>
}
```

これにより、先ほど 1000ms ずつのスリープを入れた場合に 2s かかっていたレスポンスは、1s で終わるようになることが確かめられます。

![Chrome devtool 内の Network での Timing データ。Waiting for server response が 1.04s で、total が 1.05s。](/images/rsc-preload-pattern/pattern-1.png =300x)
_devtool > Network > Timing_

## `use` API を使うパターン

experimental なためか、Next.js の公式ドキュメントには記載がありませんが、React の `use` API でも同じような改善ができます。

https://react.dev/reference/react/use

```tsx:app/billing-plan/page.tsx
import { redirect } from 'next/navigation';
import { use } from 'react';
import type { BillingPlan } from '@/types';

export default function Page() {
  // await せず、Promise を保持する
  const billingPlanPromise = fetch('https://api.example.com/api/billing-plan').then((r) => r.json());

  const isAuthenticated = await getIsAuthenticated();
  if (!isAuthenticated) {
    redirect('/login');
  }

  return <BillingPlan billingPlanPromise={billingPlanPromise} />;
}

function BillingPlan({ billingPlanPromise }: { billingPlanPromise: Promise<BillingPlan> }) {
  const billingPlan = use(billingPlanPromise);

  return <h1>{billingPlan.displayName}</h1>;
}
```

![Chrome devtool 内の Network での Timing データ。Waiting for server response が 1.03s で、total が 1.04s。](/images/rsc-preload-pattern/pattern-2.png =300x)
_devtool > Network > Timing_

## 補足

言わずもがなですが、上記のコードで生じるどの suspend も Page に伝播されるので、示した Timing のようにレスポンス時間に影響を与えます。
そのため、実際には上記のような書き方ではなく、都度 `<Suspense>` を引いたり、Next.js App Router であれば `loading.tsx` を利用するなどして、ストリーミングレスポンスを行うのが望ましいと思います。

例えば今回の例で言うと、`loading.tsx` をとりあえず置いておくだけで以下のようなストリーミングレスポンスになり、この Content Download の時間にローディング UI を表示することができます。
![Chrome devtool 内の Network での Timing データ。Waiting for server response が 43.41ms で、Content Download が 994.09ms。](/images/rsc-preload-pattern/streaming.png =300x)
_devtool > Network > Timing_

## preload パターンのデメリット

- 今回のように、先に行う処理の結果によってはデータ取得が不要になることもあるようなケースでは、単純に、不必要なリクエストが発生することになります。

## References

サンプルコード置き場
https://github.com/yo-iwamoto/sample-code-rsc-preload-pattern/tree/main/src/app
