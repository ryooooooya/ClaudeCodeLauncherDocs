# framework_astro

Astro + TypeScript + Tailwind CSS 固有の設定。
bootstrap guide 生成時に `base_*` と組み合わせて使う。
CMS はプロジェクトごとに選択するため、このファイルはプレースホルダーを含む。

---

## プロジェクト初期化

### Astro プロジェクト作成

```bash
pnpm create astro@latest . \
  --template minimal \
  --typescript strict \
  --install \
  --no-git
```

- `--template minimal`: 最小構成から始める
- `--typescript strict`: strict モードで TypeScript を有効化

### Tailwind CSS の追加

```bash
pnpm astro add tailwind
```

対話プロンプトで `Yes` を選択すると `tailwind.config.mjs` と `src/styles/global.css` が生成される。

### tsconfig.json の確認

`create astro` が生成した `tsconfig.json` に `strict` が含まれていることを確認する。
`noUncheckedIndexedAccess` は含まれないため手動で追加する。

```json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "noUncheckedIndexedAccess": true
  }
}
```

### astro sync の実行

スキーマ変更後は型を再生成する必要がある。

```bash
pnpm astro sync
```

Content Collections のスキーマを変更したとき・CMS ローダーを追加したときに必ず実行する。

---

## Content Collections（Astro 5+ / 6+）

Astro 6 では Content Layer API が必須。`src/content/config.ts`（旧）ではなく `src/content.config.ts` を使う。

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

`CollectionEntry` で型安全にコンポーネントの props を定義できる:

```typescript
import type { CollectionEntry } from 'astro:content'

type Props = {
  post: CollectionEntry<'blog'>
}
```

---

## CMS 選択（プロジェクトごとに決定）

以下から選択してbootstrap guideに反映する。
選択したら該当セクション以外を削除すること。

### 選択肢

| CMS | 向いているケース |
|---|---|
| ローカル Markdown | コンテンツ量が少なく、開発者がコンテンツを管理する |
| microCMS | 日本語コンテンツ主体・非エンジニアが編集する |
| Sanity | 構造化コンテンツ・リアルタイム共同編集が必要 |
| Storyblok | ビジュアルエディタが必要 |
| Contentful | 大規模・多言語・エンタープライズ |

---

### CMS: ローカル Markdown（CMS なし）

追加インストール不要。`src/content.config.ts` の Content Collections のみ使用。

コマンドの追加はなし。

---

### CMS: microCMS

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

`.env.local`:

```
MICROCMS_SERVICE_DOMAIN=your-service-domain
MICROCMS_API_KEY=your-api-key
```

Content Layer API でローダーとして組み込む場合:

```typescript
// src/content.config.ts
import { defineCollection } from 'astro:content'
import { client } from './src/lib/microcms'

const blog = defineCollection({
  loader: async () => {
    const data = await client.get({ endpoint: 'blog' })
    return data.contents.map((item: any) => ({ id: item.id, ...item }))
  },
})
```

---

### CMS: Sanity

```bash
pnpm astro add sanity
```

`astro.config.mjs` に追加:

```typescript
import { defineConfig } from 'astro/config'
import sanity from '@sanity/astro'

export default defineConfig({
  integrations: [
    sanity({
      projectId: import.meta.env.SANITY_PROJECT_ID,
      dataset: import.meta.env.SANITY_DATASET,
      useCdn: false,
    }),
  ],
})
```

`.env.local`:

```
SANITY_PROJECT_ID=your-project-id
SANITY_DATASET=production
```

---

## package.json スクリプト

`base_harness.md` の基本スクリプトに加えて、以下を追加する。

```json
{
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "sync": "astro sync"
  }
}
```

`lint` は Oxlint。Astro ファイルの script ブロックに対応しているため `.astro` も対象に含める:

```json
{
  "scripts": {
    "lint": "oxlint src --extension .ts,.tsx,.astro"
  }
}
```

---

## ディレクトリ構成

```
src/
├── components/       # Astro / UI コンポーネント
├── content/          # ローカル Markdown コンテンツ（CMS なしの場合）
├── layouts/          # ページレイアウト
├── lib/              # CMS クライアント・ユーティリティ
├── pages/            # ルーティング（.astro / .md）
├── styles/           # グローバルスタイル
└── types/            # 型定義
content.config.ts     # Content Collections スキーマ定義
```

---

## CLAUDE.md の Astro 固有セクション

```markdown
## プロジェクト概要

Astro + TypeScript + Tailwind CSS + {CMS名をここに記入}

## コマンド

\`\`\`bash
pnpm dev          # 開発サーバー起動
pnpm build        # プロダクションビルド
pnpm preview      # ビルド結果のプレビュー
pnpm sync         # Content Collections の型を再生成
pnpm lint         # リント（Oxlint）
pnpm typecheck    # 型チェック（tsc --noEmit）
pnpm format       # フォーマット（Biome）
\`\`\`

タスク完了時は `pnpm lint && pnpm typecheck && pnpm build` をすべて実行し、エラーがない状態でコミットすること。
Content Collections のスキーマを変更したときは `pnpm sync` を先に実行すること。

## ルール

- named export を使う（default export は避ける。ただし Astro の pages / layouts は除く）
- コンポーネントは `src/components/` に配置する
- CMS クライアントは `src/lib/` に配置し、コンポーネントから直接 API を呼ばない
- `any` 型の使用禁止。CMS レスポンスには必ず Zod でスキーマを定義する
- Content Collections のスキーマ変更後は必ず `pnpm sync` を実行する

## アーキテクチャ

\`\`\`
src/
├── components/       # Astro / UI コンポーネント
├── content/          # ローカル Markdown コンテンツ
├── layouts/          # ページレイアウト
├── lib/              # CMS クライアント・ユーティリティ
├── pages/            # ルーティング
├── styles/           # グローバルスタイル
└── types/            # 型定義
content.config.ts     # Content Collections スキーマ定義
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
- [ ] `pnpm sync` で Content Collections の型が生成される
- [ ] CMS からのデータ取得が型安全に動作する（CMS を使う場合）
