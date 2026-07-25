# @hono/zod-openapi 現状検証

> 調査日: 2026-07-26

---

## 1. 最新安定版と Hono 本体のバージョン互換性

| パッケージ | 最新安定版 | peerDependencies |
|---|---|---|
| `@hono/zod-openapi` | **1.5.1** (2026-07-15 公開) | `hono >=4.10.0`, `zod ^4.0.0` |
| `hono` | **4.12.32** (2026-07-25 公開) | — |
| `@hono/node-server` | **2.0.11** (2026-07-21 公開) | — |

- `@hono/zod-openapi@1.4.0` で `peerDependencies.hono` が `>=4.10.0` に引き上げられた。これは `@hono/zod-validator` 側の `MiddlewareHandler` 4 引数シグネチャ（Hono v4.10.0 導入）への対応が理由。
- Hono 4.12.32 は peer range `>=4.10.0` を満たすため、互換性あり。

> 出典: https://www.npmjs.com/package/@hono/zod-openapi
> 出典: https://www.npmjs.com/package/hono
> 出典: https://github.com/honojs/middleware/blob/main/packages/zod-openapi/CHANGELOG.md (1.4.0 エントリ)

---

## 2. Zod 4 対応状況

### 依存関係

`@hono/zod-openapi@1.5.1` の `package.json`:

```json
"peerDependencies": {
  "hono": ">=4.10.0",
  "zod": "^4.0.0"
}
```

**Zod 4 専用** である。Zod 3 はサポート対象外。

> 出典: https://raw.githubusercontent.com/honojs/middleware/main/packages/zod-openapi/package.json

### CHANGELOG の記録

`@hono/zod-openapi@1.0.0` の Major Changes:

> feat: support Zod v4 — Zod OpenAPI has been migrated the Zod version from v3 to v4. As a result, the `zod` in `peerDependencies` has been updated to 4.0.0 or higher.

> 出典: https://github.com/honojs/middleware/blob/main/packages/zod-openapi/CHANGELOG.md (1.0.0)

### Zod 4 の新記法 (`z.email()` / `z.uuid()` 等) の可用性

Zod 4 ではトップレベル API として以下のスキーマが提供される:

- `z.email()` — メールアドレス検証
- `z.uuid()` / `z.uuidv4()` / `z.uuidv6()` / `z.uuidv7()` — UUID 検証
- `z.url()` / `z.httpUrl()` — URL 検証
- `z.ipv4()` / `z.ipv6()` — IP アドレス検証
- `z.jwt()` — JWT 検証
- `z.e164()` — 電話番号検証
- `z.iso.date()` / `z.iso.time()` / `z.iso.datetime()` / `z.iso.duration()` — ISO 日時検証

**这些の API は Zod 4 の `z` オブジェクトそのものに属する** ため、`@hono/zod-openapi` から `import { z } from '@hono/zod-openapi'` して得られる `z` オブジェクト経由でそのまま使用可能。

ただし、`@hono/zod-openapi` が内部で使用する `@asteasolutions/zod-to-openapi` (v8.5.0+) がこれらの Zod 4 スキーマを OpenAPI スキーマに正しく変換できるかは、`zod-to-openapi` 側の対応状況に依存する。

`@asteasolutions/zod-to-openapi@9.1.0` の README によると、Zod v4 の `.meta()` メソッドに対応し、`z.email()`, `z.uuid()` 等の string format を `format` プロパティとして正しく OpenAPI スキーマへ変換する。サポート対象の string format には `.email()`, `.uuid()`, `.url()`, `.datetime()`, `.date()`, `.time()`, `.duration()`, `.ip()`, `.cidrv4()`, `.cidrv6()`, `.base64url()`, `.cuid()`, `.cuid2()`, `.ulid()`, `.emoji()` が含まれる。

> 出典: https://zod.dev/api (String formats セクション)
> 出典: https://www.npmjs.com/package/@asteasolutions/zod-to-openapi

