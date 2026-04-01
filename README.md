# README

Claude Code でのプロジェクト開発を支援するドキュメント群。
bootstrap guide の生成・更新と、各種設定の参照に使う。

---

## ファイル構成

### base_*（フレームワーク非依存）

| ファイル | 内容 |
|---|---|
| `base_harness.md` | Biome + Oxlint + Lefthook + フック設定 |
| `base_security_env.md` | .claudeignore, settings.json, セキュリティフック（Claude Code への指示） |
| `base_security_env_guide.md` | セキュリティ設定の背景・理由（人間向け解説） |
| `base_security_code.md` | TypeScript / Node.js セキュアコーディングルール（Claude Code への指示） |
| `base_security_code_guide.md` | セキュアコーディングルールの背景・理由（人間向け解説） |
| `base_security_npm.md` | npm サプライチェーンセキュリティ: 依存関係の追加・更新・緊急対応手順（Claude Code への指示） |
| `base_codex_review.md` | Codex による計画レビュー・コードレビュー連携 |
| `base_a11y.md` | アクセシビリティセットアップ（Playwright + jest-axe） |
| `base_ux_checklist_critical.md` | UX チェックリスト（CRITICAL: 常時適用） |
| `base_ux_checklist_high.md` | UX チェックリスト（HIGH: UI実装・レビュー時） |
| `base_ux_checklist_medium.md` | UX チェックリスト（MEDIUM/LOW: 該当機能の実装時） |
| `base_ui_motion.md` | UIの触感・質感（アニメーション・インタラクションフィードバック・ジェスチャー応答） |
| `base_claude_md_knowledge.md` | CLAUDE.md の設計・運用に関する知識まとめ |
| `base_skill_md_prompt.md` | SKILL.md 生成プロンプト |

### framework_*（フレームワーク固有）

| ファイル | 内容 |
|---|---|
| `framework_nextjs.md` | Next.js (App Router) + shadcn/ui 固有の設定 |
| `framework_astro.md` | Astro 固有の設定（CMS はプロジェクトごとに選択） |
| `framework_wordpress.md` | WordPress / SWELL 子テーマ固有の設定 |

### bootstrap guide（組み立て済み成果物）

| ファイル | 対応 `framework_*` | 内容 |
|---|---|---|
| `project_bootstrap_guide_nextjs.md` | `framework_nextjs.md` | Next.js (App Router) + shadcn/ui |
| `project_bootstrap_guide_astro.md` | `framework_astro.md` | Astro + Tailwind（CMS はプレースホルダー） |
| `project_bootstrap_guide_wordpress.md` | `framework_wordpress.md` | WordPress + SWELL 子テーマ + wp-env |

---

## bootstrap guide の生成・更新方法

bootstrap guide は `base_*` と `framework_*` を組み合わせた成果物として生成する。
直接編集せず、元ファイルを更新してから再生成する。

### 新規生成（Next.js 版）

```
base_harness.md, base_security_env.md, base_security_code.md,
base_codex_review.md, base_a11y.md, base_ui_motion.md, framework_nextjs.md を参照して、
Next.js プロジェクトのセットアップ手順を Phase 0 から順番に実行できる
project_bootstrap_guide_nextjs.md を生成してください。
```

### 新規生成（Astro 版）

```
base_harness.md, base_security_env.md, base_security_code.md,
base_codex_review.md, base_a11y.md, base_ui_motion.md, framework_astro.md を参照して、
Astro プロジェクトのセットアップ手順を Phase 0 から順番に実行できる
project_bootstrap_guide_astro.md を生成してください。
```

### 新規生成（WordPress 版）

```
base_security_env.md, base_codex_review.md, framework_wordpress.md を参照して、
WordPress + SWELL 子テーマのセットアップ手順を Phase 0 から順番に実行できる
project_bootstrap_guide_wordpress.md を生成してください。
```

### base_* 更新後の再生成

```
base_security_env.md を更新しました。
これを反映して project_bootstrap_guide_nextjs.md を再生成してください。
```

---

## フレームワーク間の主な差異

| 観点 | Next.js | Astro | WordPress |
|---|---|---|---|
| 言語 | TypeScript | TypeScript | PHP + CSS |
| ハーネス（Biome / Oxlint） | フル活用 | フル活用（.astro 対応） | JS ファイルのみ |
| CMS | なし | プロジェクトごとに選択 | SWELL |
| ローカル環境 | `pnpm dev` | `pnpm dev` | wp-env（Docker） |
| セキュアコーディング | TypeScript / Node.js ルール | TypeScript / Node.js ルール | PHP WordPress ルール |
| UI Motion | Framer Motion パターン含む | Framer Motion パターン含む | CSS のみ（参考適用） |

---

## 運用ルール

### ファイルの役割分担

- `base_*` と `framework_*` が「ソース」。直接編集してよい唯一のファイル群
- bootstrap guide は「生成物」。直接編集しない。変更は必ず元ファイルに入れてから再生成する
- このルールを守らないと `base_*` と bootstrap guide の内容が乖離して、次の再生成時に意図しない差分が出る

### base_* を更新したとき

1. 該当の `base_*` を更新する
2. その内容を使っているすべての bootstrap guide を再生成する
3. このプロジェクト知識のファイルを差し替える

どの bootstrap guide が影響を受けるかは「フレームワーク間の主な差異」表を参照。
`base_harness.md` や `base_security_env.md` などフレームワーク非依存のものを変えた場合は、原則すべての bootstrap guide が再生成の対象になる。

### framework_* を更新したとき

1. 該当の `framework_*` を更新する
2. 対応する bootstrap guide のみ再生成する（他のフレームワークには影響しない）
3. このプロジェクト知識のファイルを差し替える

### 新しいフレームワーク・CMS を追加したとき

1. `framework_{name}.md` を新規作成する
2. 対応する `project_bootstrap_guide_{name}.md` を生成する
3. このREADMEのファイル一覧と bootstrap guide 一覧・フレームワーク間の差異表を更新する
4. このプロジェクト知識に3ファイルをまとめて追加する

### ベストプラクティス・外部情報を反映したとき

1. 反映先の `base_*` を特定して更新する
2. 影響する bootstrap guide を再生成する
3. このプロジェクト知識のファイルを差し替える

「どの `base_*` に反映すべきか」判断に迷う場合はこのプロジェクトで相談すること。

### このREADMEを更新するタイミング

- ファイルの追加・削除・リネームをしたとき
- 運用の考え方が変わったとき

---

## 参考

- Zenn: Claude Code / MCP を安全に使うための実践ガイド
- Trail of Bits / claude-code-config
- owayo: Claude Code と Codex の連携を MCP から Skill に変えたら体験が劇的に改善した
