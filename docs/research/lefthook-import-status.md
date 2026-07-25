# lefthook `import` 機能の現状調査

> 調査日: 2026-07-26
> 対象: evilmartians/lefthook v2.1.10

---

## 0. 前提: lefthook に `import` ディレクティブは存在しない

lefthook には **`import` というディレクティブは存在しない**。
ルート設定から別ファイルの設定を束ねる機能は以下の 2 つが相当する:

| 機能 | 用途 | ソース |
|------|------|--------|
| `extends` | ローカルファイル（絶対パス・相対パス・glob）から設定をマージ | [lefthook.dev/configuration/extends](https://lefthook.dev/configuration/extends/) |
| `remotes` | Git リモートリポジトリから設定をダウンロードしてマージ | [lefthook.dev/configuration/remotes](https://lefthook.dev/configuration/remotes/) |

---

## 1. 最新安定版と extends/remotes のシンタックス

### 最新安定版

**v2.1.10** (2026-07-08 リリース)

- ソース: [GitHub Releases](https://github.com/evilmartians/lefthook/releases/tag/v2.1.10)

### `extends` シンタックス (v2 系)

```yaml
# lefthook.yml (ルート)
extends:
  - /absolute/path/to/frontend-lefthook.yml
  - ../relative/path.yml
  - frontend/lefthook.yml
  - projects/*/specific-lefthook-config.yml  # glob サポート (v1.10.0 以降)
```

- ソース: [lefthook.dev/configuration/extends](https://lefthook.dev/configuration/extends/)
- glob サポートは Discussion #852 で提案され、PR #853 で実装
  - ソース: [Discussion #852](https://github.com/evilmartians/lefthook/discussions/852)

### `remotes` シンタックス

```yaml
# lefthook.yml
remotes:
  - git_url: git@github.com:org/shared-lefthook-config
    ref: v1.0.0
    configs:
      - examples/ruby-linter.yml
```

- ソース: [lefthook.dev/configuration/remotes](https://lefthook.dev/configuration/remotes/)

### v1 系との差分

公式ドキュメントに v1 → v2 の移行ガイドは見つからなかった。v2 系では `jobs` 構文が導入され (v1.10.0)、コマンド/スクリプトを統一的に扱えるようになった。

- ソース: [lefthook.dev/configuration/jobs](https://lefthook.dev/configuration/jobs/)

---

## 2. extends で取り込んだ設定の pre-commit / pre-push 発火挙動

### マージ順序

設定は以下の優先順位でマージされる:

1. `lefthook.yml` (メイン設定)
2. `extends` (extends で指定された設定)
3. `remotes` (remotes で指定された設定)
4. `lefthook-local.yml` (ローカル設定)

後勝ち (later overrides earlier)。

- ソース: [lefthook.dev/configuration/extends](https://lefthook.dev/configuration/extends/)

### 並列・順序制御

- デフォルト: **逐次実行** (`parallel: false`)
- `parallel: true` を指定すると全ジョブが並列実行される
- `piped: true` を指定するとジョブが順番に実行され、1 つ失敗すると以降をスキップ
- `parallel` と `piped` は同時に指定できない

```yaml
pre-commit:
  parallel: true  # 全ジョブ並列
  jobs:
    - name: lint
      run: yarn lint
    - name: test
      run: yarn test
```

- ソース: [lefthook.dev/configuration/parallel](https://lefthook.dev/configuration/parallel/)
- ソース: [lefthook.dev/configuration/piped](https://lefthook.dev/configuration/piped/)

### extends 先のジョブ発火

extends でマージされたジョブは、メイン設定のジョブと同じタイミングで発火する。**名前付きジョブ (`name` 指定) は設定をマージ**し、**名前なしジョブは定義順で追加**される。

- ソース: [lefthook.dev/configuration/jobs](https://lefthook.dev/configuration/jobs/)

---

## 3. glob フィルタのパス評価基点

**重要: `glob` は常に Git リポジトリルートからの相対パスで評価される。`root` オプションは `glob` に影響しない。**

```yaml
pre-commit:
  commands:
    lint:
      root: "client/"        # CWD を client/ に変更
      glob: "*.{js,ts}"      # これは repo-root からの相対で評価される
      run: yarn eslint --fix {staged_files}
```

公式ドキュメントの `root` のページに明記:

> "Globs are always calculated from the actual root of the git repo — `root` does not affect glob matching."

- ソース: [lefthook.dev/configuration/root](https://lefthook.dev/configuration/root/)
- ソース: [lefthook.dev/configuration/glob](https://lefthook.dev/configuration/glob/)

### extends 先のパス解決の問題

**既知のバグ**: `extends` の再帰的解決時、相対パスがメイン設定ファイルからの相対として解決される (extends 先ファイルからの相対ではない)。

Issue #1258 で報告されており、修正 PR は提出されているが未マージ:

```go
// 現在の挙動: root は常にメイン設定ファイルのディレクトリ
extendRecursive(extent, filesystem, root, extent.Strings("extends"), visited)

// 期待される挙動: extends 先ファイルのディレクトリを基点にするべき
extentDir := filepath.Dir(path)
extendRecursive(extent, filesystem, extentDir, extent.Strings("extends"), visited)
```

- ソース: [Issue #1258](https://github.com/evilmartians/lefthook/issues/1258)

**実運用への影響**: `frontend/lefthook.yml` から `extends: [../shared.yml]` のような相対パスを使う場合、再帰的な extends が発生するとパス解決が壊れる可能性がある。

---

## 4. parallel / piped / skip 等の制御ディレクティブが extends 先で機能するか

**機能する。** extends でマージされた設定は、メイン設定と区別なく扱われる。

- `parallel`: hook レベルで指定可能。extends 先でも有効
- `piped`: hook レベルで指定可能。extends 先でも有効
- `skip`: hook レベル・ジョブレベル両方で指定可能。`ref: main`, `merge`, `rebase`, `run: ...` 等の条件付き skip も可能
- `root`: ジョブレベルで指定可能。**group レベルでは未対応** (Issue #1469 で feature request 中)
- `glob`, `exclude`: ジョブレベルで指定可能。group の場合は全ネストジョブに適用される

```yaml
# extends 先の例 (frontend/lefthook.yml)
pre-commit:
  skip:
    - ref: main
  parallel: true
  jobs:
    - name: lint
      root: frontend/
      glob: "*.{ts,tsx}"
      run: yarn lint {staged_files}
```

- ソース: [lefthook.dev/configuration/skip](https://lefthook.dev/configuration/skip/)
- ソース: [lefthook.dev/configuration/jobs](https://lefthook.dev/configuration/jobs/)

### group レベルの root について

Issue #1469 (2026-07-25 作成) で `group` レベルでの `root` サポートが feature request として提起されている。現状では個別のジョブごとに `root` を指定する必要がある。

- ソース: [Issue #1469](https://github.com/evilmartians/lefthook/issues/1469)

---

## 5. per-directory 独立 vs ルートからの extends 束ね: 挙動の差

### 公式ドキュメントの記述

**公式ドキュメントに per-directory とルート束ねの明示的な比較は存在しない。**

ただし、以下の事実が確認できる:

1. lefthook は **Git リポジトリルート** で動作する前提のツール。`.lefthook/` (キャッシュ・スクリプト格納) は Git ルートに作成される
   - ソース: [lefthook.dev/configuration/source_dir](https://lefthook.dev/configuration/source_dir/)

2. `lefthook install` は `.git/hooks/` にシェルスクリプトを生成し、その中でlefthook バイナリへのパスをベイクする
   - ソース: [Issue #1398](https://github.com/evilmartians/lefthook/issues/1398)

3. モノレポでの推奨パターンは、ルートに 1 つの `lefthook.yml` を置き、`root` オプションでサブディレクトリを指定する方式
   - ソース: [Discussion #852](https://github.com/evilmartians/lefthook/discussions/852)

### per-directory 独立運用の制約

- サブディレクトリに独立した `lefthook.yml` を置き、そのディレクトリで `lefthook install` を実行することは技術的に可能だが、lefthook は Git ルートを基点に動作するため、**意図した通りに動作しない可能性がある**
- `lefthook install` が `.git/hooks/` に絶対パスをベイクするため、worktree 環境では壊れる (Issue #1398)

### Discussion で確認されたモノレポ運用パターン

Discussion #852 で、モノレポでの現実的な運用パターンが共有されている:

```yaml
# ルート lefthook.yml - YAML エイリアスで DRY 化
__shared__:
  commands:
    tsc: &tsc
      run: pnpm tsc --noEmit
    lint: &lint
      glob: "*.{js,ts}"
      run: pnpm eslint {staged_files}

pre-commit:
  parallel: true
  commands:
    "@pkg/database:tsc": { root: packages/database, <<: *tsc }
    "@pkg/database:lint": { root: packages/database, <<: *lint }
    "@app/frontend:tsc": { root: apps/frontend, <<: *tsc }
    "@app/frontend:lint": { root: apps/frontend, <<: *lint }
```

- ソース: [Discussion #852](https://github.com/evilmartians/lefthook/discussions/852)

---

## 6. キャッシュ機構の per-directory 競合

### source_dir (デフォルト: `.lefthook/`)

lefthook のキャッシュ・スクリプト格納ディレクトリは `source_dir` で変更可能 (デフォルト: `.lefthook/`)。これは Git ルートに作成される。

- ソース: [lefthook.dev/configuration/source_dir](https://lefthook.dev/configuration/source_dir/)

### 競合の可能性

- ルートとサブディレクトリで別々の lefthook 設定を `lefthook install` した場合、**両方が `.git/hooks/` の同じシェルスクリプトを上書き**する可能性がある
- Issue #1398 では、`lefthook install` が hooks パスを上書きする問題が報告されている
  - ソース: [Issue #1398](https://github.com/evilmartians/lefthook/issues/1398)

**結論**: per-directory で独立して `lefthook install` を実行すると、hooks の競合が発生するリスクがある。**ルートに 1 つの設定を置く方式が推奨される。**

---

## 7. 最小動作例: extends を使ったルート束ね構成

```
repo/
├── lefthook.yml              # ルート設定
├── frontend/
│   └── lefthook.yml          # フロントエンド設定
└── backend/
    └── lefthook.yml          # バックエンド設定
```

### ルート `lefthook.yml`

```yaml
extends:
  - frontend/lefthook.yml
  - backend/lefthook.yml

# または glob (v1.10.0+)
extends:
  - "*/lefthook.yml"
```

### `frontend/lefthook.yml`

```yaml
pre-commit:
  jobs:
    - name: frontend-lint
      root: "frontend/"
      glob: "*.{ts,tsx}"
      run: yarn eslint {staged_files}
```

### `backend/lefthook.yml`

```yaml
pre-commit:
  jobs:
    - name: backend-lint
      root: "backend/"
      glob: "*.py"
      run: ruff check {staged_files}
```

**注意点**:
- `glob` は Git ルートからの相対で評価される (`root` の影響を受けない)
- `root` はコマンド実行時の CWD を変更するだけ
- extends 先の相対パス解決には Issue #1258 のバグが残存中

---

## まとめ: 未確認事項と判断

| 調査項目 | 結論 |
|----------|------|
| `import` ディレクティブの存在 | **存在しない**。`extends` / `remotes` が相当機能 |
| 最新安定版 | v2.1.10 (2026-07-08) |
| 発火挙動 | extends 先も同一 hook で発火。parallel/piped で制御可能 |
| glob のパス基点 | **常に Git ルート**。`root` は影響しない |
| 制御ディレクティブの継承 | extends 先でも parallel/piped/skip は機能する |
| per-directory vs ルート束ね | 公式比較なし。ルート束ね推奨 |
| キャッシュ競合 | per-directory 独立運用は hooks 上書きのリスクあり |
| extends の相対パス解決 | **既知バグあり** (Issue #1258) |
| group レベル root | **未対応** (Issue #1469 で feature request 中) |
