# oxlint / oxfmt 現状検証レポート

> 調査日: 2026-07-26  
> 対象: `frontend/`, `backend/` へのテンプレート導入意思決定

---

## 1. 最新安定版とリリースサイクル

### 現時点の最新安定版

| ツール | バージョン | リリース日 |
|--------|-----------|-----------|
| **oxlint** | v1.75.0 | 2026-07-21 |
| **oxfmt** | v0.60.0 | 2026-07-21 |

- 出典: [oxc-project/oxc Releases — apps_v1.75.0](https://github.com/oxc-project/oxc/releases/tag/apps_v1.75.0)

### リリースサイクル

直近のリリース履歴（apps リリース = oxlint + oxfmt 同時リリース）:

| リリース | 日付 | 間隔 |
|----------|------|------|
| v1.75.0 / v0.60.0 | 2026-07-21 | — |
| v1.74.0 / v0.59.0 | 2026-07-14 | ~7日 |
| v1.73.0 / v0.58.0 | 直前 | ~7日 |
| v1.72.0 / v0.57.0 | 直前 | ~7日 |
| v1.71.0 / v0.56.0 | 直前 | ~7日 |

- **oxlint**: SemVer を採用。v1.x は安定版。Minor バージョンで新ルール追加、Patch でバグ修正。Major バージョンは CLI・設定フォーマットの破壊的変更のみ。
- **oxfmt**: v0.x（Beta フェーズ）。Prettier 互換性の改善が継続中。
- 出典: [Versioning policy — Oxlint](https://oxc.rs/docs/guide/usage/linter/versioning.html)
- 出典: [oxc-project/oxc Releases](https://github.com/oxc-project/oxc/releases)

### マイルストーン

- **2025-06-10**: Oxlint v1.0 Stable リリース
- **2025-12-01**: Oxfmt Alpha リリース
- **2026-02-24**: Oxfmt Beta リリース
- **2026-03-11**: Oxlint JS Plugins Alpha リリース
- **2026-07-22**: Type-Aware Linting Stable リリース（tsgolint v7、TypeScript v7.0.2 ベース）
- 出典: [oxc.rs/blog](https://oxc.rs/blog)

---

## 2. oxlint のルールカバレッジ

### ルール総数

**844+ ルール**（公式ドキュメントより）。ESLint core + 主要プラグインのルールをネイティブ実装。

- 出典: [Oxlint Overview](https://oxc.rs/docs/guide/usage/linter.html)

### ビルトインプラグイン一覧

| プラグイン名 | デフォルト有効 | 由来 |
|-------------|:---:|------|
| `eslint` | ✅ | ESLint core rules |
| `typescript` | ✅ | typescript-eslint（type-aware ルール含む） |
| `unicorn` | ✅ | eslint-plugin-unicorn |
| `react` | ❌ | eslint-plugin-react / react-hooks / react-refresh / React Compiler |
| `react-perf` | ❌ | eslint-plugin-react-perf |
| `nextjs` | ❌ | @next/eslint-plugin-next |
| `oxc` | ✅ | Oxc 独自ルール + deepscan 由来ルール |
| `import` | ❌ | eslint-plugin-import / import-x |
| `jsdoc` | ❌ | eslint-plugin-jsdoc |
| `jsx-a11y` | ❌ | eslint-plugin-jsx-a11y |
| `node` | ❌ | eslint-plugin-n |
| `promise` | ❌ | eslint-plugin-promise |
| `jest` | ❌ | eslint-plugin-jest |
| `vitest` | ❌ | @vitest/eslint-plugin |
| `vue` | ❌ | eslint-plugin-vue |

- 出典: [Built-in Plugins — Oxlint](https://oxc.rs/docs/guide/usage/linter/plugins.html)

### `eslint:recommended` 相当

Oxlint はデフォルトで "correctness" カテゴリのルールを有効化。これは `eslint:recommended` に相当する高シグナルなルールセット。

- 出典: [Oxlint Overview — Correctness-focused defaults](https://oxc.rs/docs/guide/usage/linter.html)

### Type-Aware Linting（2026-07-22 Stable）

- tsgolint v7（TypeScript v7.0.2 ベース）
- typescript-eslint の 61 ルール中 **59 ルール** をカバー
- 残り 2 ルール未対応（具体的ルール名は未確認）
- ESLint + typescript-eslint と比較して **12〜18倍高速**
- 出典: [Type-Aware Linting Stable — Blog](https://oxc.rs/blog/2026-07-22-type-aware-linting-stable)

### React / Hono / TypeScript 固有ルール

| カテゴリ | 対応状況 |
|---------|---------|
| React (hooks, JSX, refresh, compiler) | ✅ `react` プラグインで対応 |
| TypeScript (型チェック不要ルール) | ✅ `typescript` プラグインで対応 |
| TypeScript (型チェック必要ルール) | ✅ `--type-aware` フラグ + tsgolint v7 |
| Hono 固有ルール | ❌ Hono 固有の lint ルールは存在しない（一般的な Node.js/TypeScript ルールで十分） |

---

## 3. 未対応の有名 ESLint ルール

### 対応済みプラグイン（ネイティブ実装）

| ESLint プラグイン | oxlint 対応 | 備考 |
|------------------|:---:|------|
| `@typescript-eslint/*` | ✅ | 844+ ルール中大部分。type-aware 59/61 ルール |
| `eslint-plugin-unicorn` | ✅ | デフォルト有効 |
| `eslint-plugin-import` | ✅ | `import` プラグイン（`--import-plugin`） |
| `eslint-plugin-react` | ✅ | `react` プラグイン |
| `eslint-plugin-react-hooks` | ✅ | `react` プラグインに統合 |
| `eslint-plugin-jsx-a11y` | ✅ | `jsx-a11y` プラグイン |
| `eslint-plugin-jest` | ✅ | `jest` プラグイン |
| `@vitest/eslint-plugin` | ✅ | `vitest` プラグイン |
| `eslint-plugin-n` | ✅ | `node` プラグイン |
| `eslint-plugin-promise` | ✅ | `promise` プラグイン |
| `eslint-plugin-jsdoc` | ✅ | `jsdoc` プラグイン |
| `@next/eslint-plugin-next` | ✅ | `nextjs` プラグイン |
| `eslint-plugin-vue` | ✅ | `vue` プラグイン（script タグのみ） |

- 出典: [Built-in Plugins — Oxlint](https://oxc.rs/docs/guide/usage/linter/plugins.html)

### 未対応 / 制限あり

| プラグイン/ルール | 状態 | 備考 |
|-----------------|------|------|
| カスタム ESLint プラグイン（企業独自等） | ⚠️ JS Plugins (Alpha) | ESLint v9 API 互換の JS プラグインとして動作可能。Alpha 段階 |
| `eslint-plugin-perfectionist` | ⚠️ 部分対応 | oxfmt の `sortImports` が同アルゴリズム採用。lint ルールとしては未確認 |
| `eslint-plugin-functional` | ❌ 未確認 | 一次資料に記載なし |
| `eslint-plugin-security` | ❌ 未確認 | 一次資料に記載なし |
| `eslint-plugin-no-unsanitized` | ❌ 未確認 | 一次資料に記載なし |

- 出典: [Migrate from ESLint — Oxlint](https://oxc.rs/docs/guide/usage/linter/migrate-from-eslint.html)
- 出典: [Rule/plugin support](https://oxc.rs/docs/guide/usage/linter/migrate-from-eslint.html#rule-plugin-support)
- 出典: [Meta issue — Rule implementation status](https://github.com/oxc-project/oxc/issues/481)

### 移行ツール

- `@oxlint/migrate`: ESLint flat config → `.oxlintrc.json` 自動変換
- `eslint-plugin-oxlint`: ESLint 側の重複ルールを無効化（段階的移行用）
- JS Plugins: 未対応プラグインを ESLint 互換 API で利用可能（Alpha）
- 出典: [Migrate from ESLint](https://oxc.rs/docs/guide/usage/linter/migrate-from-eslint.html)

---

## 4. oxfmt の Prettier 互換性

### 互換性レベル

- **Prettier v3.8 と互換**。JavaScript/TypeScript のフォーマット差分はバグとして扱われる
- **Prettier の conformance test に 100% 合格**（JS/TS）
- 出典: [Oxfmt Overview — Prettier-compatible](https://oxc.rs/docs/guide/usage/formatter.html)
- 出典: [Migrate from Prettier](https://oxc.rs/docs/guide/usage/formatter/migrate-from-prettier.html)

### サポートされる Prettier オプション

| オプション | 対応 | デフォルト | 備考 |
|-----------|:---:|:---:|------|
| `printWidth` | ✅ | `100` | Prettier は `80`。**注意** |
| `tabWidth` | ✅ | `2` | |
| `useTabs` | ✅ | `false` | |
| `semi` | ✅ | `true` | |
| `singleQuote` | ✅ | `false` | |
| `jsxSingleQuote` | ✅ | `false` | |
| `trailingComma` | ✅ | `"all"` | |
| `bracketSpacing` | ✅ | `true` | |
| `bracketSameLine` | ✅ | `false` | |
| `arrowParens` | ✅ | `"always"` | |
| `endOfLine` | ✅ | `"lf"` | `"auto"` は未サポート |
| `quoteProps` | ✅ | `"as-needed"` | |
| `proseWrap` | ✅ | `"preserve"` | |
| `htmlWhitespaceSensitivity` | ✅ | `"css"` | |
| `singleAttributePerLine` | ✅ | `false` | |
| `embeddedLanguageFormatting` | ✅ | `"auto"` | |
| `objectWrap` | ✅ | `"preserve"` | |
| `vueIndentScriptAndStyle` | ✅ | `false` | |
| `insertFinalNewline` | ✅ | `true` | |

- 出典: [Configuration file reference — Oxfmt](https://oxc.rs/docs/guide/usage/formatter/config-file-reference.html)

### oxfmt 独自拡張機能

| 機能 | 説明 | Prettier プラグイン代替 |
|------|------|----------------------|
| `sortImports` | import 文ソート | `eslint-plugin-perfectionist/sort-imports` |
| `sortTailwindcss` | Tailwind CSS クラスソート | `prettier-plugin-tailwindcss` |
| `sortPackageJson` | package.json キー並び替え（デフォルト有効） | `prettier-plugin-packagejson` |
| `jsdoc` | JSDoc コメント整形 | `prettier-plugin-jsdoc` |

- 出典: [Unsupported features — Oxfmt](https://oxc.rs/docs/guide/usage/formatter/unsupported-features.html)

### 未サポート機能

| 機能 | 状態 |
|------|------|
| `prettier` field in `package.json` | ❌ 未対応 |
| ネストされた `.editorconfig`（サブディレクトリ） | ❌ 未対応 |
| `experimentalTernaries` / `experimentalOperatorPosition` | ❌ 未対応 |
| Prettier プラグイン全般 | ❌ 未対応（ビルトイン代替あり） |

- 出典: [Unsupported features — Oxfmt](https://oxc.rs/docs/guide/usage/formatter/unsupported-features.html)

### Biome との比較

| 比較項目 | oxfmt | biome |
|---------|-------|-------|
| Prettier 互換性 | Prettier v3.8 JS/TS conformance 100% | 独自フォーマッタ（Prettier 非互換） |
| ベンチマーク | Prettier の約 **30倍高速**、Biome の約 **2倍高速** | Prettier の約 15倍高速 |
| 設定互換 | Prettier オプションをほぼそのまま利用 | 独自設定形式 |

- 出典: [Oxfmt Overview — Built for scale](https://oxc.rs/docs/guide/usage/formatter.html)

---

## 5. per-project 独立導入

### インストール手順

**frontend/**:
```bash
cd frontend/
pnpm add -D oxlint oxfmt
```

**backend/**:
```bash
cd backend/
pnpm add -D oxlint oxfmt
```

### 設定ファイル

| ツール | 設定ファイル名 | 備考 |
|--------|--------------|------|
| oxlint | `.oxlintrc.json` / `.oxlintrc.jsonc` / `oxlint.config.ts` / `oxlint.config.mts` | 自動検出 |
| oxfmt | `.oxfmtrc.json` / `.oxfmtrc.jsonc` / `oxfmt.config.ts` / `oxfmt.config.mts` | 自動検出 |

### ネスト設定（Monorepo パターン）

oxlint は **nested config** をサポート。各ディレクトリの `.oxlintrc.json` が nearest-config で解決される。

```
project/
├── .oxlintrc.json          ← 共有ベース設定
├── frontend/
│   └── .oxlintrc.json      ← extends で親設定を継承 + frontend 固有ルール
└── backend/
    └── .oxlintrc.json      ← extends で親設定を継承 + backend 固有ルール
```

frontend 例:
```json
{
  "extends": ["../.oxlintrc.json"],
  "plugins": ["react"],
  "rules": {
    "react/jsx-key": "error"
  }
}
```

backend 例:
```json
{
  "extends": ["../.oxlintrc.json"],
  "plugins": ["node"],
  "rules": {
    "no-console": "warn"
  }
}
```

- 出典: [Nested configuration files — Oxlint](https://oxc.rs/docs/guide/usage/linter/nested-config.html)

### package.json スクリプト

```json
{
  "scripts": {
    "lint": "oxlint",
    "lint:fix": "oxlint --fix",
    "fmt": "oxfmt",
    "fmt:check": "oxfmt --check"
  }
}
```

### キャッシュの競合有無

- **oxlint**: キャッシュ機能は公式ドキュメントに記載なし。ネイティブ Rust ベースで高速なためキャッシュ不要と推測。競合リスクなし。
- **oxfmt**: `--cache` / `--cache-strategy` フラグは現時点で未確認（Prettier にはあるが oxfmt の CLI ドキュメントには記載なし）。
- **結論**: 各プロジェクトが独立した `node_modules` を持つ場合、キャッシュ競合の問題は**発生しない**。Monorepo で共有 `node_modules` を使う場合でも、oxlint/oxfmt はグローバルステートを持たないため競合リスクは極めて低い。
- 出典: 未確認（キャッシュ機能の明記なし）

### `--type-aware` の制約

- `options.typeAware` / `options.typeCheck` は **root config でのみ有効**。nested config で設定するとエラー。
- 出典: [Nested configuration files](https://oxc.rs/docs/guide/usage/linter/nested-config.html)

---

## 6. Hono 公式プロジェクトでの採用事例

### 調査結果

Hono 公式リポジトリ（`honojs/hono`）の `package.json` を確認:

- **lint**: `eslint src runtime-tests build perf-measures benchmarks` — ESLint を使用
- **format**: `prettier --check --cache` — Prettier を使用
- **oxlint / oxfmt の採用**: **なし**

- 出典: [honojs/hono package.json](https://github.com/honojs/hono/blob/main/package.json)

### Oxlint 採用プロジェクト（公式発表）

Oxlint をプロダクションで使用しているプロジェクト:

- [elastic/kibana](https://github.com/elastic/kibana)
- [getsentry/sentry-javascript](https://github.com/getsentry/sentry-javascript)
- [renovatebot/renovate](https://github.com/renovatebot/renovate)
- [preactjs/preact](https://github.com/preactjs/preact)
- [date-fns/date-fns](https://github.com/date-fns/date-fns)
- [outline/outline](https://github.com/outline/outline)
- [PostHog/posthog](https://github.com/PostHog/posthog)
- [actualbudget/actual](https://github.com/actualbudget/actual)
- [cloudflare/agents](https://github.com/cloudflare/agents)

- 出典: [Projects using Oxlint](https://oxc.rs/docs/guide/usage/linter.html#projects-using-oxlint)

---

## 総合判断マトリクス

| 評価軸 | oxlint | oxfmt | 備考 |
|--------|--------|-------|------|
| 安定性 | ✅ v1.x (SemVer) | ⚠️ v0.x (Beta) | oxfmt はまだ Beta |
| パフォーマンス | ✅ ESLint の 50〜100倍 | ✅ Prettier の 30倍 | |
| ESLint ルールカバレッジ | ✅ 844+ ルール | — | |
| Prettier 互換性 | — | ✅ JS/TS 100% | printWidth デフォルト差異あり |
| TypeScript type-aware | ✅ tsgolint v7 (Stable) | — | 2026-07-22 に Stable |
| Hono での採用 | ❌ 未採用 | ❌ 未採用 | Hono は ESLint + Prettier 継続 |
| per-project 導入 | ✅ nested config 対応 | ✅ 自動検出 | |
| 移行ツール | ✅ @oxlint/migrate | ✅ --migrate=prettier | |
| JS プラグイン互換 | ⚠️ Alpha | — | 未対応プラグインのフォールバック |

---

## 推奨導入戦略

1. **oxlint**: 即時導入可能。`eslint:recommended` 相当 + 主要プラグインをカバー。`@oxlint/migrate` で既存 ESLint 設定を自動変換。
2. **oxfmt**: Beta 段階だが Prettier 互換性は高い。`printWidth` デフォルト値の差異（100 vs 80）に注意。`.oxfmtrc.json` で `printWidth: 80` を明示推奨。
3. **段階的移行**: `eslint-plugin-oxlint` を使って ESLint と oxlint を並行稼働させ、未対応ルールのみ ESLint でカバー。
4. **Hono**: 固有の lint ルールは不要。一般的な TypeScript + Node.js ルールセットで対応可能。