---

## 3. @hono/node-server との production-ready 判定

| 項目 | 値 |
|---|---|
| `@hono/node-server` 最新版 | 2.0.11 |
| 週間ダウンロード数 | 46,503,974 |
| Node.js 要件 | >= 20.x |
| WebSocket 対応 | `ws` パッケージ + `upgradeWebSocket` で対応 |
| 静的ファイル配信 | `@hono/node-server/serve-static` で対応 |
| HTTPS | `createServer` オプションで `node:https` を渡せる |

`@hono/node-server` は Hono 本体と同じ作者（yusukebe）がメンテナンスし、Hono の公式ドキュメントでも Node.js ランタイム用の推奨アダプターとして記載されている。週間 4600 万超のダウンロードがあり、production-ready と判断できる。

**注意点**: `@hono/zod-openapi` 自体はランタイムに依存しない。Cloudflare Workers / Bun / Deno / Node.js いずれでも同じコードが動作する。`@hono/node-server` は Node.js で Hono を動かすためのアダプターであり、`@hono/zod-openapi` との直接的な互換性問題はない。

> 出典: https://www.npmjs.com/package/@hono/node-server
> 出典: https://hono.dev (公式ドキュメント)

---

## 4. TypeScript 7 との型生成の整合性

### 現状

`@hono/zod-openapi@1.5.1` の `devDependencies` には:

```json
"typescript": "npm:@typescript/typescript6@^6.0.2"
```

つまり開発元は **TypeScript 6 系** で開発・テストしている。TypeScript 7 (現在の最新安定版は **7.0.2**) での動作確認は公式には行われていない。

> 出典: https://raw.githubusercontent.com/honojs/middleware/main/packages/zod-openapi/package.json
> 出典: https://www.npmjs.com/package/typescript (7.0.2)

### 既知の課題: tsgo と `.openapi()` チェーンの型計算コスト

GitHub Issue #1918 (`tsgo causes major issues with @hono/zod-openapi route chain accumulation`) によると:

- `OpenAPIHono` で `.openapi(route, handler)` をチェーンすると、型 generics が累積する
- tsc 5.9.3 と tsgo (TypeScript 7 の native port) の両方で、チェーンパターンは `.route()` マージパターンより **約 3 倍** 型インスタンス化コストが高い
- 特に `hc<>` RPC クライアントとの組み合わせで tsgo のコストが顕著に増大する（tsc で +0.4M → tsgo で +5.1M の型インスタンス化）

**推奨ワークアラウンド** (Issue 内で実証済み):

```ts
// チェーンではなく .route() マージを使う
export const app = new OpenAPIHono()
  .route("/", new OpenAPIHono().openapi(routeA, handlerA))
  .route("/", new OpenAPIHono().openapi(routeB, handlerB));
```

このパターンで型インスタンス化を **約 1/3** に削減できる。

**結論**: TypeScript 7 (tsgo) での strict 設定下での動作は、チェーンパターンを避ければ問題ない可能性が高いが、**公式での検証は未実施**。「未確認」とする。ただし上記ワークアラウンドが存在するため、リスクは軽減可能。

> 出典: https://github.com/honojs/middleware/issues/1918

---

## 5. ルート定義 → OpenAPI YAML 出力の API 仕様

### `app.doc()` — OpenAPI v3.0 出力

```ts
app.doc('/doc', {
  openapi: '3.0.0',
  info: {
    version: '1.0.0',
    title: 'My API',
  },
})
```

- 第 1 引数: エンドポイントパス (例: `/doc`)
- 第 2 引数: OpenAPI オブジェクト設定 (または `(c) => OpenAPIObject` のコールバック)
- レスポンスは JSON 形式で返却される
- コールバック形式で `c.req.url` から動的に `servers` を設定可能

### `app.doc31()` — OpenAPI v3.1 出力

