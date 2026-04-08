# framework_nextjs

Next.js (App Router) + TypeScript + Tailwind CSS + shadcn/ui 固有の設定。
bootstrap guide 生成時に `base_*` と組み合わせて使う。

---

## プロジェクト初期化

### Next.js プロジェクト作成

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

### shadcn/ui の初期化

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

### tsconfig.json の確認

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

---

## package.json スクリプト

`base_harness.md` の基本スクリプトに加えて、以下を追加する。

```json
{
  "scripts": {
    "lint:next": "next lint"
  }
}
```

`lint` は Oxlint（Claude Code のハーネスが使う）、`lint:next` は Next.js 標準の ESLint（eslint-plugin-jsx-a11y 等）。

---

## ディレクトリ構成

```
src/
├── app/              # App Router（page, layout, route）
├── components/       # UI コンポーネント
│   └── ui/           # shadcn/ui コンポーネント
├── lib/              # ユーティリティ・ヘルパー
├── hooks/            # カスタムフック
└── types/            # 型定義
```

---

## Storybook（Next.js 固有）

`base_storybook.md` のセットアップ手順に加えて、Next.js 固有の設定を行う。

### フレームワーク選択

`storybook init` を実行すると Next.js プロジェクトが自動検出され、`@storybook/nextjs` または `@storybook/nextjs-vite` が選択される。

- `@storybook/nextjs`: Webpack ベース。Next.js の設定（`next.config.js`）をそのまま引き継ぐ
- `@storybook/nextjs-vite`: Vite ベース。ビルドが速い。Next.js 15 以降で推奨

どちらでも `next/image`, `next/link`, `next/navigation`, `next/font` 等の Next.js 固有モジュールが Storybook 内で自動的に動作する。

### Server Components の Story

Server Components は Storybook 上ではクライアントコンポーネントとしてレンダリングされる。
async コンポーネント（データフェッチを含む）の Story では、データ取得関数をモックする必要がある。

```tsx
import type { Meta, StoryObj } from "@storybook/nextjs";
import { UserProfile } from "./UserProfile";

const meta = {
  component: UserProfile,
} satisfies Meta<typeof UserProfile>;

export default meta;

type Story = StoryObj<typeof meta>;

/**
 * ユーザー情報が正常に取得できた場合の表示。
 *
 * @summary 正常取得時の表示
 */
export const Default: Story = {
  args: {
    user: { name: "田中太郎", email: "tanaka@example.com" },
  },
};
```

データフェッチのモックには Storybook の [Module mocking](https://storybook.js.org/docs/writing-stories/mocking-data-and-modules/mocking-modules) を使う。

### shadcn/ui コンポーネントの Story

shadcn/ui のコンポーネント（`src/components/ui/`）はプロジェクトにコピーされたソースコードなので、Story を書くことができる。
ただし、カスタマイズしていない素の shadcn/ui コンポーネントについては Story を書く必要はない（公式ドキュメントが十分な情報源になる）。

Story を書くべきケース:

- shadcn/ui コンポーネントを拡張・カスタマイズした場合
- 複数の shadcn/ui コンポーネントを組み合わせた複合コンポーネントを作った場合
- プロジェクト固有のバリエーション（テーマ・サイズ等）を追加した場合

### Story ファイルの配置

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

コンポーネントと同じディレクトリに `.stories.tsx` を配置する（コロケーション）。

### protect-config.sh への追加

`base_harness.md` の `protect-config.sh` で保護するファイルに `.storybook/main.ts` を追加する。

```bash
protected="eslint.config biome.json pyproject.toml .prettierrc tsconfig.json lefthook.yml .oxlintrc.json next.config .storybook/main.ts"
```

---

## CLAUDE.md の Next.js 固有セクション

```markdown
## プロジェクト概要

Next.js (App Router) + TypeScript + Tailwind CSS + shadcn/ui

## コマンド

\`\`\`bash
pnpm dev          # 開発サーバー起動
pnpm build        # プロダクションビルド
pnpm lint         # リント（Oxlint）
pnpm lint:next    # リント（Next.js ESLint + jsx-a11y）
pnpm typecheck    # 型チェック（tsc --noEmit）
pnpm format       # フォーマット（Biome）
pnpm storybook    # Storybook 起動
\`\`\`

タスク完了時は `pnpm lint && pnpm typecheck && pnpm build` をすべて実行し、エラーがない状態でコミットすること。

## ルール

- named export を使う（default export は避ける。ただし Next.js の page / layout / route は除く）
- コンポーネントは `src/components/` に配置する。shadcn/ui のコンポーネントは `src/components/ui/`
- Server Components をデフォルトとし、クライアント状態が必要な場合のみ `"use client"` をつける
- 新しいコンポーネントを作ったら Story も書く（shadcn/ui の素のコンポーネントは除く）

## アーキテクチャ

\`\`\`
src/
├── app/              # App Router（page, layout, route）
├── components/       # UI コンポーネント
│   └── ui/           # shadcn/ui コンポーネント
├── lib/              # ユーティリティ・ヘルパー
├── hooks/            # カスタムフック
└── types/            # 型定義
\`\`\`

決定の経緯は `/docs/adr/` の ADR を参照。

## 完了の定義

- テストがすべて通ること
- リントエラーがないこと
- 型エラーがないこと
- `pnpm build` が成功すること
```

---

## 動作確認

- [ ] `pnpm dev` で開発サーバーが起動する
- [ ] `pnpm build` がエラーなく完了する
- [ ] shadcn/ui のコンポーネントが `src/components/ui/` に存在する
- [ ] `pnpm storybook` で Storybook が起動する
- [ ] `@storybook/nextjs` または `@storybook/nextjs-vite` がフレームワークとして設定されている
