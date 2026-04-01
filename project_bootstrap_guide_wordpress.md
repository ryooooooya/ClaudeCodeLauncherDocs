# project_bootstrap_guide_wordpress

SWELL 子テーマ開発用のローカル環境を立ち上げ、
Claude Code のセキュリティ設定と Codex 連携をセットアップする手順。
Phase 0 から順番に実行すれば完了する。

Next.js / Astro 版との主な違い:

- PHP + CSS 環境のためビルドツールを使わない
- ハーネス（Biome / Oxlint）の恩恵が限定的（JS ファイルのみ有効）
- ローカル環境は wp-env（Docker）

各設定の背景・理由は以下を参照:

- セキュリティ（環境制御）→ `base_security_env.md` / `base_security_env_guide.md`
- Codex 連携 → `base_codex_review.md`
- WordPress / SWELL 固有の設定 → `framework_wordpress.md`

---

## 前提

- Docker Desktop がインストール済み・起動済み
- Node.js 20+
- pnpm がインストール済み
- Claude Code がインストール済み
- Codex CLI がインストール済み
- SWELL 親テーマ zip 取得済み（SWELLER'S マイページ）
- SWELL 子テーマ zip 取得済み（同上、無料）

---

## Phase 0: ディレクトリ構成と初期ファイルの作成

### 0-1. プロジェクト構成

```
project-root/
├── swell/          # SWELL 親テーマ zip を展開して配置
├── swell-child/    # SWELL 子テーマ zip を展開して配置
├── .wp-env.json
├── package.json
└── .gitignore
```

SWELL 親テーマと子テーマの zip を展開し、上記の構成で配置する。

### 0-2. .gitignore の作成

SWELL 親テーマはバージョン管理対象外にする（ライセンス上の理由・容量）。
子テーマのみを管理する。

```
# SWELL 親テーマ
/swell/

# wp-env 生成ファイル
.wp-env/
node_modules/

# 機密情報
.env*
*.pem
*.key
```

### 0-3. package.json の作成

```json
{
  "name": "your-project",
  "version": "1.0.0",
  "scripts": {
    "wp:start": "wp-env start",
    "wp:stop": "wp-env stop",
    "wp:reset": "wp-env clean all && wp-env start",
    "wp:logs": "wp-env logs",
    "lint:js": "oxlint swell-child/js"
  }
}
```

### 0-4. wp-env のインストール

```bash
pnpm add -D @wordpress/env
```

### 0-5. .wp-env.json の作成

```json
{
  "core": null,
  "phpVersion": "8.2",
  "themes": [
    "./swell",
    "./swell-child"
  ],
  "plugins": [
    "https://downloads.wordpress.org/plugin/wp-multibyte-patch.latest-stable.zip"
  ],
  "config": {
    "WP_DEBUG": true,
    "WP_DEBUG_LOG": true,
    "SCRIPT_DEBUG": true
  }
}
```

### 0-6. 子テーマの最小ファイルを作成

#### swell-child/style.css

```css
/*
Theme Name: SWELL CHILD
Template: swell
Version: 1.0.0
*/
```

#### swell-child/functions.php

```php
<?php
add_action('wp_enqueue_scripts', function() {
    $version = filemtime(get_stylesheet_directory() . '/css/custom.css');
    wp_enqueue_style(
        'swell-child-custom',
        get_stylesheet_directory_uri() . '/css/custom.css',
        [],
        $version
    );
}, 11);
```

#### swell-child/css/tokens.css

Figma MCP 経由で生成するファイルのプレースホルダー。初期は空ファイルで可。

```css
/* このファイルは Figma MCP 経由で自動生成する */
/* pnpm wp:figma-sync で更新する */
:root {
}
```

#### swell-child/css/custom.css

```css
/* SWELL カスタムスタイル */
/* SWELL のカスタムプロパティ（--color_main 等）を参照する */
```

### 0-7. ローカル環境の起動

Docker Desktop が起動していることを確認してから実行する:

```bash
pnpm wp:start
```

`✔ Done!` が表示されたら以下でアクセスできる:

- フロントエンド: `http://localhost:8888`
- 管理画面: `http://localhost:8888/wp-admin`（`admin` / `password`）

管理画面 > 外観 > テーマ から「SWELL CHILD」を有効化する。

### 0-8. git 初期化とコミット

```bash
git init
git add -A
git commit -m "init: SWELL child theme + wp-env"
```

---

## Phase 1: セキュリティ設定

### 1-1. .claudeignore

プロジェクトルートに `.claudeignore` を作成する。

```
.env*
*.pem
*.key
*.p12
*.pfx
credentials/
secrets/
.ssh/
.aws/
.config/gh/
*.token
*secret*
*credential*
.wp-env/
```

