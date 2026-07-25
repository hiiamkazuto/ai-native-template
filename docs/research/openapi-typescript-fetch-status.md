# openapi-typescript / openapi-fetch 現状検証

> 調査日: 2026-07-26
> 対象バージョン: openapi-typescript **7.13.0** / openapi-fetch **0.17.0** / openapi-react-query **0.5.4**

---

## 1. 最新安定版とメジャーバージョンの方針

| パッケージ | 最新安定版 | リリース日 | ダウンロード/週 |
|---|---|---|---|
| `openapi-typescript` | **7.13.0** | 2026-02-11 | 5,740,919 |
| `openapi-fetch` | **0.17.0** | 2026-02-11 | 6,627,564 |
| `openapi-react-query` | **0.5.4** | 2026-02-11 | — |

- `openapi-typescript` は **v7.x** が現行メジャー。v6.x からの移行ガイドが公式ドキュメントに存在する。Breaking changes は主に「リモートスキーマ取得を Redocly CLI 経由に変更」「TypeScript AST 出力に変更」「`defaultNonNullable` をデフォルト有効化」の3点。
- `openapi-fetch` は **v0.x**（semver 上はプレリリース扱い）だが、ダウンロード数は openapi-typescript を上回り実質的に安定版として使われている。
- `openapi-react-query` も同様に **v0.x** で、openapi-fetch と同期リリースされている。

**出典:**
- リリース一覧: https://github.com/openapi-ts/openapi-typescript/releases
- npm openapi-typescript: https://www.npmjs.com/package/openapi-typescript
- npm openapi-fetch: https://www.npmjs.com/package/openapi-fetch
- 移行ガイド: https://openapi-ts.dev/migration-guide

---

## 2. 生成コード品質

### 2.1 `paths` 型の構造

生成コードは以下の構造を持つ:

```ts
export interface paths {
  "/blogposts/{post_id}": {
    parameters: { /* path/query/header params */ };
    get: {
      responses: {
        200: { content: { "application/json": { /* schema */ } } };
        500: { content: { "application/json": { /* schema */ } } };
      };
    };
    put: {
      requestBody: { content: { "application/json": { /* schema */ } } };
      responses: { /* … */ };
    };
  };
}
```

- `paths.<path>.parameters` — パスレベル・オペレーションレベルのパラメータをそのまま型化
- `paths.<path>.<method>.responses.<status>.content.<media-type>.schema` — レスポンスボディの型
- `paths.<path>.<method>.requestBody.content.<media-type>.schema` — リクエストボディの型

**出典:** https://openapi-ts.dev/introduction（Basic usage セクション）

### 2.2 enum の扱い

- デフォルトでは **string union** 型として生成: `"cat" | "dog" | "rabbit"`
- `--enum` フラグで **TypeScript enum** に変更可能
- `--enum-values` で enum 値を配列としてエクスポート
- `x-enum-varnames` / `x-enum-descriptions` 拡張に対応（JSDoc コメント生成）
- `--dedupe-enums` で enum の重複排除
- `--conditional-enums` で `x-enum-*` メタデータがある場合のみ true enum 生成

**出典:** https://openapi-ts.dev/cli#flags, https://openapi-ts.dev/advanced#enum-extensions

### 2.3 oneOf / allOf / nullable の扱い

- **`oneOf`** → TypeScript union 型に変換。ただし TS の union は排他型（XOR）ではないため、`oneOf` を他の composition と組み合わせると複雑な intersection & union 型になる。公式ドキュメントでは `oneOf` を単独で使うことを推奨。
- **`allOf`** → TypeScript intersection 型に変換
- **`nullable`** → `T | null` の union 型に変換（7.x では `defaultNonNullable` がデフォルト有効のため、`default` 値がある場合は non-nullable 扱い）
- **discriminator** → OpenAPI 3.1 の discriminator オブジェクトをサポート

**出典:** https://openapi-ts.dev/advanced#use-oneof-by-itself, https://openapi-ts.dev/migration-guide#defaultnonnullable-true-by-default

### 2.4 `any` 型の排除

> "openapi-typescript will **never produce an `any` type**. Anything not explicated in your schema may as well not exist."

スキーマ未定義のプロパティは `Record<string, never>` または `Record<string, unknown>` として生成される。

**出典:** https://openapi-ts.dev/advanced#be-specific-in-your-schema

---

## 3. CLI フラグ一覧（strict モード相当）

| フラグ | デフォルト | 説明 |
|---|---|---|
| `--enum` | `false` | true TS enum を生成 |
| `--enum-values` | `false` | enum 値を配列としてエクスポート |
| `--alphabetize` | `false` | 型名をアルファベット順にソート |
| `--default-non-nullable` | **`true`** | `default` 値を持つスキーマを non-nullable として扱う |
| `--properties-required-by-default` | `false` | `required` 未指定でも全プロパティを required にする |
| `--additional-properties` | `false` | `additionalProperties: false` がないスキーマに任意プロパティを許可 |
| `--empty-objects-unknown` | `false` | プロパティ未定義の空オブジェクトに任意プロパティを許可 |
| `--array-length` | `false` | `minItems`/`maxItems` からタプル型を生成 |
| `--immutable` | `false` | 全プロパティ・配列を readonly にする |
| `--export-type` | `false` | `interface` ではなく `type` でエクスポート |
| `--exclude-deprecated` | `false` | deprecated フィールドを除外 |
| `--path-params-as-types` | `false` | paths オブジェクトの動的ルックアップを許可 |
| `--root-types` | `false` | `components` の型をトップレベルにエイリアス |
| `--make-paths-enum` | `false` | 全パスの `ApiPaths` enum を生成 |
| `--read-write-markers` | `false` | readOnly/writeOnly の `$Read<T>`/`$Write<T>` マーカーを生成 |
| `--check` | `false` | 生成済み型が最新かチェック（CI 用） |

