# project_bootstrap_guide_nextjs

空のリポジトリから Next.js プロジェクトを立ち上げ、
Claude Code のハーネス・セキュリティ・Storybook・a11y・Codex 連携まで一気にセットアップする手順。
Phase 0 から順番に実行すれば完了する。

各設定の背景・理由は以下を参照:

- ハーネスの設計思想 → `base_harness.md`
- セキュリティ（環境制御）→ `base_security_env.md` / `base_security_env_guide.md`
- セキュリティ（生成コード）→ `base_security_code.md` / `base_security_code_guide.md`
- Storybook + AI 連携 → `base_storybook.md`
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

## Phase 0: Next.js プロジェクト初期化

### 0-1. プロジェクト作成

```bash
pnpm create next-app@latest . \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --use-pnpm
```

- App Router (`--app`)
- `src/` ディレクトリ (`--src-dir`)
- TypeScript, Tailwind CSS, ESLint を有効化

### 0-2. shadcn/ui の初期化

```bash
pnpm dlx shadcn@latest init
```

対話プロンプトの推奨設定:

- Style: Default
- Base color: Slate
- CSS variables: Yes

初期化後、よく使うコンポーネントを追加する:

```bash
pnpm dlx shadcn@latest add button input label card dialog
```

### 0-3. tsconfig.json の確認

`create-next-app` が生成した `tsconfig.json` に以下が含まれていることを確認する。
`noUncheckedIndexedAccess` はデフォルトに含まれないため手動で追加する。

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

### 0-4. git 初期化とコミット

```bash
git init
git add -A
git commit -m "init: Next.js + Tailwind + shadcn/ui"
```

---

## Phase 1: ハーネス

コード品質を自動で維持する仕組みを入れる。

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

既存の `scripts` に以下を追加する（`dev`, `build`, `start` は残す）。

```json
{
  "scripts": {
    "lint": "oxlint src",
    "lint:next": "next lint",
    "format": "biome format --write src",
    "typecheck": "tsc --noEmit"
  }
}
```

`lint` は Oxlint（Claude Code のハーネスが使う）、`lint:next` は Next.js 標準の ESLint（eslint-plugin-jsx-a11y 等）。

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

PostToolUse フックとリンター設定保護フックを登録する。

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
  *.ts|*.tsx|*.js|*.jsx) ;;
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

protected="eslint.config biome.json pyproject.toml .prettierrc tsconfig.json lefthook.yml .oxlintrc.json next.config .storybook/main.ts"

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

UIコンポーネント生成時のアクセシビリティルール。

```markdown
# アクセシビリティルール

UIコンポーネントを生成・編集するすべての場面で適用する。

## strict

- インタラクティブ要素には `button` / `a` / `input` / `select` / `textarea` を使う。`div` / `span` に `onClick` をつけない
- `img` には必ず `alt` を指定する。装飾画像は `alt=""` と `aria-hidden="true"` を併用する
- フォームの `input` / `select` / `textarea` には必ず `label` を紐付ける（`htmlFor` + `id`）
- フォームのエラーメッセージは `aria-describedby` で入力要素と紐付け、`aria-invalid` を設定する
- モーダル・ダイアログには `role="dialog"` と `aria-modal="true"` を設定し、Esc キーで閉じられるようにする
- フォーカス管理: モーダルを開いたらモーダル内にフォーカスを移動し、閉じたらトリガー要素に戻す

## warning

- 見出しの階層を飛ばさない（h1 → h3 のように h2 を飛ばさない）
- 色だけで情報を伝えない（エラー状態を赤色のみで表現しない。アイコンやテキストを併用する）
- ローディング状態は `aria-busy="true"` で伝える
- 動的に追加されるコンテンツ（トースト・通知）には `role="alert"` または `aria-live="polite"` を使う

## advisory

- タッチターゲットは 44x44px 以上を目安にする
- アニメーションには `prefers-reduced-motion` で無効化オプションを提供する
- 新規コンポーネントには jest-axe のテストを含める
```

---

## Phase 3: Storybook

コンポーネントの開発・テスト・ドキュメント環境を構築し、Claude Code の MCP server と接続する。

### 3-1. Storybook のインストール

```bash
pnpm dlx storybook@latest init
```

Next.js プロジェクトが自動検出され、`@storybook/nextjs` または `@storybook/nextjs-vite` が選択される。