### 1-2. ~/.claude/settings.json

既存の内容がある場合はバックアップを取ってから上書きする。
WordPress 環境では `php` コマンドと `composer` を追加で許可する。

```json
{
  "enableAllProjectMcpServers": false,
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "defaultMode": "ask",
    "allow": [
      "Read",
      "Bash(pnpm *)",
      "Bash(php *)",
      "Bash(composer *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git branch *)",
      "Bash(git switch *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git stash *)",
      "Bash(git fetch *)",
      "Bash(git pull *)",
      "Bash(gh issue *)",
      "Bash(gh pr *)",
      "Bash(echo *)",
      "Bash(ls *)",
      "Bash(jq *)",
      "Bash(grep *)",
      "Bash(sort *)",
      "Bash(find *)",
      "Bash(awk *)",
      "Bash(sed *)",
      "Bash(cut *)",
      "Bash(diff *)",
      "Bash(node --version)",
      "Bash(pnpm --version)",
      "Bash(php --version)",
      "Bash(git --version)",
      "Write(/tmp/**)"
    ],
    "deny": [
      "Bash(rm *)",
      "Bash(rm -r *)",
      "Bash(rm -rf *)",
      "Bash(rm -fr *)",
      "Bash(sudo *)",
      "Bash(su *)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(nc *)",
      "Bash(ncat *)",
      "Bash(telnet *)",
      "Bash(ssh *)",
      "Bash(scp *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)",
      "Bash(osascript *)",
      "Bash(security *)",
      "Bash(pbcopy *)",
      "Bash(pbpaste *)",
      "Bash(open *)",
      "Bash(* .env*)",
      "Bash(* ~/.ssh/*)",
      "Bash(* ~/.aws/*)",
      "Bash(* ~/.config/gh/*)",
      "Bash(* ~/.git-credentials)",
      "Bash(* ~/.netrc)",
      "Bash(* ~/.npmrc)",
      "Read(./.env)",
      "Edit(./.env)",
      "Write(./.env)",
      "Read(./.env.*)",
      "Edit(./.env.*)",
      "Write(./.env.*)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(~/.git-credentials)",
      "Read(~/.config/gh/**)",
      "Read(~/.netrc)",
      "Read(~/.npmrc)",
      "Edit(~/.zshrc)",
      "Write(~/.zshrc)",
      "Edit(~/.bashrc)",
      "Write(~/.bashrc)"
    ],
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "Bash",
          "hooks": [
            { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/bash-firewall.sh" }
          ]
        },
        {
          "matcher": "Read|Edit|MultiEdit|Write",
          "hooks": [
            { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/protect-files.py" }
          ]
        }
      ]
    }
  }
}
```

構文チェック:

```bash
jq . ~/.claude/settings.json > /dev/null && echo "OK" || echo "JSON構文エラー"
```

### 1-3. .claude/settings.json（プロジェクトレベル）

WordPress 環境では PHP / CSS ファイルに対応したフックに差し替える。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/post-wp-lint.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/protect-config.sh"
          }
        ]
      }
    ]
  }
}
```

### 1-4. フックスクリプトの作成

`.claude/hooks/` ディレクトリを作成し、以下の4ファイルを配置する。

#### .claude/hooks/bash-firewall.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

if echo "$COMMAND" | grep -qE '(curl|wget).*\|.*(sh|bash|zsh)'; then
  echo "Blocked: pipe-to-shell is prohibited." >&2
  exit 2
fi

if echo "$COMMAND" | grep -qE 'rm\s+-[a-z]*[rf]'; then
  echo "Blocked: rm with recursive/force flags is prohibited." >&2
  exit 2
fi

if echo "$COMMAND" | grep -qP 'git\s+push\s+(origin\s+)?(main|master)\b'; then
  echo "Blocked: direct push to main/master is prohibited." >&2
  exit 2
fi

if echo "$COMMAND" | grep -qE '^\s*sudo\s+'; then
  echo "Blocked: sudo is prohibited inside Claude Code sessions." >&2
  exit 2
fi

exit 0
```

#### .claude/hooks/protect-files.py

