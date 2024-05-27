---
title: 'Vitest の vi.mock は巻き上げられる'
emoji: '🐣'
type: 'tech'
topics: ['vitest', 'test']
published: true
---

## はじめに

Jest でも同じかもしれませんが、検証していないためあくまで Vitest についてということで書いています。

## `vi.mock` の呼び出しは巻き上げられる

最もオーソドックスなモック手法である `vi.mock` ですが、**ファイル内での記載位置は関係なく**、実行時に Vitest によって hoist され、テストファイルの先頭で実行されます。
（厳密には、`vi.mock` が先頭で実行された後、callback である factory 関数が実行されるのは対象の参照時のようです）

https://vitest.dev/api/vi.html#vi-mock

例として以下のようなテストについて、

```tsx:user-profile.test.tsx
import { render } from '@testing-library/react';
import { UserProfile } from './user-profile';

vi.mock('./use-user-query', () => ({
  useUserQuery: vi.fn(() => ({
    data: { name: 'John Doe' },
  })),
}));

describe('UserProfile', () => {
  it('ユーザー名が表示されること', () => {
    const { getByText } = render(<UserProfile />);
    expect(getByText('John Doe')).toBeInTheDocument();
  });
});
```

なんとなく `vi.mock` を追記すると、そのタイミングでモックが上書きできるような気がするんですが、続きに以下のような test suite を追加しても、fail します。

```tsx:user-profile.test.tsx
it('データが取得中の時、ローディング UI が表示されること', () => {
  vi.mock('./use-user-query', () => ({
    useUserQuery: vi.fn(() => ({
      data: undefined,
    })),
  }));

  const { getByText } = render(<UserProfile />);
  expect(getByText('Loading...')).toBeInTheDocument();
})
```

具体的には、同じパス（`'./use-user-query'`）に対する `vi.mock` の呼び出しがファイル内に 2 つ存在するので、**後から呼び出された方が優先され**、data が undefined となり、最初の suite が落ちます。

ただ、UserProfile の実装にもよりますが、ここでは useUserQuery は単にスタブすればよい依存ではなく、UserProfile の挙動を観察するための主要な入力となっていそうです。
モックしたモジュールの振る舞いを任意のタイミングで更新する方法について、実現する手法は色々ありますが、結論から書くと以下のようにするのが最も簡潔そうだと思っています。

## 簡潔なモックの書き方

```tsx:user-profile.test.tsx
import { render } from '@testing-library/react';
import { UserProfile } from './user-profile';

const useUserQuery = vi.hoisted(() =>
  vi.fn(() => ({
    data: { name: 'John Doe' },
  })),
)
vi.mock('./use-user-query', () => ({
  useUserQuery,
}));

describe('UserProfile', () => {
  it('ユーザー名が表示されること', () => {
    const { getByText } = render(<UserProfile />);
    expect(getByText('John Doe')).toBeInTheDocument();
  });

  it('データが取得中の時、ローディング UI が表示されること', () => {
    // モック関数の返り値を変更する
    useUserQuery.mockReturnValue({
      data: undefined,
    });

    const { getByText } = render(<UserProfile />);
    expect(getByText('Loading...')).toBeInTheDocument();
  });
});
```

:::details 別の書き方

```tsx:user-profile.test.tsx
import { useUserQuery } from './use-user-query';
import { render } from '@testing-library/react';
import { UserProfile } from './user-profile';

vi.mock('./use-user-query', () => ({
  useUserQuery: vi.fn(() => ({
    data: { name: 'John Doe' },
  })),
}));
// vi.mock は巻き上げられているので、import された時点で useUserQuery は vi.fn() に置き換えられている
const useUserQueryMock = useUserQuery as Mock;
```

:::

## ポイント

`vi.mock` の実行はテストファイル先頭に巻き上げられるため、トップレベルの変数は一見使えそうに見えて、使えません。`vi.hoisted` の実行は `vi.mock` と同様に巻き上げられるため、ここでモックに使用する値を宣言することで、そのまま使えるようになります。
例えば以下のように書いても失敗します。

```tsx:user-profile.test.tsx
const useUserQuery = vi.fn(() => ({
  data: { name: 'John Doe' }
}));
vi.mock('./use-user-profile', () => ({
  useUserQuery,
}));
```

```plane:stdout
Error: [vitest] There was an error when mocking a module. If you are using "vi.mock" factory, make sure there are no top level variables inside, since this call is hoisted to top of the file. Read more: https://vitest.dev/api/vi.html#vi-mock
```

この問題を解決する別の方法として、`vi.doMock` という API も存在しますが、これは巻き上げられないために、モックがその呼び出し以降の dynamic import にしか反映されないものなので、少なくとも使い所は変わってくると思っています。

## 注意

`vi.mock` や `vi.hoisted` はこういう巻き上げの対象なので、（挙動の把握が正確でない可能性がありますが）テストファイルのトップレベルにベタ書きされている必要があり、例えば以下のように共通化するということはできません。

```ts:__tests__/utils.ts
export function mockUseUserQuery() {
  const { useUserQuery } = vi.hoisted(() => ({
    useUserQuery: ,
  }));
  vi.mock('./use-user-query', () => ({
    useUserQuery,
  }));

  return { useUserQuery };
}
```

```tsx:user-profile.test.tsx
import { UserProfile } from './user-profile';
import { mockUseUserQuery } from '@/__tests__/utils';

const { useUserQuery } = mockUseUserQuery();

describe('UserProfile', () => {
  it('ユーザー名が表示されること', () => {
    const { getByText } = render(<UserProfile />);
    expect(getByText('John Doe')).toBeInTheDocument();
  });
});
```

## 場合によっては **mocks** での静的なモックで良い

ここではテスト対象の入力として振る舞いを操作したかったので、テストファイルから `useUserQuery` にアクセスできる必要がありこのような書き方をしていますが、単にスタブすれば十分である場合、`__mocks__` からスタブ用のモジュールを export することで、factory 関数無しでモックできるので、それで済ませて良い場合もあります。

https://vitest.dev/api/vi.html#vi-mock

```tsx:components/image.tsx
import NextImage from 'next/image';

export function Image(props) {
  return <NextImage {...props} />;
}
```

```tsx:components/__mocks__/image.tsx
export function Image(props) {
  return <img {...props} />;
}
```

```tsx:user-avatar.tsx
vi.mock('@/components/image');
```
