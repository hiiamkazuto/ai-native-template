# TypeScript 7 現状検証

> 調査日: 2026-07-26
> 目的: `ai-native-template` で TypeScript 7 の exact version pin 採用の可否を一次資料で判断する

---

## 1. 正式リリース状況

**結論: GA 済み（2026-07-08）。本番利用可能。**

| マイルストーン | 日付 | 出典 |
|---|---|---|
| Beta | 2026-04-21 | [Announcing TypeScript 7.0 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-beta/) |
| RC | 2026-06-18 | [Announcing TypeScript 7.0 RC](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/) |
| **GA** | **2026-07-08** | [Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) |

- npm 最新版: **7.0.2**（2026-07-09 公開）
  - 出典: [npm typescript](https://www.npmjs.com/package/typescript)
- GitHub リリースページでは TS 6.0.3 が Latest タグとして表示されているが、npm registry 上では `7.0.2` が `latest` dist-tag を保持
  - 出典: [microsoft/TypeScript releases](https://github.com/microsoft/TypeScript/releases)

---

## 2. 実速度向上の公式数値

**結論: フルビルドで 8x–12x、エディタ初回エラー表示で 13x+ 高速化。**

GA ブログ掲載の公式ベンチマーク（`--checkers 4` デフォルト設定）:

| コードベース | TS 6 | TS 7 | 高速化倍率 |
|---|---|---|---|
| vscode | 125.7s | 10.6s | **11.9x** |
| sentry | 139.8s | 15.7s | 8.9x |
| bluesky | 24.3s | 2.8s | 8.7x |
| playwright | 12.8s | 1.47s | 8.7x |
| tldraw | 11.2s | 1.46s | 7.7x |

- `--checkers 8` での最大値: vscode **16.7x**
- メモリ使用量: 6%〜26% 削減
- VS Code でエラー含むファイルを開いた際の初回エラー表示: 17.5s → 1.3s（**13x+**）
- Slack CI: type-check 7.5分 → 1.25分
- 出典: [Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)

---

## 3. 主要 `@types/*` パッケージの TS 7 互換性

**結論: 型定義パッケージ自体は TS 7 と互換。ただし TS 7 のデフォルト変更により `tsconfig` 調整が必要。**

| パッケージ | 最新版 | 最終更新 | 出典 |
|---|---|---|---|
| `@types/node` | 26.1.1 | 2026-07-08 | [npm @types/node](https://www.npmjs.com/package/@types/node) |
| `@types/react` | 19.2.17 | 2026-06-05 | [npm @types/react](https://www.npmjs.com/package/@types/react) |
| `@types/react-dom` | 19.2.3 | 2025-11-12 | [npm @types/react-dom](https://www.npmjs.com/package/@types/react-dom) |

注意点:
- `@types/*` は純粋な `.d.ts` 型定義であり、TS コンパイラの実装言語（Go 化）とは独立。TS 7 が型定義を解釈する方式に変更がない限り互換性は維持される。
- TS 7 では `types` コンパイラオプションのデフォルトが `[]` に変更されたため、`@types/node` 等を明示的に `tsconfig.json` の `"types"` に列挙する必要がある。
  - 出典: [Announcing TypeScript 7.0 – Updates Since 5.x](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)
- DefinitelyTyped 側で TS 7 を明示的にターゲットした互換性テストや CI は未確認。

---

## 4. exact version pin の公式推奨

**結論: 公式には推奨されていない。caret (`^`) がデフォルト運用。**

- GA ブログのインストール例は `npm install -D typescript`（latest を取得）であり、pin の言及なし。
- npm registry 上の `typescript` パッケージは `7.0.2` が latest。`package.json` でのデフォルト表現は `^7.0.2`（caret）。
- TypeScript チームはセマンティックバージョニングに従い、パッチリリース（7.0.x）でバグ修正のみを行う公開方針。
- ただし、TS 7.0 は Go 移植というアーキテクチャ刷新であり、エコシステム全体のツールチェーン互換が完全ではないため、**現時点では exact pin の方が安全側** という判断は合理的。
- 出典: [npm typescript](https://www.npmjs.com/package/typescript), [Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)

---

## 5. `type: "module"` / `moduleResolution` まわりの挙動変更

**結論: `moduleResolution: "bundler"` / `"nodenext"` は引き続きサポート。`"node"` / `"node10"` / `"classic"` は廃止。`"type": "module"` 自体への直接言及はない。**

TS 7.0 で廃止・エラー化された設定:

| 廃止された設定 | 推奨代替 | 出典 |
|---|---|---|
| `moduleResolution: node / node10` | `nodenext` または `bundler` | [Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) |
| `moduleResolution: classic` | `bundler` または `nodenext` | 同上 |
| `module: amd, umd, systemjs, none` | `esnext` または `preserve` | 同上 |
| `target: es5` | ES2015 以上 | 同上 |
| `esModuleInterop: false` | `true` のみ（false 設定不可） | 同上 |
| `baseUrl` | `paths` をプロジェクトルート相対に変更 | 同上 |

デフォルト値の変更:
- `module` デフォルト: **`esnext`**
- `strict` デフォルト: **`true`**
- `types` デフォルト: **`[]`**（明示列挙が必要）
- `rootDir` デフォルト: **`./`**

`"type": "module"` に関する直接の挙動変更言及は GA ブログにない。ただし `module` デフォルトが `esnext` になったことは、ESM プロジェクトとの親和性が高い。

---

## 総合判断

| 評価軸 | 判定 |
|---|---|
| GA 済みか | **Yes** — 2026-07-08 GA、7.0.2 が latest |
| 速度面で実用的か | **Yes** — 8x–12x の公式実測、大規模企業で実績 |
| `@types/*` 互換 | **Yes** — 型定義は独立。ただし `types` 配列の明示が必要 |
| exact pin 推奨 | **No（公式推奨なし）** — ただし現時点では保守的判断として妥当 |
| `moduleResolution` 影響 | **要対応** — `bundler` / `nodenext` への移行は必須 |
| 埋め込み言語（Vue/Svelte/Astro） | **非対応** — TS 7.0 は API を提供せず、7.1 で対応予定 |

### 推奨アクション

1. `"typescript": "7.0.2"` で exact pin する（保守的運用として妥当。7.0.x パッチ追跡は手動で行う）
2. `tsconfig.json` で `moduleResolution: "bundler"` を明示
3. `"types": ["node"]` 等、必要な `@types` を明示的に列挙
4. Vue/Svelte/Astro 等の埋め込み言語を使用する場合は TS 6.0 との併用を検討（`@typescript/typescript6` パッケージ）
5. TS 7.0 は API 未提供のため、`typescript-eslint` 等のツールは TS 6.0 API 経由での動作が必要
