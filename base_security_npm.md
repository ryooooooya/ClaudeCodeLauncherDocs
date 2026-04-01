# base_security_npm

このファイルに書かれたルールは、npm パッケージの追加・更新・監査に関わるすべての場面で適用すること。
`base_security_code.md` セクション14「依存関係」を補完する詳細手順。

---

## 適用レベルの定義

- strict: 省略できない。違反した場合は作業を中断してユーザーに報告する
- warning: 警告を添えた上で代替案を提示する
- advisory: ベストプラクティスとして言及する

---

## 1. パッケージ追加時の確認手順（strict）

`pnpm add` / `pnpm install` で新しいパッケージを追加する前に、以下を順番に実行する。

### 1-1. 実在確認

npm レジストリにパッケージが存在し、意図したものであることを確認する。
AI が生成したパッケージ名は存在しない場合がある（スクワッティング攻撃のリスク）。

```bash
pnpm info <package-name> --json | jq '{name, version, description, maintainers, homepage}'
```

週間ダウンロード数・メンテナ数・最終更新日が極端に少ない／古い場合はユーザーに報告する。

### 1-2. postinstall スクリプトの確認

```bash
pnpm info <package-name> --json | jq '.scripts // {}' | grep -iE 'install|prepare|build'
```

`postinstall`・`preinstall`・`install` スクリプトが含まれている場合は、その内容をユーザーに提示して承認を得る。

### 1-3. 依存ツリーの確認

```bash
pnpm why <package-name>
```

追加されるパッケージが持つ transitive dependency の数が多い場合（目安: 10以上）はユーザーに報告する。

### 1-4. 追加実行と差分提示

```bash
pnpm add <package-name>
git diff pnpm-lock.yaml | head -100
```

lockfile の差分を提示し、意図しないパッケージが含まれていないことを確認する。

---

## 2. パッケージ更新時の確認手順（strict）

`pnpm update` でパッケージを更新する前に、以下を実行する。

### 2-1. 更新前の依存スナップショット

```bash
pnpm list --depth 0 > /tmp/deps-before.txt
```

### 2-2. 更新実行と差分確認

```bash
pnpm update <package-name>
pnpm list --depth 0 > /tmp/deps-after.txt
diff /tmp/deps-before.txt /tmp/deps-after.txt
```

### 2-3. transitive dependency の増減確認

更新によって新しい transitive dependency が追加された場合、そのパッケージに対してセクション1の手順（実在確認・postinstall 確認）を実行する。

これは axios (2026-03) のような攻撃パターンへの対策。メンテナアカウントが乗っ取られ、正規パッケージのバージョン更新に悪意ある transitive dependency が注入されたケースでは、更新時の差分確認でのみ検出できた。

---

## 3. lockfile の管理方針（strict）

### 3-1. lockfile は必ずコミットする

`pnpm-lock.yaml`（または `package-lock.json` / `yarn.lock`）を `.gitignore` に入れてはならない。

### 3-2. CI では frozen install を使う

```bash
# CI 環境
pnpm install --frozen-lockfile
```

lockfile と `package.json` の不一致があればインストールを失敗させる。

### 3-3. バージョンレンジの方針（advisory）

`package.json` でのバージョン指定は `^`（caret）をデフォルトとする（pnpm のデフォルト挙動）。
lockfile がバージョンをピン留めするため、`pnpm install` だけで意図しない更新は起きない。

ただし以下のケースでは exact pinning（`--save-exact` / `-E`）を検討する:

- 認証・暗号化に関わるパッケージ（例: jsonwebtoken, argon2）
- postinstall スクリプトを持つパッケージ
- 過去にサプライチェーン攻撃を受けたパッケージ

```bash
pnpm add <package-name> --save-exact
```

---

## 4. 防御的な .npmrc 設定（advisory）

プロジェクトルートの `.npmrc` に以下を検討する。

```ini
# postinstall スクリプトをデフォルトで無効化
ignore-scripts=true
```

`ignore-scripts=true` を設定した場合、ビルドが必要なネイティブモジュール（例: sharp, argon2）は個別に許可する:

```bash
pnpm rebuild <package-name>
```

このトレードオフ（利便性 vs 安全性）はプロジェクト開始時にユーザーと相談して決める。

---

## 5. 監査と検出（warning）

### 5-1. pnpm audit

```bash
pnpm audit
```

`high` 以上の脆弱性がある場合はユーザーに報告し、対処方針を相談する。

### 5-2. npm provenance の確認（advisory）

npm provenance（SLSA）はパッケージが正規の CI/CD パイプラインからビルド・公開されたことを証明する署名。メンテナアカウントが乗っ取られた場合、攻撃者のローカル環境から公開されるため provenance がつかない。

```bash
npm view <package-name>@<version> --json | jq '.dist.attestations'
```