- `@storybook/nextjs`: Webpack ベース。`next.config.js` の設定をそのまま引き継ぐ
- `@storybook/nextjs-vite`: Vite ベース。ビルドが速い。Next.js 15 以降で推奨

どちらでも `next/image`, `next/link`, `next/navigation`, `next/font` 等の Next.js 固有モジュールが Storybook 内で自動的に動作する。

### 3-2. addon-mcp のインストール

```bash
pnpm dlx storybook add @storybook/addon-mcp
```

これにより以下が有効になる:

- `http://localhost:6006/mcp` で MCP server が起動する
- Manifest（components / docs）が自動生成される
- AI エージェントがコンポーネント情報の参照・テスト実行をできるようになる

### 3-3. react-docgen-typescript の設定

Manifest の Props 情報精度を上げるため、`.storybook/main.ts` に以下を追加する。

```ts
const config: StorybookConfig = {
  // ...
  typescript: {
    reactDocgen: "react-docgen-typescript",
  },
};
```

`react-docgen-typescript` は `react-docgen` より遅いが、Props の型情報がより正確になる。
Manifest 生成が遅い場合は `react-docgen` に切り替えてもよい。

### 3-4. package.json にスクリプトを追加

既存の `scripts` に以下を追加する。

```json
{
  "scripts": {
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  }
}
```

### 3-5. Claude Code の MCP 接続

Storybook を起動した状態で、Claude Code に MCP server を登録する。

```bash
pnpm storybook &
npx mcp-add --type http --url "http://localhost:6006/mcp" --scope project
```

名前は `{project-name}-sb-mcp` のように、プロジェクトを特定できる命名にする。

手動で `.mcp.json` に追加する場合:

```json
{
  "mcpServers": {
    "{project-name}-sb-mcp": {
      "type": "http",
      "url": "http://localhost:6006/mcp"
    }
  }
}
```

### 3-6. .claude/rules/storybook.md の配置

Claude Code がコンポーネント実装時に常時参照する Story 作成ルールを配置する。

```markdown
# Storybook Story ルール

## Story の基本方針

- 1 Story = 1 概念。複数のバリエーションを1つの Story にまとめない
- CSF 3 形式で書く（`satisfies Meta<typeof Component>` + `StoryObj`）
- named export を使う（default export は meta のみ）
- Story 名は状態やユースケースを表す名前にする（`Primary`, `Disabled`, `WithIcon` 等）

## JSDoc コメント（必須）

AI エージェントが Manifest 経由でコンポーネントを理解するための最重要情報源。

### コンポーネントの JSDoc

コンポーネントの export に description と summary を書く。
summary はエージェントが最初に受け取る情報なので、用途を簡潔に伝える。

\`\`\`tsx
/**
 * ユーザー操作に使うボタン。ページ遷移には Link を使うこと。
 *
 * @summary ユーザー操作用のボタン（遷移には Link を使う）
 */
export const Button = // ...
\`\`\`

### Props の JSDoc

すべての Props に description を書く。

\`\`\`tsx
export interface ButtonProps {
  /** ボタンテキストの前に表示するアイコン */
  icon?: ReactNode;
  /** ボタンを無効化する */
  disabled?: boolean;
}
\`\`\`

### Story の JSDoc

各 Story にも description と summary を書く。
「何を」ではなく「なぜこの状態を使うか」を記述する。

\`\`\`tsx
/**
 * Primary ボタンは画面のメインアクションに使う。
 * 1画面に複数の Primary ボタンを配置しない。
 *
 * @summary 画面のメインアクションに使う
 */
export const Primary: Story = {
  args: { primary: true },
};
\`\`\`

## Manifest の管理

- エージェントに見せる必要がない Story（アンチパターン例・deprecated 等）には `tags: ['!manifest']` を付ける
- MDX ドキュメントも同様に `<Meta tags={['!manifest']} />` で除外できる
- 不要な情報が多すぎるとエージェントのパフォーマンスが落ちる。必要なものだけ含める

## テストの活用

- インタラクションテストは play function で書く
- Story を書いたら `run-story-tests` で動作確認する
- a11y テストが設定されている場合、アクセシビリティの問題も自動検出される
```

### 3-7. Story ファイルの配置規則

コンポーネントと同じディレクトリに `.stories.tsx` を配置する（コロケーション）。

```
src/
├── components/
│   ├── ui/                    # shadcn/ui（カスタマイズ時のみ Story を書く）
│   ├── LoginForm/
│   │   ├── LoginForm.tsx
│   │   └── LoginForm.stories.tsx
│   └── Header/
│       ├── Header.tsx
│       └── Header.stories.tsx
```

