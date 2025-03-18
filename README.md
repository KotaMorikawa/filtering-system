# プロダクトフィルタリングシステム

モダンなアプローチで実装された製品フィルタリングシステムです。Web 開発の基本原則を実践的に示すために作られています。

## 概要

このアプリケーションは、Next.js と Upstash Vector データベースを使用して構築された、高性能な製品フィルタリングシステムです。ユーザーが様々な条件（色、サイズ、価格など）で製品を絞り込み、検索結果をリアルタイムで確認できるインターフェースを提供します。

### デモ

![製品フィルタリングデモ](https://via.placeholder.com/800x400?text=Product+Filtering+Demo)

## 機能

- **複数条件フィルタリング**: 色、サイズ、価格などの複合条件での製品検索
- **リアルタイム更新**: フィルター変更時に即座に結果が更新
- **カスタム価格範囲**: スライダーで自由に価格範囲を設定可能
- **ソート機能**: 価格の昇順/降順でのソート
- **レスポンシブデザイン**: モバイルからデスクトップまであらゆるデバイスに対応
- **高速パフォーマンス**: ベクトルデータベースによる効率的な検索

## 技術スタック

- **フロントエンド**: Next.js 14, React 18, TypeScript
- **UI**: Tailwind CSS, Radix UI コンポーネント
- **状態管理**: React Hooks, TanStack Query
- **データベース**: Upstash Vector (ベクトルデータベース)
- **最適化**: デバウンス処理、効率的なクエリ構築

## アーキテクチャ

このアプリケーションは以下のアーキテクチャコンポーネントで構成されています：

1. **クライアントサイド（React/Next.js）**

   - ユーザーインターフェース
   - フィルター状態管理
   - API 呼び出しと結果表示

2. **サーバーサイド（Next.js API Routes）**

   - フィルター条件の検証
   - クエリ構築
   - データベース操作

3. **データベース（Upstash Vector）**
   - 製品データの格納
   - ベクトル検索による高速フィルタリング

## アクターとシステムの役割

このフィルターシステムには以下のアクターが存在し、それぞれが連携してパフォーマンスを最適化しています：

### 1. ユーザー（クライアント）

エンドユーザーは UI を通じてフィルター条件を操作し、リアルタイムなフィードバックを期待します。

### 2. フロントエンド（React/Next.js）

フロントエンドはユーザー入力を処理し、状態を管理します：

```tsx
// フィルター状態の管理（src/app/page.tsx）
const [filter, setFilter] = useState<ProductState>({
  color: ["beige", "blue", "green", "purple", "white"],
  price: { isCustom: false, range: DEFAULT_CUSTOM_PRICE },
  size: ["L", "M", "S"],
  sort: "none",
});

// フィルター適用のロジック例
const applyArrayFilter = ({
  category,
  value,
}: {
  category: keyof Omit<typeof filter, "price" | "sort">;
  value: string;
}) => {
  const isFilterApplied = filter[category].includes(value as never);

  if (isFilterApplied) {
    setFilter((prev) => ({
      ...prev,
      [category]: prev[category].filter((v) => v !== value),
    }));
  } else {
    setFilter((prev) => ({
      ...prev,
      [category]: [...prev[category], value],
    }));
  }

  _debouncedSubmit();
};
```

### 3. バックエンド API（Next.js API Routes）

```typescript
// フィルター条件の処理と検証（src/app/api/products/route.ts）
export const POST = async (req: NextRequest) => {
  try {
    const body = await req.json();

    const { color, price, size, sort } = ProductFilterValidator.parse(
      body.filter
    );

    const filter = new Filter();

    if (color.length > 0)
      color.forEach((color) => filter.add("color", "=", color));
    else if (color.length === 0) filter.addRaw("color", `color = ""`);

    if (size.length > 0) size.forEach((size) => filter.add("size", "=", size));
    else if (size.length === 0) filter.addRaw("size", `size = ""`);

    filter.addRaw("price", `price >= ${price[0]} AND price <= ${price[1]}`);

    // クエリの実行と結果の返却
    const products = await db.query({
      topK: 12,
      vector: [
        0,
        0,
        sort === "none"
          ? AVG_PRODUCT_PRICE
          : sort === "price-asc"
          ? 0
          : MAX_PRODUCT_PRICE,
      ],
      includeMetadata: true,
      filter: filter.hasFilter() ? filter.get() : undefined,
    });

    return new Response(JSON.stringify(products));
  } catch (err) {
    console.error(err);
    return new Response(JSON.stringify({ message: "Internal Server Error" }), {
      status: 500,
    });
  }
};
```

### 4. ベクトルデータベース（Upstash Vector）

製品データをベクトル形式で保存し、効率的な検索を可能にします：

```typescript
// 製品のデータ構造とベクトル変換（src/db/seed.ts）
const SIZE_MAP = {
  S: 0,
  M: 1,
  L: 2,
};

const COLOR_MAP = {
  white: 0,
  beige: 1,
  blue: 2,
  green: 3,
  purple: 4,
};

await db.upsert(
  products.map((product) => ({
    id: product.id,
    vector: [COLOR_MAP[product.color], SIZE_MAP[product.size], product.price],
    metadata: product,
  }))
);
```

## パフォーマンス最適化ポイント

アプリケーションのパフォーマンスを最適化するために、以下の手法を実装しています：

### 1. デバウンス処理

ユーザーのフィルター操作が頻繁に行われる場合でも、API リクエストの数を制限します：

```typescript
// デバウンス処理によるAPI呼び出しの最適化（src/app/page.tsx）
const debouncedSubmit = debounce(onSubmit, 400);
const _debouncedSubmit = useCallback(debouncedSubmit, []);
```

### 2. 効率的なクエリ構築

SQL ライクなフィルタークエリを動的に構築するカスタムクラスを実装：

```typescript
// 効率的なフィルタークエリ構築（src/app/api/products/route.ts）
class Filter {
  private filters: Map<string, string[]> = new Map();

  hasFilter() {
    return this.filters.size > 0;
  }

  add(key: string, operator: string, value: string | number) {
    const filter = this.filters.get(key) || [];
    filter.push(
      `${key} ${operator} ${typeof value === "number" ? value : `"${value}"`}`
    );
    this.filters.set(key, filter);
  }

  addRaw(key: string, rawFilter: string) {
    this.filters.set(key, [rawFilter]);
  }

  get() {
    const parts: string[] = [];
    this.filters.forEach((filter) => {
      const groupedValues = filter.join(` OR `);
      parts.push(`(${groupedValues})`);
    });
    return parts.join(" AND ");
  }
}
```

### 3. ベクトル検索の活用

製品属性を数値ベクトルとして表現し、高速な類似性検索を実現：

```typescript
// ベクトル検索による高速なソートとフィルタリング（src/app/api/products/route.ts）
const products = await db.query({
  topK: 12,
  vector: [
    0,
    0,
    sort === "none"
      ? AVG_PRODUCT_PRICE
      : sort === "price-asc"
      ? 0
      : MAX_PRODUCT_PRICE,
  ],
  includeMetadata: true,
  filter: filter.hasFilter() ? filter.get() : undefined,
});
```

### 4. スケルトンローディング

データ取得中にユーザーにビジュアルフィードバックを提供：

```tsx
// 商品一覧の条件付きレンダリング（src/app/page.tsx）
{
  products && products.length === 0 ? (
    <EmptyState />
  ) : products ? (
    products.map((product) => <Product product={product.metadata!} />)
  ) : (
    new Array(12).fill(null).map((_, i) => <ProductSkeleton key={i} />)
  );
}
```

## アーキテクチャの制限と最適化

- 現在のデータベース設計は製品検索に最適化されていますが、ユーザーデータや複雑なリレーショナルデータには適していません
- 大規模なカタログに対応するためには、ページネーションの実装が必要
- リアルタイム更新のパフォーマンスを確保するためのデバウンス処理を実装済み

## 開発環境のセットアップ

### 前提条件

- Node.js 18.x 以上
- pnpm（推奨）または npm/yarn

### インストール手順

1. 依存関係のインストール

   ```bash
   npm install
   ```

2. 環境変数の設定
   `.env.example`をコピーして`.env`を作成し、必要な環境変数を設定します。

   ```bash
   cp .env.example .env
   ```

3. 開発サーバーの起動

   ```bash
   npm run dev
   ```

4. データベースのシード
   ```bash
   npm run seed
   ```

## 将来の拡張計画

現在はフィルタリング機能のみを実装していますが、以下の機能拡張を計画しています：

- **ユーザー認証・ログイン機能**: セキュアなユーザーアカウント管理
- **お気に入り機能**: 製品をお気に入りに追加
- **レビュー・評価システム**: ユーザーレビューと評価
- **詳細な商品ページ**: 個別商品の詳細表示
- **カート・購入機能**: 完全な E コマース機能の実装

これらの拡張には、Upstash Vector に加えて、認証とユーザーデータを管理するための追加データベース（PostgreSQL、MongoDB 等）の導入が必要になります。