```python
#!/usr/bin/env python3
import sys, json
from pathlib import Path

SENSITIVE_PATTERNS = {
    '.env', '.pem', '.key', '.p12', '.pfx',
    '.credential', '.token', 'credentials.json',
    'id_rsa', 'id_ed25519', 'id_ecdsa'
}

def is_sensitive(path: str) -> bool:
    p = Path(path)
    name = p.name.lower()
    for pattern in SENSITIVE_PATTERNS:
        if name == pattern or name.endswith(pattern):
            return True
    if name.startswith('.env'):
        return True
    if any(kw in name for kw in ('secret', 'credential', 'private_key')):
        return True
    return False

def is_parent_theme(path: str) -> bool:
    # SWELL 親テーマへの書き込みをブロック
    p = Path(path)
    parts = p.parts
    if 'swell' in parts and 'swell-child' not in parts:
        return True
    return False

def main():
    try:
        data = json.load(sys.stdin)
    except json.JSONDecodeError:
        sys.exit(0)
    tool_name = data.get('tool_name', '')
    file_path = data.get('tool_input', {}).get('path') \
             or data.get('tool_input', {}).get('file_path', '')
    if not file_path:
        sys.exit(0)
    if is_sensitive(file_path):
        print(
            f"SECURITY: Access to '{file_path}' is blocked.\n"
            "Credential files and .env must not be read or modified by Claude.",
            file=sys.stderr
        )
        sys.exit(2)
    if tool_name in ('Write', 'Edit', 'MultiEdit') and is_parent_theme(file_path):
        print(
            f"BLOCKED: '{file_path}' は SWELL 親テーマのファイルです。\n"
            "親テーマは編集禁止。カスタマイズは swell-child/ で行ってください。",
            file=sys.stderr
        )
        sys.exit(2)
    sys.exit(0)

if __name__ == '__main__':
    main()
```

#### .claude/hooks/post-wp-lint.sh

PHP / CSS / JS ファイルの編集後に対応するリンターを自動実行する。

```bash
#!/usr/bin/env bash
set -euo pipefail

input="$(cat)"
file="$(jq -r '.tool_input.file_path // .tool_input.path // empty' <<< "$input")"

case "$file" in
  *.php)
    if command -v phpcs &>/dev/null; then
      diag="$(phpcs --standard=WordPress "$file" 2>&1 | head -20 || true)"
      if [ -n "$diag" ]; then
        jq -Rn --arg msg "$diag" '{
          hookSpecificOutput: {
            hookEventName: "PostToolUse",
            additionalContext: $msg
          }
        }'
      fi
    fi
    ;;
  *.js)
    npx oxlint --fix "$file" >/dev/null 2>&1 || true
    diag="$(npx oxlint "$file" 2>&1 | head -20)"
    if [ -n "$diag" ]; then
      jq -Rn --arg msg "$diag" '{
        hookSpecificOutput: {
          hookEventName: "PostToolUse",
          additionalContext: $msg
        }
      }'
    fi
    ;;
  *.css)
    # CSS はリンターなし。構文エラーがあれば手動で確認する
    ;;
  *)
    exit 0
    ;;
esac
```

#### .claude/hooks/protect-config.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

input="$(cat)"
file="$(jq -r '.tool_input.file_path // .tool_input.path // empty' <<< "$input")"

protected=".wp-env.json package.json lefthook.yml"

for p in $protected; do
  case "$file" in
    *$p*)
      echo "BLOCKED: $file は保護された設定ファイルです。" >&2
      exit 2
      ;;
  esac
done
```

実行権限を付与する:

```bash
chmod +x .claude/hooks/bash-firewall.sh
chmod +x .claude/hooks/protect-files.py
chmod +x .claude/hooks/post-wp-lint.sh
chmod +x .claude/hooks/protect-config.sh
```

---

## Phase 2: CLAUDE.md

以下の内容をプロジェクトルートの `CLAUDE.md` として配置する。

````markdown
# CLAUDE.md

## プロジェクト概要

WordPress + SWELL 子テーマ

## コマンド

```bash
pnpm wp:start     # ローカル WordPress 起動（http://localhost:8888）
pnpm wp:stop      # ローカル WordPress 停止
pnpm wp:reset     # 環境リセット（データも消える）
pnpm lint:js      # JS リント（Oxlint）
```

管理画面: http://localhost:8888/wp-admin（admin / password）

## ルール

- SWELL 親テーマ（swell/）のファイルは編集禁止。protect-files.py がブロックする
- カスタマイズはすべて swell-child/ で行う
- CSS カスタムプロパティは SWELL のアンダースコア命名に従う（`--color_main` 形式）
- 子テーマで色をハードコードしない。必ず SWELL の CSS 変数（`var(--color_main)` 等）を使う
- Figma のデザイントークンを変更したときは Figma MCP 経由で `css/tokens.css` を再生成する
- functions.php にすべての処理を書かない。機能ごとにファイルを分けて require_once で読み込む
- WordPress の nonce を使わないフォーム送信を書かない
- PHP のバージョンは .wp-env.json の phpVersion に合わせる（現在: 8.2）

## アーキテクチャ

```
swell-child/
├── style.css           # テーマ定義（必須）
├── functions.php       # フック・読み込みのみ記述
├── css/
│   ├── tokens.css      # Figma MCP から生成したデザイントークン
│   └── custom.css      # カスタムスタイル
└── js/
    └── custom.js       # カスタムスクリプト（必要な場合）