shadcn/ui のコンポーネント（`src/components/ui/`）はプロジェクトにコピーされたソースコードなので Story を書くことができる。ただし、カスタマイズしていない素の shadcn/ui コンポーネントについては Story を書く必要はない。

Story を書くべきケース:

- shadcn/ui コンポーネントを拡張・カスタマイズした場合
- 複数の shadcn/ui コンポーネントを組み合わせた複合コンポーネントを作った場合
- プロジェクト固有のバリエーション（テーマ・サイズ等）を追加した場合

### 3-8. コミット

```bash
git add -A
git commit -m "feat: add Storybook + MCP server"
```

---

## Phase 4: CLAUDE.md

以下の内容をプロジェクトルートの `CLAUDE.md` として配置する。
プロジェクト固有のパス・命名規則は適宜書き換えること。
`{project-name}-sb-mcp` は Phase 3-5 で登録した MCP server 名に置き換える。

````markdown
# CLAUDE.md

## プロジェクト概要

Next.js (App Router) + TypeScript + Tailwind CSS + shadcn/ui

## コマンド

```bash
pnpm dev          # 開発サーバー起動
pnpm build        # プロダクションビルド
pnpm lint         # リント（Oxlint）
pnpm lint:next    # リント（Next.js ESLint + jsx-a11y）
pnpm typecheck    # 型チェック（tsc --noEmit）
pnpm format       # フォーマット（Biome）
pnpm storybook    # Storybook 起動
```

タスク完了時は `pnpm lint && pnpm typecheck && pnpm build` をすべて実行し、エラーがない状態でコミットすること。

## ルール

- `any` 型の使用禁止。不明な場合は `unknown` を使い型ガードで絞り込む
- `eslint.config`、`biome.json`、`tsconfig.json`、`next.config.*`、`.storybook/main.ts` などの設定ファイルは編集禁止
- `git commit --no-verify` 禁止
- named export を使う（default export は避ける。ただし Next.js の page / layout / route は除く）
- コンポーネントは `src/components/` に配置する。shadcn/ui のコンポーネントは `src/components/ui/`
- Server Components をデフォルトとし、クライアント状態が必要な場合のみ `"use client"` をつける
- 新しいコンポーネントを作ったら Story も書く（shadcn/ui の素のコンポーネントは除く）

## アーキテクチャ

```
src/
├── app/              # App Router（page, layout, route）
├── components/       # UI コンポーネント
│   └── ui/           # shadcn/ui コンポーネント
├── lib/              # ユーティリティ・ヘルパー
├── hooks/            # カスタムフック
└── types/            # 型定義
```

決定の経緯は `/docs/adr/` の ADR を参照。

## Storybook

UI コンポーネントの実装・修正時は `{project-name}-sb-mcp` MCP ツールを使って
Storybook のコンポーネント・ドキュメント情報を確認してから作業すること。

- コンポーネントのプロパティを使う前に必ず `get-documentation` で確認する。推測で Props を使わない
- `list-all-documentation` で利用可能なコンポーネント一覧を取得できる
- Story の書き方は `get-storybook-story-instructions` で最新のルールを取得する
- 作業後は `run-story-tests` でテストを実行し、失敗があれば修正してから完了とする

## セキュアコーディングルール

生成するすべてのコードに以下を適用する。
詳細は `base_security_code.md` を参照。

### strict ルール（必ず守ること）

- TypeScript strict モード（`strict: true`, `noImplicitAny`, `strictNullChecks`, `noUncheckedIndexedAccess`）
- すべての外部入力はエントリーポイントで Zod の `safeParse` でバリデーション
- SQL は必ずパラメータ化クエリまたは ORM の安全なメソッドを使用
- `exec` / `execSync` にユーザー入力を渡さない。`spawn` を使い引数は配列で渡す
- `eval` / `new Function(string)` は生成しない
- 外部 JSON のマージ前に `__proto__` / `constructor` / `prototype` キーを除去
- `innerHTML` への直接代入禁止。DOMPurify でサニタイズするか `textContent` を使う
- ファイルパスは `path.resolve` で正規化してベースディレクトリ内か検証する
- パスワードハッシュは `argon2`（または `bcrypt`）。`md5` / `sha1` / `sha256` は禁止
- トークン比較は `crypto.timingSafeEqual` を使う
- JWT の `verify` では `algorithms` を明示する。`alg: "none"` を受け入れない
- シークレットはすべて環境変数から取得。ハードコード禁止

