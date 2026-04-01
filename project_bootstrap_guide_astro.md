# project_bootstrap_guide_astro

空のリポジトリから Astro プロジェクトを立ち上げ、
Claude Code のハーネス・セキュリティ・a11y・Codex 連携まで一気にセットアップする手順。
Phase 0 から順番に実行すれば完了する。

CMS はプロジェクトごとに異なるため、Phase 0 の末尾でいずれかを選択する。
選択後、該当セクション以外の CMS 手順を削除してから進めること。

各設定の背景・理由は以下を参照:

- ハーネスの設計思想 → `base_harness.md`
- セキュリティ（環境制御）→ `base_security_env.md` / `base_security_env_guide.md`
- セキュリティ（生成コード）→ `base_security_code.md` / `base_security_code_guide.md`
- アクセシビリティ → `base_a11y.md`
- Codex 連携 → `base_codex_review.md`

---

## 前提

- Node.js 20+
- pnpm がインストール済み
- Claude Code がインストール済み
- Codex CLI がインストール済み
- 空のディレクトリまたは git init 済みのリポジトリ

---

## Phase 0: Astro プロジェクト初期化

### 0-1. プロジェクト作成

```bash
pnpm create astro@latest . \
  --template minimal \
  --typescript strict \
  --install \
  --no-git
```

### 0-2. Tailwind CSS の追加

```bash
pnpm astro add tailwind
```

対話プロンプトで `Yes` を選択する。`tailwind.config.mjs` と `src/styles/global.css` が生成される。

### 0-3. tsconfig.json の確認

`noUncheckedIndexedAccess` を手動で追加する。

```json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "noUncheckedIndexedAccess": true
  }
}
```

### 0-4. CMS の選択とセットアップ

以下から1つ選択し、該当セクションの手順を実行する。
選択後、このセクション全体を選択した CMS の内容に置き換えること。

選択肢:
- A: ローカル Markdown（CMS なし）
- B: microCMS
- C: Sanity
- D: Storyblok
- E: Contentful

#### A: ローカル Markdown（CMS なし）

追加インストール不要。`content.config.ts` を作成する:

```typescript
// src/content.config.ts
import { defineCollection, z } from 'astro:content'
import { glob } from 'astro/loaders'

const blog = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    description: z.string(),
    publishDate: z.string().transform((s) => new Date(s)),
    draft: z.boolean().default(false),
    tags: z.array(z.string()).optional(),
  }),
})

export const collections = { blog }
```

#### B: microCMS

```bash
pnpm add microcms-js-sdk
```

```typescript
// src/lib/microcms.ts
import { createClient } from 'microcms-js-sdk'

export const client = createClient({
  serviceDomain: import.meta.env.MICROCMS_SERVICE_DOMAIN,
  apiKey: import.meta.env.MICROCMS_API_KEY,
})
```

`.env.local` に追加:

```
MICROCMS_SERVICE_DOMAIN=your-service-domain
MICROCMS_API_KEY=your-api-key
```

#### C: Sanity

```bash
pnpm astro add sanity
```

`astro.config.mjs` に追加:

```typescript
import { defineConfig } from 'astro/config'
import tailwind from '@astrojs/tailwind'
import sanity from '@sanity/astro'

export default defineConfig({
  integrations: [
    tailwind(),
    sanity({
      projectId: import.meta.env.SANITY_PROJECT_ID,
      dataset: import.meta.env.SANITY_DATASET,
      useCdn: false,
    }),
  ],
})
```

`.env.local` に追加:

```
SANITY_PROJECT_ID=your-project-id
SANITY_DATASET=production
```

#### D / E: Storyblok / Contentful

公式ドキュメントに従ってセットアップする:

- Storyblok: https://docs.astro.build/en/guides/cms/storyblok/
- Contentful: https://docs.astro.build/en/guides/cms/contentful/

### 0-5. astro sync の実行

Content Collections のスキーマを定義したら型を生成する:

```bash
pnpm astro sync
```

### 0-6. git 初期化とコミット

```bash
git init
git add -A
git commit -m "init: Astro + Tailwind + {選択した CMS}"
```

---

## Phase 1: ハーネス

### 1-1. ツールのインストール

```bash
pnpm add -D @biomejs/biome oxlint lefthook
```

### 1-2. Biome 設定

`biome.json` をプロジェクトルートに作成する。

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  }
}
```

### 1-3. Oxlint 設定

`.oxlintrc.json` をプロジェクトルートに作成する。

```json
{
  "rules": {
    "no-explicit-any": "error",
    "no-unused-vars": "error"
  }
}
```

### 1-4. package.json にスクリプトを追加

```json
{
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "sync": "astro sync",
    "lint": "oxlint src --extension .ts,.tsx,.astro",
    "format": "biome format --write src",
    "typecheck": "tsc --noEmit"
  }
}
```

### 1-5. Lefthook でプリコミットフックを設定

`lefthook.yml` をプロジェクトルートに作成する。

```yaml
pre-commit:
  parallel: true
  commands:
    typecheck:
      run: pnpm typecheck
    lint:
      run: pnpm lint
    format-check:
      run: pnpm biome format src --diagnostic-level=error
