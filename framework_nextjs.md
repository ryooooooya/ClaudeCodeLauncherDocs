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
\`\`\`

タスク完了時は `pnpm lint && pnpm typecheck && pnpm build` をすべて実行し、エラーがない状態でコミットすること。

## ルール

- named export を使う（default export は避ける。ただし Next.js の page / layout / route は除く）
- コンポーネントは `src/components/` に配置する。shadcn/ui のコンポーネントは `src/components/ui/`
- Server Components をデフォルトとし、クライアント状態が必要な場合のみ `"use client"` をつける

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