### warning ルール

- CORS は許可オリジンを明示。`origin: '*'` は禁止
- 認証エンドポイントにはレート制限を設ける
- スタックトレース・内部パスをクライアントに返さない

### 禁止パターン一覧

```
eval(...)
new Function(string)
exec(`...${userInput}...`)
innerHTML = userInput
Object.assign(target, JSON.parse(untrustedInput))
alg: 'none'
md5 / sha1 でのパスワードハッシュ
const secret = '...'
`SELECT ... WHERE id = ${userInput}`
```

### セキュリティレビュー

以下のタイミングで `/security-review` を実行して報告する:

- 認証・認可に関わるコードを書いたとき
- 外部入力を DB や shell に渡す処理を書いたとき
- 新しい API エンドポイントを追加したとき
- 依存パッケージを追加・更新したとき

## アクセシビリティルール

UIコンポーネントを生成・編集するときに適用する。
詳細は `.claude/rules/a11y.md` を参照。

## 実装計画レビュー（Codex 連携）

実装計画をユーザーに提示する前に、必ず Codex でレビューを行う。

初回レビュー:

```bash
codex exec -m gpt-5.3-codex "このプランをレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {plan_full_path} (ref: {CLAUDE.md full_path})"
```

修正後の再レビュー（`resume --last` を必ずつける）:

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

## Phase 5: 動作確認

### Next.js の確認

- [ ] `pnpm dev` で開発サーバーが起動する
- [ ] `pnpm build` がエラーなく完了する
- [ ] shadcn/ui のコンポーネントが `src/components/ui/` に存在する

### ハーネスの確認

- [ ] `.ts` / `.tsx` ファイルを編集したときに Biome のフォーマットが自動適用される
- [ ] Oxlint のリントが走り、エラーがあればコンテキストに注入される
- [ ] `biome.json` 等の保護対象ファイルを編集しようとするとブロックされる
- [ ] コミット時に Lefthook が型チェックとリントを実行する

### セキュリティの確認

- [ ] `.claudeignore` が存在し、機密ファイルパターンが記載されている
- [ ] `~/.claude/settings.json` の JSON 構文が正しい（`jq .` で確認）
- [ ] `disableBypassPermissionsMode` が `"disable"` になっている
- [ ] `enableAllProjectMcpServers` が `false` になっている
- [ ] `.claude/hooks/` の4ファイルが存在し、実行権限がある

### Storybook の確認

- [ ] `pnpm storybook` で Storybook が起動する
- [ ] `@storybook/nextjs` または `@storybook/nextjs-vite` がフレームワークとして設定されている
- [ ] `http://localhost:6006/mcp` にアクセスすると MCP server のページが表示される
- [ ] `http://localhost:6006/manifests/components.html` で Manifest デバッガーが表示される
- [ ] Claude Code から `list-all-documentation` ツールを呼び出してコンポーネント一覧が返る
- [ ] `.claude/rules/storybook.md` が配置されている

### MCP サーバーの確認

- [ ] `~/.claude.json` に不要な MCP サーバーが登録されていない
- [ ] `.mcp.json` に Storybook MCP server が登録されている
- [ ] `.mcp.json` に不審な MCP サーバーがない

### CLAUDE.md の確認

- [ ] プロジェクトルートに `CLAUDE.md` が配置されている
- [ ] セキュアコーディングルールが含まれている
- [ ] Storybook セクション（MCP ツール名・利用手順）が含まれている
- [ ] `.claude/rules/a11y.md` が配置されている
- [ ] `.claude/rules/storybook.md` が配置されている
- [ ] Codex 連携（計画レビュー・コードレビュー）のセクションが含まれている

---

## 運用ルール

- エージェントが新しい種類のミスをしたら、それを防ぐテストまたはリンタールールを追加する
- 一度追加したルールはすべての将来のセッションに適用される。これがハーネスの複利効果の源泉
- 信頼できないリポジトリを開くときは MCP 無効化・`package.json` の scripts 確認・`.claude/settings.json` の hooks 確認を行う
- お金が動くサービスの認証情報は AI がアクセスできる環境から完全に隔離する
- Storybook の MCP server は開発中常時起動しておく。エージェントがコンポーネント情報にアクセスできる状態を維持する
- 新しいコンポーネントを作ったら必ず Story を書く。Story がないコンポーネントは Manifest に載らず、エージェントから見えない