```

```bash
pnpm lefthook install
```

---

## Phase 2: セキュリティ設定

### 2-1. .claudeignore

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
```

### 2-2. ~/.claude/settings.json

既存の内容がある場合はバックアップを取ってから上書きすること。

```json
{
  "enableAllProjectMcpServers": false,
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "defaultMode": "ask",
    "allow": [
      "Read",
      "Bash(pnpm *)",
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
      "Bash(python3 --version)",
      "Bash(node --version)",
      "Bash(pnpm --version)",
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
      "Read(./**/.env)",
      "Read(./**/.env.*)",
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

### 2-3. .claude/settings.json（プロジェクトレベル）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/post-ts-lint.sh"
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

### 2-4. フックスクリプトの作成

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
  echo "Blocked: rm with recursive/force flags is prohibited. Use trash or mv instead." >&2
  exit 2
fi

if echo "$COMMAND" | grep -qP 'git\s+push\s+(origin\s+)?(main|master)\b'; then
  echo "Blocked: direct push to main/master is prohibited. Use a feature branch." >&2
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
    'service-account.json', 'id_rsa', 'id_ed25519', 'id_ecdsa'
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

def main():
    try:
        data = json.load(sys.stdin)
    except json.JSONDecodeError:
        sys.exit(0)
    file_path = data.get('tool_input', {}).get('path') \
             or data.get('tool_input', {}).get('file_path', '')
    if file_path and is_sensitive(file_path):
        print(
            f"SECURITY: Access to '{file_path}' is blocked.\n"
            "Credential files and .env must not be read or modified by Claude.",
            file=sys.stderr
        )
        sys.exit(2)
    sys.exit(0)

if __name__ == '__main__':
    main()
```

#### .claude/hooks/post-ts-lint.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

input="$(cat)"
file="$(jq -r '.tool_input.file_path // .tool_input.path // empty' <<< "$input")"

case "$file" in
  *.ts|*.tsx|*.astro) ;;
  *) exit 0 ;;
esac

npx biome format --write "$file" >/dev/null 2>&1 || true
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
```

#### .claude/hooks/protect-config.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

input="$(cat)"
file="$(jq -r '.tool_input.file_path // .tool_input.path // empty' <<< "$input")"

protected="eslint.config biome.json pyproject.toml .prettierrc tsconfig.json lefthook.yml .oxlintrc.json astro.config tailwind.config"

for p in $protected; do
  case "$file" in
    *$p*)
      echo "BLOCKED: $file は保護された設定ファイルです。リンター設定ではなくコードを修正してください。" >&2
      exit 2
      ;;
  esac
done
```

実行権限を付与する:

```bash
chmod +x .claude/hooks/bash-firewall.sh
chmod +x .claude/hooks/protect-files.py
chmod +x .claude/hooks/post-ts-lint.sh
chmod +x .claude/hooks/protect-config.sh
```

### 2-5. .claude/rules/a11y.md

```markdown
# アクセシビリティルール

UIコンポーネントを生成・編集するすべての場面で適用する。

## strict

- インタラクティブ要素には `button` / `a` / `input` / `select` / `textarea` を使う。`div` / `span` に `onclick` をつけない
- `img` には必ず `alt` を指定する。装飾画像は `alt=""` と `aria-hidden="true"` を併用する
- フォームの `input` / `select` / `textarea` には必ず `label` を紐付ける（`for` + `id`）
- フォームのエラーメッセージは `aria-describedby` で入力要素と紐付け、`aria-invalid` を設定する
- モーダル・ダイアログには `role="dialog"` と `aria-modal="true"` を設定し、Esc キーで閉じられるようにする

## warning

- 見出しの階層を飛ばさない
- 色だけで情報を伝えない
- ローディング状態は `aria-busy="true"` で伝える
- 動的に追加されるコンテンツには `role="alert"` または `aria-live="polite"` を使う

## advisory

- タッチターゲットは 44x44px 以上を目安にする
- アニメーションには `prefers-reduced-motion` で無効化オプションを提供する
```

---

## Phase 3: CLAUDE.md

以下の内容をプロジェクトルートの `CLAUDE.md` として配置する。
`{CMS名}` の部分は選択した CMS に書き換えること。

````markdown
# CLAUDE.md

## プロジェクト概要

Astro + TypeScript + Tailwind CSS + {CMS名}

## コマンド