**推奨 strict 設定:**
```bash
npx openapi-typescript schema.yaml -o schema.d.ts \
  --default-non-nullable \
  --properties-required-by-default \
  --export-type \
  --immutable \
  --array-length
```

**出典:** https://openapi-ts.dev/cli#flags

---

## 4. TypeScript 厳格設定下での挙動

### 4.1 `noUncheckedIndexedAccess`

公式ドキュメントで **Highly recommended** として明示的に推奨されている。`additionalProperties` のキーが `T | undefined` として型付けされ、null reference エラーを防止する。

> "Enable `compilerOptions.noUncheckedIndexedAccess` in TSConfig so any `additionalProperties` key will be typed as `T | undefined`."

**出典:** https://openapi-ts.dev/advanced#enable-nouncheckedindexedaccess-in-tsconfig, https://openapi-ts.dev/openapi-fetch#getting-started

### 4.2 `exactOptionalPropertyTypes`

公式ドキュメントでの言及は **未確認**。openapi-fetch の README やドキュメントでは `exactOptionalPropertyTypes` に関する記述は見当たらない。

ただし、生成コードではプロパティに `?` オプショナルトークンが付与されるため、`exactOptionalPropertyTypes` を有効にした場合、`undefined` の明示的な代入が必要になる可能性がある。これは TypeScript の一般的な制約であり、openapi-typescript 固有の問題ではない。

**結論:** 未確認。導入前に `exactOptionalPropertyTypes: true` 環境で生成コードをビルドし、型エラーが出ないか検証が必要。

**出典:** 該当する一次資料なし

---

## 5. Vite HMR との統合

### 5.1 型生成の watch モード

公式ドキュメントに Vite HMR に関する記述は **未確認**。openapi-typescript CLI には `--watch` フラグは存在しない。

### 5.2 推奨ワークフロー

1. **ビルド時生成**: `package.json` の `prebuild` / `predev` スクリプトで `npx openapi-typescript` を実行
2. **CI で整合性チェック**: `--check` フラグで生成済み型がスキーマと一致するか検証
3. **Vite の `tsconfig.json` 設定**: `module: "ESNext"`, `moduleResolution: "Bundler"` が必要

生成コードは `.d.ts` ファイルであるため、HMR に直接影响しない。型定義の変更は TypeScript の型チェック時に反映され、Vite の HMR は実行時コード（`.ts`/`.tsx`）の変更で発火する。

**出典:** https://openapi-ts.dev/introduction#setup, https://openapi-ts.dev/cli#flags

---

## 6. React Query / TanStack Query との統合

### 6.1 `openapi-react-query` の存在

**`openapi-react-query`** が公式パッケージとして存在する。1kb の薄いラッパーで、`@tanstack/react-query` の上に型安全なラッパーを提供する。

- 最新版: **0.5.4**（2026-02-11 リリース）
- openapi-fetch と同期リリース

**出典:** https://openapi-ts.dev/openapi-react-query/, https://github.com/openapi-ts/openapi-typescript/releases

### 6.2 提供される Hook

| Hook | 説明 |
|---|---|
| `$api.useQuery(method, path, options?, queryOptions?)` | 通常のクエリ。query key は `[method, path, params]` |
| `$api.useMutation(method, path, queryOptions?)` | ミューテーション。mutation key は `[method, path]` |
| `$api.useSuspenseQuery(…)` | Suspense 対応クエリ |
| `$api.useInfiniteQuery(…)` | 無限スクロール用クエリ |
| `queryOptions(…)` | `queryOptions` ヘルパー |

### 6.3 使用例

```ts
import createFetchClient from "openapi-fetch";
import createClient from "openapi-react-query";
import type { paths } from "./my-openapi-3-schema";

const fetchClient = createFetchClient<paths>({ baseUrl: "https://myapi.dev/v1/" });
const $api = createClient(fetchClient);

// useQuery
const { data, error, isLoading } = $api.useQuery("get", "/users/{user_id}", {
  params: { path: { user_id: 5 } },
});

// useMutation
const { mutate } = $api.useMutation("patch", "/users");
```

### 6.4 特徴

- `data` と `error` が完全に型付けされる
- query key が自動的に `[method, path, params]` になるため、手動での key 管理が不要
- openapi-fetch のミドルウェア・認証機能をそのまま利用可能

**出典:** https://openapi-ts.dev/openapi-react-query/use-query, https://openapi-ts.dev/openapi-react-query/use-mutation

---

## まとめ

| 調査項目 | 結論 |
|---|---|
| 最新安定版 | openapi-typescript 7.13.0 / openapi-fetch 0.17.0 / openapi-react-query 0.5.4 |
| 生成コード品質 | paths/responses/requestBody を完全型化。enum/oneOf/allOf/nullable に対応。`any` を生成しない |
| strict モード | `--default-non-nullable`（デフォルト有効）、`--properties-required-by-default`、`--enum`、`--alphabetize` 等が利用可能 |
| TS 厳格設定 | `noUncheckedIndexedAccess` は公式推奨。`exactOptionalPropertyTypes` は未確認 |
| Vite HMR | 公式言及なし。`.d.ts` 生成のため HMR に直接影响しない。ビルド時生成 + CI `--check` で対応 |
| React Query 統合 | `openapi-react-query` が公式提供。useQuery/useMutation/useSuspenseQuery/useInfiniteQuery をサポート |
