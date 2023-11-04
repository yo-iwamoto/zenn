---
title: 'Vitest での個人的なモックパターンまとめ'
emoji: '🐝'
type: 'tech'
topics: ['vitest', 'test', 'recoil']
published: false
---

## はじめに

テスト領域や設計にもよりますが、特にユニットテストではモックは頻出しますね。
Vitest（あるいは Jest）ではモックの手法が非常に多いので、この記事では自分がよく使うモックのパターンをまとめておきます。

### 触れないこと

- モックするべきところ・そうでないところ

## テスト対象の依存モジュールのモック

以下のようなコンポーネントをテスト対象とします。

```tsx:product-card.tsx
import { useProductQuery } from './use-product-query';

export function ProductCard() {
  const product = useProductQuery();

  return (
    <div>
      <img src={product.imageUrl} alt={product.imageCaption} height={400} width={400} />
      <h1>{product.name}</h1>
    </div>
  );
}
```

基本的には `vi.mock()` で完結します。

```tsx:product-card.test.tsx
vi.mock('./use-product-query', () => ({
  useProductQuery: () => ({
    imageUrl: 'https://example.com/image.png',
    imageCaption: 'image caption',
    name: 'product name',
  }),
}));
```