provenance が存在するバージョンと存在しないバージョンが混在している場合は、存在しないバージョンを避ける。

### 5-3. 推奨ツール（advisory）

| ツール | 用途 |
|---|---|
| `pnpm audit` | 既知の脆弱性検出（標準搭載） |
| Socket.dev | 新規・更新パッケージのリアルタイムリスク検出 |
| Snyk | 脆弱性検出 + 修正 PR 自動生成 |
| npm provenance | パッケージの出自証明 |

---

## 6. CI での依存関係チェック（advisory）

GitHub Actions に組み込む例:

```yaml
# .github/workflows/dependency-check.yml
name: Dependency Check
on:
  pull_request:
    paths:
      - 'package.json'
      - 'pnpm-lock.yaml'

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4
      - uses: pnpm/action-setup@a7487c7e89a18df4991f7f222e4898a00d66ddda  # v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm audit --audit-level=high
```

`package.json` または lockfile に変更がある PR でのみ実行される。

---

## 7. 緊急対応: インシデント情報を受けたとき

ユーザーが「○○パッケージがやられたらしい」「このCVEは影響ある？」のようにサプライチェーンインシデントの情報を伝えてきた場合、以下の手順で影響を確認する。

### 7-1. 影響範囲の特定

```bash
# lockfile でパッケージとバージョンを検索
grep -n "<package-name>" pnpm-lock.yaml

# 依存ツリー上の位置を確認
pnpm why <package-name>

# node_modules に実際にインストールされているバージョンを確認
ls node_modules/<package-name>/package.json 2>/dev/null && \
  jq '{name, version, scripts}' node_modules/<package-name>/package.json

# transitive dependency として入り込んでいないか再帰的に検索
pnpm list --depth Infinity | grep "<package-name>"
```

### 7-2. 影響判定

上記の結果をユーザーに提示し、以下の3パターンのいずれかを報告する。

a) 影響なし — 該当パッケージが依存ツリーに存在しない
b) バージョンが安全 — 該当パッケージは存在するが、侵害されたバージョンとは異なる
c) 侵害の可能性あり — 侵害されたバージョンが lockfile または node_modules に存在する

### 7-3. 侵害が確認された場合の対処

c) の場合、以下をユーザーに提案する。すべての実行にはユーザーの承認を得ること。

```bash
# 1. 安全なバージョンにダウングレード
pnpm add <package-name>@<safe-version> --save-exact

# 2. 悪意あるパッケージが transitive dependency の場合は削除
rm -rf node_modules/<malicious-package>
pnpm install --frozen-lockfile

# 3. lockfile の差分を確認
git diff pnpm-lock.yaml
```

加えて、以下の運用アクションをユーザーに伝える（Claude Code の管轄外だが、必ず言及すること）:

- 侵害バージョンを `pnpm install` した環境のクレデンシャルをすべてローテーションする（環境変数、SSH鍵、APIトークン、クラウド認証情報）
- CI/CD パイプラインのログを確認し、侵害バージョンがインストールされたビルドを特定する
- 侵害パッケージが postinstall を持っていた場合、インストールした全マシンを侵害済みとして扱う
- インシデントの公式アドバイザリ（GitHub Security Advisory、CVE）が公開されたらリンクを記録する

---

## 8. サンドボックスとの関係

`base_security_env.md` で設定するサンドボックスは、npm の postinstall スクリプトから生まれる子プロセスにも制限を継承する。

- ファイルシステム隔離: postinstall が `~/.bashrc` や `~/.ssh/` を改ざんしようとしてもブロックされる
- ネットワーク隔離: postinstall から C2 サーバーへの通信がブロックされる

ただし以下の環境ではサンドボックスが効かない:

- WSL1（bubblewrap 非対応）
- Docker を `excludedCommands` に設定している環境での Docker 内 `npm install`
- サンドボックスを有効化していない環境

サンドボックスがない環境では、このファイルのセクション1〜5の手順がより重要になる。

---

## 参考事例

| 事例 | 年月 | 手口 | 教訓 |
|---|---|---|---|
| axios | 2026-03 | メンテナアカウント乗っ取り。正規パッケージに悪意ある transitive dependency を注入。postinstall で RAT をドロップ | 更新時の transitive dependency 差分チェックが重要。provenance のないバージョンを避ける |
| event-stream | 2018-11 | メンテナ権限を善意の第三者に譲渡 → 攻撃者が flatmap-stream を注入。特定の暗号通貨ウォレットを狙った攻撃 | メンテナの変更は信頼性のリセットと考える |
| ua-parser-js | 2021-10 | メンテナアカウント乗っ取り。暗号通貨マイナーとパスワード窃取ツールを注入 | 週間ダウンロード数が多いパッケージも標的になる |
| colors / faker | 2022-01 | メンテナ自身が意図的に破壊コードを挿入（抗議行為） | メンテナの意図に依存しない防御が必要 |
