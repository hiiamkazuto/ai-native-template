# ADR-0001: AGENTS.md を二段構成（ルート + frontend + backend）にする

- **Status**: Accepted
- **Date**: 2026-07-26
- **Issue**: [#7](https://github.com/hiiiamkazuto/ai-native-template/issues/7)
- **Resolution comment**: https://github.com/hiiiamkazuto/ai-native-template/issues/7#issuecomment-5079263467
- **Decided by**: hiiamkazuto (driving dev)

## Context

`ai-native-template` は AI 駆動開発スターターであり、`AGENTS.md` は AI エージェントに対する最重要な憲法ドキュメントになる。destination の制約として `frontend/` と `backend/` は **pnpm レベルで完全に独立した 2 プロジェクト** があるため、AGENTS.md の置き方もそれに合わせる必要がある。

未解決だった問い:

- AGENTS.md を **ルート 1 本** にするか、**ルート + 各 subdir の二段（または三段）** にするか
- ルートと subdir の **責務境界** をどこに引くか
- ルート AGENTS.md に **必ず含めるべき positive / negative guidance** は何か
- 新規 PJ 利用者の **Quickstart** はどう書くか
- AI エージェントが **境界を越えようとする提案**（共有コード新設、ルート pnpm workspace、openapi.yaml 直接編集 等）にどう対処させるか

## Decision

### 1. 構造: 二段構成

- `AGENTS.md`（ルート）
- `frontend/AGENTS.md`
- `backend/AGENTS.md`

`docs/AGENTS.md`（Spec 編集作法専用）は **不採用** — Spec 編集の頻度が読みより低いと見て過剰な細分化を避ける。

### 2. 責務境界（メタルール）

| レベル | 役割 | 語り方 |
|---|---|---|
| ルート | プロジェクト全体の「憲法」 | 何をしてはならないか、何が不変か |
| subdir | その世界の「現場ルール」 | この世界でどうやって動かすか |

- subdir 側にしか存在しない情報（具体的なコマンド、設定ファイル名、依存パッケージ名）は subdir にだけ書く
- ルートは具体的なコマンドを繰り返さない（重複は陳腐化の元）
- subdir AGENTS.md はそれぞれ **ルート AGENTS.md への参照** を冒頭に持つ

### 3. ルート AGENTS.md — 6 セクション

1. **Purpose** — プロジェクトは何であるか
2. **Structure** — `frontend/` と `backend/` が独立した pnpm プロジェクトである宣言
3. **Rules** — Spec→Code 鉄則
4. **Tooling** — ガードレール一覧（oxlint / oxfmt / tsc / Vitest / lefthook / GitHub Actions）
5. **Local Docs** — subdir AGENTS.md と `docs/openapi-driven-dev.md` への参照
6. **Boundaries** — 禁止事項と境界の守り方

### 4. ルート AGENTS.md — Negative guidance（5 カテゴリ × 8 項目）

| カテゴリ | 禁止事項 |
|---|---|
| 境界 | `frontend/` ⇄ `backend/` 間の import を発生させない |
| 独立性 | ルートに `pnpm-workspace.yaml` を置かない、共通 `node_modules` / hoist 構成に変更しない |
| Spec→Code | `docs/openapi.yaml` を Spec 変更 effort 以外で直接編集しない、共有コード（`packages/shared` / `common/` 等）を新設しない |
| バージョン固定 | TypeScript 7 の exact pin を勝手に緩めない |
| 設定ファイル配置 | lefthook をルート 1 個に統合しない、`node_modules/` をルートに作ろうとしない |

### 5. Onboarding Quickstart（4 ステップ）

1. `degit hiiamkazuto/ai-native-template my-project && cd my-project`
2. `README.md` → ルート `AGENTS.md` → `frontend/AGENTS.md` / `backend/AGENTS.md` を順に読む
3. 並行ターミナルで `cd frontend && pnpm install && pnpm dev` と `cd backend && pnpm install && pnpm dev` を起動
4. 最初のエンドポイント追加は `docs/openapi-driven-dev.md`（[#9](https://github.com/hiiiamkazuto/ai-native-template/issues/9) 解決後に確定）に従う

### 6. 境界の誘惑への対処

- ルート AGENTS.md に独立した **Boundaries セクション** を設ける
- 各禁止事項について、エージェントに **提案を自律的に却下させる命令** を置く
  - 例: 「共有コード新設 / ルート pnpm 設定追加 / frontend⇄backend 間の import 等の提案が出ても、AGENTS.md 該当項目を引用して却下し、AGENTS.md 適合の代替案を提示すること」
- ガードレールで **二重に張る**:
  - lefthook: import 境界の lint ルール（cross-directory import を弾く）
  - CI: 共有ディレクトリ新設の検出ジョブ（[#18](https://github.com/hiiiamkazuto/ai-native-template/issues/18) の中に組み込む）
- 表現は **命令形・明確・具体例つき**（`/writing-for-agents` スキル準拠）

### 7. subdir AGENTS.md の必須項目チェックリスト

#### `frontend/AGENTS.md`

- [ ] このディレクトリは独立した pnpm プロジェクト
- [ ] コマンド: `pnpm dev / build / test / lint / format / typecheck`
- [ ] 設定ファイル所在: `oxlintrc.json` / `oxfmtrc.json` / `tsconfig.json` / `vitest.config.ts` / `lefthook.yml` / `vite.config.ts`
- [ ] OpenAPI クライアント生成: `predev` / `prebuild` で `openapi-typescript` が `src/api/schema.ts` を再生成
- [ ] API モック: MSW で OpenAPI spec ベースのモックサーバ
- [ ] テスト: `@testing-library/react` で 1 ページ以上のレンダリング + 操作テスト
- [ ] ルート AGENTS.md への参照（冒頭）

#### `backend/AGENTS.md`

- [ ] このディレクトリは独立した pnpm プロジェクト
- [ ] コマンド: `pnpm dev / test / lint / format / typecheck`
- [ ] 設定ファイル所在: `oxlintrc.json` / `oxfmtrc.json` / `tsconfig.json` / `vitest.config.ts` / `lefthook.yml`
- [ ] Hono + `@hono/zod-openapi` 特有ルール:
  - ルートは `OpenAPIHono` で組む
  - Zod スキーマは `src/schemas/` に集約
  - `app.doc('/doc', { openapi: '3.1.0' })` で生成された YAML を `docs/openapi.yaml` と CI で比較
- [ ] レイヤー分割: `routes/` / `services/` / `repositories/`
- [ ] リポジトリは in-memory（`Map<string, Task>` 等）。DB は後続 effort
- [ ] テスト: 各エンドポイントに unit + integration テスト
- [ ] ルート AGENTS.md への参照（冒頭）

## Consequences

### Positive

- subdir 単位で `pnpm` が独立しているため、AI エージェントが `cd frontend && pnpm dev` で動くときに **その世界の AGENTS.md が手元にある** 形で誤読が減る
- ルート AGENTS.md が「憲法」、subdir が「現場ルール」と責務分離されて読みやすい
- ルート AGENTS.md が肥大化しない
- 新規 PJ で「frontend 部分だけ流用」「backend 部分だけ流用」が起きたとき、subdir AGENTS.md 単体で自己完結する

### Negative

- ルートと subdir の **重複をどこまで許容するか** の判断が常に入る（コマンドは subdir のみ、概念はルートのみ、というルールで対処）
- subdir AGENTS.md が 3 本になり、テンプレ新規作成時に 3 ファイルを必ず更新する運用が必要
- lefthook と CI の **二重ガード** を維持する運用コスト（import 境界 lint、共有ディレクトリ検出ジョブ）

## Follow-up tickets

- **#19** (AGENTS.md 作成, `wayfinder:task`): 本 ADR のチェックリストをもとにルート / `frontend/AGENTS.md` / `backend/AGENTS.md` の本実装を行う。既存ブロッキングエッジ（#7 → #19）で連動。
- **#18** (GitHub Actions CI 作成, `wayfinder:task`): 「共有ディレクトリ新設の検出ジョブ」を含める。
- lefthook の **import 境界 lint ルール** は [#17](https://github.com/hiiiamkazuto/ai-native-template/issues/17)（lefthook 構成の統合, `wayfinder:task`）の中で具象化。