```

## SWELL CSS 変数リファレンス

```css
--color_main        /* メインカラー */
--color_main_thin   /* メインカラー（薄め） */
--color_text        /* テキストカラー */
--color_link        /* リンクカラー */
--font_size_base    /* 基本フォントサイズ */
--content_width     /* コンテンツ幅 */
```

## セキュアコーディングルール（PHP）

### strict ルール（必ず守ること）

- すべての外部入力（$_GET, $_POST, $_REQUEST）を sanitize_* 関数でサニタイズする
- すべての出力を esc_html() / esc_attr() / esc_url() / wp_kses() でエスケープする
- フォームには必ず nonce を使う（wp_nonce_field() / check_admin_referer()）
- DB クエリは $wpdb->prepare() を使う。文字列結合でクエリを組み立てない
- capability チェック（current_user_can()）を認証が必要な処理に必ず含める

### 禁止パターン

```php
// NG: サニタイズなし
$value = $_POST['value'];

// NG: エスケープなし
echo $_GET['name'];

// NG: nonce なしのフォーム
if (isset($_POST['submit'])) { ... }

// NG: プリペアなし SQL
$wpdb->query("SELECT * FROM $wpdb->posts WHERE ID = " . $_GET['id']);
```

## Figma MCP 連携（デザイントークン更新）

Figma Variables が更新されたら以下の手順で tokens.css を再生成する:

```
Figma ファイルの SWELL Tokens コレクションから get_variable_defs でトークンを取得し、
:root { --color_main: ...; } の形式で swell-child/css/tokens.css に出力してください。
変数名はアンダースコア命名（--color_main 形式）を維持してください。
```

## 実装計画レビュー（Codex 連携）

実装計画をユーザーに提示する前に、必ず Codex でレビューを行う。

```bash
codex exec -m gpt-5.3-codex "このプランをレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {plan_full_path} (ref: {CLAUDE.md full_path})"
```

修正後の再レビュー:

```bash
codex exec resume --last -m gpt-5.3-codex "プランを更新したからレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {plan_full_path} (ref: {CLAUDE.md full_path})"
```

## コードレビュー（Codex 連携）

実装完了後、ユーザーに報告する前に Codex でコードレビューを行う。

```bash
codex exec -m gpt-5.3-codex "このコードをレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {commit_hash} (ref: {CLAUDE.md full_path})"
```

修正後の再レビュー:

```bash
codex exec resume --last -m gpt-5.3-codex "修正したから再レビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {commit_hash} (ref: {CLAUDE.md full_path})"
```

判断基準: セキュリティ・ロジックの問題は必ず修正。スタイル・命名等は無視。

## 完了の定義

- ローカル環境（http://localhost:8888）で表示が崩れていない
- `pnpm lint:js` がエラーなく完了する
- SWELL 親テーマ（swell/）のファイルが変更されていない
````

---

## Phase 3: 動作確認

### wp-env の確認

- [ ] `pnpm wp:start` で `http://localhost:8888` が表示される
- [ ] 管理画面に SWELL と SWELL CHILD が存在する
- [ ] SWELL CHILD が有効化されている
- [ ] `swell-child/css/custom.css` の変更がブラウザに反映される

### セキュリティの確認

- [ ] `.claudeignore` が存在し、機密ファイルパターンが記載されている
- [ ] `~/.claude/settings.json` の JSON 構文が正しい（`jq .` で確認）
- [ ] `disableBypassPermissionsMode` が `"disable"` になっている
- [ ] `.claude/hooks/` の4ファイルが存在し、実行権限がある
- [ ] `swell/` 配下のファイルを編集しようとすると `protect-files.py` がブロックする

### MCP サーバーの確認

- [ ] `~/.claude.json` に不要な MCP サーバーが登録されていない

### CLAUDE.md の確認

- [ ] プロジェクトルートに `CLAUDE.md` が配置されている
- [ ] PHP セキュアコーディングルールが含まれている
- [ ] SWELL CSS 変数リファレンスが含まれている
- [ ] Codex 連携のセクションが含まれている

---

## 運用ルール

- SWELL 親テーマのアップデートは手動で行う（zip を差し替えて `pnpm wp:reset`）
- Figma のデザイントークンが変わったら必ず Figma MCP 経由で `tokens.css` を再生成する
- `tokens.css` は自動生成ファイルのため直接編集しない
- お金が動くサービスの認証情報は AI がアクセスできる環境から完全に隔離する