```ts
app.doc31('/docs', {
  openapi: '3.1.0',
  info: { title: 'foo', version: '1' }
})
```

### `app.getOpenAPI31Document()` — スキーマオブジェクトとして取得

```ts
const spec = app.getOpenAPI31Document(
  { openapi: '3.1.0', info: { title: 'foo', version: '1' } },
  { unionPreferredType: 'oneOf' } // オプション
)
```

### 特定ルートの除外

```ts
const route = createRoute({
  // ...
  hide: true, // OpenAPI ドキュメントから除外
})
```

### OpenAPI バージョンの固定値

- `app.doc()` で `'3.0.0'` を指定可能
- `app.doc31()` で `'3.1.0'` を指定可能
- バージョンはユーザーが明示的に指定する必要がある（デフォルト値は設定側による）

> 出典: https://www.npmjs.com/package/@hono/zod-openapi (README)

---

## 6. Hono ルートと Zod スキーマを 1 対 1 で対応させるベストプラクティス

### 推奨パターン

1 つのルート定義 (`createRoute`) に 1 つの Zod スキーマを紐付ける:

```ts
// schemas/user.ts
const UserSchema = z.object({
  id: z.string().openapi({ example: '123' }),
  name: z.string().openapi({ example: 'John Doe' }),
}).openapi('User')

// routes/users.ts
const getUserRoute = createRoute({
  method: 'get',
  path: '/users/{id}',
  request: { params: ParamsSchema },
  responses: {
    200: {
      content: { 'application/json': { schema: UserSchema } },
      description: 'Retrieve the user',
    },
  },
})

// handler とルートをペアで定義
app.openapi(getUserRoute, (c) => {
  const { id } = c.req.valid('param')
  return c.json({ id, name: 'John' }, 200)
})
```

### バッチ登録パターン (v1.3.0+)

```ts
import { defineOpenAPIRoute, createRoute, z } from '@hono/zod-openapi'

const getUserRoute = defineOpenAPIRoute({
  route: createRoute({ /* ... */ }),
  handler: (c) => { /* ... */ },
})

const createUserRoute = defineOpenAPIRoute({
  route: createRoute({ /* ... */ }),
  handler: (c) => { /* ... */ },
})

// 一括登録
app.openapiRoutes([getUserRoute, createUserRoute] as const)
```

### 大規模アプリ向け `.route()` マージパターン

型計算コストを抑えるため、各ルートを個別の `OpenAPIHono` インスタンスで登録し `.route()` でマージする:

```ts
export const app = new OpenAPIHono()
  .route("/", new OpenAPIHono().openapi(routeA, handlerA))
  .route("/", new OpenAPIHono().openapi(routeB, handlerB))
```

> 出典: https://www.npmjs.com/package/@hono/zod-openapi (README)
> 出典: https://github.com/honojs/middleware/issues/1918

---

## まとめ

| 調査項目 | 結論 |
|---|---|
| 最新安定版 | `@hono/zod-openapi@1.5.1` / `hono@4.12.32` — 互換性あり |
| Zod 4 対応 | **Zod 4 専用** (`peerDependencies: zod ^4.0.0`)。`z.email()`, `z.uuid()` 等の新記法は `zod-to-openapi@9.1.0` 経由で OpenAPI スキーマに変換可能 |
| @hono/node-server | **production-ready**。週間 4600 万 DL、公式推奨 |
| TypeScript 7 | **未確認**。開発元は TS 6 系で動作。tsgo での型計算コスト問題が Issue #1918 で報告済み。`.route()` マージパターンで回避可能 |
| OpenAPI 出力 API | `app.doc()` (v3.0), `app.doc31()` (v3.1), `app.getOpenAPI31Document()` (スキーマオブジェクト) |
| ルート/スキーマ対応 | `createRoute` + `defineOpenAPIRoute` + `openapiRoutes` バッチ登録が推奨 |