```bash
pnpm dev          # 開発サーバー起動
pnpm build        # プロダクションビルド
pnpm preview      # ビルド結果のプレビュー
pnpm sync         # Content Collections の型を再生成
pnpm lint         # リント（Oxlint）
pnpm typecheck    # 型チェック（tsc --noEmit）
pnpm format       # フォーマット（Biome）
```

タスク完了時は `pnpm lint && pnpm typecheck && pnpm build` をすべて実行し、エラーがない状態でコミットすること。
Content Collections のスキーマを変更したときは `pnpm sync` を先に実行すること。

## ルール

- `any` 型の使用禁止。CMS レスポンスには必ず Zod でスキーマを定義する
- `astro.config.*`、`tailwind.config.*`、`biome.json`、`tsconfig.json` などの設定ファイルは編集禁止
- `git commit --no-verify` 禁止
- named export を使う（default export は避ける。ただし Astro の pages / layouts は除く）
- CMS クライアントは `src/lib/` に配置し、コンポーネントから直接 API を呼ばない
- Content Collections のスキーマ変更後は必ず `pnpm sync` を実行する

## アーキテクチャ

```
src/
├── components/       # Astro / UI コンポーネント
├── content/          # ローカル Markdown コンテンツ
├── layouts/          # ページレイアウト
├── lib/              # CMS クライアント・ユーティリティ
├── pages/            # ルーティング
├── styles/           # グローバルスタイル
└── types/            # 型定義
content.config.ts     # Content Collections スキーマ定義
```

決定の経緯は `/docs/adr/` の ADR を参照。

## セキュアコーディングルール

生成するすべてのコードに以下を適用する。

### strict ルール（必ず守ること）

- TypeScript strict モード（`strict: true`, `noUncheckedIndexedAccess`）
- すべての外部入力・CMS レスポンスはエントリーポイントで Zod の `safeParse` でバリデーション
- `eval` / `new Function(string)` は生成しない
- シークレットはすべて環境変数から取得。ハードコード禁止

### 禁止パターン一覧

```
eval(...)
new Function(string)
innerHTML = userInput
const secret = '...'
```

### セキュリティレビュー

以下のタイミングで `/security-review` を実行して報告する:

- 外部入力を扱う処理を書いたとき
- 新しい API エンドポイントや CMS 連携を追加したとき
- 依存パッケージを追加・更新したとき

## アクセシビリティルール

UIコンポーネントを生成・編集するときに適用する。
詳細は `.claude/rules/a11y.md` を参照。

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

オシレーション検出: 同じ箇所への指摘が A→B→A と反復した場合、優れた方を directive として固定し `directive: {内容}` をコミットメッセージに記録する。

## 完了の定義

- テストがすべて通ること
- リントエラーがないこと
- 型エラーがないこと
- `pnpm build` が成功すること
````

---

## Phase 4: 動作確認

### Astro の確認

- [ ] `pnpm dev` で開発サーバーが起動する
- [ ] `pnpm build` がエラーなく完了する
- [ ] `pnpm sync` で Content Collections の型が生成される
- [ ] CMS からのデータ取得が型安全に動作する（CMS を使う場合）

### ハーネスの確認

- [ ] `.ts` / `.astro` ファイルを編集したときに Biome のフォーマットが自動適用される
- [ ] Oxlint のリントが走り、エラーがあればコンテキストに注入される
- [ ] `astro.config.*` 等の保護対象ファイルを編集しようとするとブロックされる
- [ ] コミット時に Lefthook が型チェックとリントを実行する

### セキュリティの確認

- [ ] `.claudeignore` が存在し、機密ファイルパターンが記載されている
- [ ] `~/.claude/settings.json` の JSON 構文が正しい（`jq .` で確認）
- [ ] `disableBypassPermissionsMode` が `"disable"` になっている
- [ ] `enableAllProjectMcpServers` が `false` になっている
- [ ] `.claude/hooks/` の4ファイルが存在し、実行権限がある

### MCP サーバーの確認

- [ ] `~/.claude.json` に不要な MCP サーバーが登録されていない
- [ ] `.mcp.json` に不審な MCP サーバーがない

### CLAUDE.md の確認

- [ ] プロジェクトルートに `CLAUDE.md` が配置されている（`{CMS名}` を書き換え済み）
- [ ] `.claude/rules/a11y.md` が配置されている
- [ ] Codex 連携のセクションが含まれている

---

## 運用ルール

- エージェントが新しい種類のミスをしたら、それを防ぐテストまたはリンタールールを追加する
- 一度追加したルールはすべての将来のセッションに適用される
- Content Collections のスキーマを変更したら必ず `pnpm sync` を実行してから作業を続ける
- お金が動くサービスの認証情報は AI がアクセスできる環境から完全に隔離する
