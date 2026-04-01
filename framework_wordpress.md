# framework_wordpress

WordPress + SWELL 子テーマ固有の設定。
bootstrap guide 生成時に `base_*` と組み合わせて使う。

Next.js / Astro とは環境が大きく異なる点に注意:

- 言語: PHP + CSS（JS は補助的）
- ビルドツールなし（子テーマは素の PHP / CSS）
- ハーネスの範囲が限定的（Biome / Oxlint の恩恵が薄い）
- ローカル環境は wp-env（Docker）

---

## 前提

- Docker Desktop がインストール済み・起動済み
- Node.js 20+（wp-env のために必要）
- pnpm がインストール済み
- SWELL 親テーマ zip 取得済み（SWELLER'S マイページからダウンロード）
- SWELL 子テーマ zip 取得済み（同上、無料）

---

## ローカル環境（wp-env）

### セットアップ

```bash
pnpm add -D @wordpress/env
```

`.wp-env.json` をプロジェクトルートに作成する:

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

`package.json` にスクリプトを追加:

```json
{
  "scripts": {
    "wp:start": "wp-env start",
    "wp:stop": "wp-env stop",
    "wp:reset": "wp-env clean all && wp-env start",
    "wp:logs": "wp-env logs"
  }
}
```

起動:

```bash
pnpm wp:start
```

- フロントエンド: `http://localhost:8888`
- 管理画面: `http://localhost:8888/wp-admin`
- ログイン: `admin` / `password`

### ディレクトリ配置

SWELL 親テーマと子テーマを `.wp-env.json` の `themes` で参照できる形に配置する:

```
project-root/
├── swell/          # SWELL 親テーマ（zip を展開）
├── swell-child/    # SWELL 子テーマ（zip を展開）
├── .wp-env.json
└── package.json
```

SWELL 親テーマは編集しない。カスタマイズはすべて子テーマで行う。

---

## SWELL 子テーマの構成

```
swell-child/
├── style.css           # テーマ定義（必須）
├── functions.php       # フック・スクリプト読み込み
├── css/
│   └── custom.css      # カスタムスタイル
└── js/
    └── custom.js       # カスタムスクリプト（必要な場合）
```

### style.css の最小構成

```css
/*
Theme Name: SWELL CHILD
Template: swell
Version: 1.0.0
*/
```

### functions.php の最小構成

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

---

## SWELL CSS カスタムプロパティ

SWELL のカスタムプロパティはアンダースコア命名（`--color_main` 形式）。
`--swl-color-*` のようなハイフン形式は SWELL では使われていない。

主要なカスタムプロパティ:

```css
/* カラー */
--color_main        /* メインカラー */
--color_main_thin   /* メインカラー（薄め） */
--color_text        /* テキストカラー */
--color_link        /* リンクカラー */

/* フォント */
--font_size_base    /* 基本フォントサイズ */
--font_size_s       /* 小 */
--font_size_l       /* 大 */

/* レイアウト */
--content_width     /* コンテンツ幅 */
```

子テーマの `css/custom.css` でこれらを参照する:

```css
.my-component {
  color: var(--color_main);
  font-size: var(--font_size_base);
}
```

カスタマイザーで設定した色はこれらの変数に自動的に反映される。
子テーマで色をハードコードせず、必ず SWELL の変数を経由する。

---

## Figma MCP との連携（デザイントークン）

Figma Variables から SWELL のカスタムプロパティに CSS を伝播させるフロー:

1. Figma に「Primitive」と「SWELL Tokens」の2コレクション構造を用意する
2. Claude Code（Figma MCP の `get_variable_defs`）でトークンを取得する
3. SWELL のアンダースコア命名に変換して `css/custom.css` に出力する

```
Figma Variables
  └── Primitive（生の色値: #1a73e8 等）
  └── SWELL Tokens（SWELL変数名に対応: color_main → Primitive.brand.primary）
```

Claude Code への指示例:

```
Figma ファイルの SWELL Tokens コレクションから get_variable_defs でトークンを取得し、
:root { --color_main: ... } の形式で swell-child/css/tokens.css に出力してください。
変数名はアンダースコア命名（--color_main 形式）を維持してください。
```

---

## ハーネスの制限事項

PHP + 素の CSS 環境では Next.js / Astro と比べてハーネスの恩恵が限定的。

| ツール | 適用可否 | 備考 |
|---|---|---|
| Biome | △ | `.js` ファイルには効く。PHP / CSS には効かない |
| Oxlint | △ | `.js` ファイルのみ |
| Lefthook | ○ | PHP_CodeSniffer と組み合わせて使える |
| PostToolUse フック | ○ | PHP / CSS ファイルにも対応するスクリプトに差し替える |

PostToolUse フックは PHP / CSS にも対応させる（後述）。

---

## package.json スクリプト

```json
{
  "scripts": {
    "wp:start": "wp-env start",
    "wp:stop": "wp-env stop",
    "wp:reset": "wp-env clean all && wp-env start",
    "wp:logs": "wp-env logs",
    "lint:php": "phpcs --standard=WordPress swell-child/",
    "lint:js": "oxlint swell-child/js"
  }
}
```

PHP_CodeSniffer のインストール（PHP がインストールされている場合）:

```bash
composer global require squizlabs/php_codesniffer wp-coding-standards/wpcs
phpcs --config-set installed_paths $(composer config -g home)/vendor/wp-coding-standards/wpcs
```

---

## CLAUDE.md の WordPress / SWELL 固有セクション

```markdown
## プロジェクト概要

WordPress + SWELL 子テーマ

## コマンド

\`\`\`bash
pnpm wp:start     # ローカル WordPress 起動（http://localhost:8888）
pnpm wp:stop      # ローカル WordPress 停止
pnpm wp:reset     # 環境リセット
pnpm lint:php     # PHP コードスタイルチェック（WordPress Coding Standards）
pnpm lint:js      # JS リント（Oxlint）
\`\`\`

## ルール

- SWELL 親テーマのファイルは編集禁止。カスタマイズはすべて子テーマ（swell-child/）で行う
- CSS カスタムプロパティは SWELL のアンダースコア命名に従う（`--color_main` 形式、`--swl-color-*` ではない）
- 子テーマで色をハードコードしない。必ず SWELL の CSS 変数（`var(--color_main)` 等）を使う
- Figma のデザイントークンを変更したときは Figma MCP 経由で `css/tokens.css` を再生成する
- functions.php にすべての処理を書かない。機能ごとにファイルを分けて `require_once` で読み込む
- WordPress の nonce を使わないフォーム送信を書かない

## アーキテクチャ

\`\`\`
swell-child/
├── style.css           # テーマ定義（必須）
├── functions.php       # フック・読み込みのみ記述
├── css/
│   ├── tokens.css      # Figma MCP から生成したデザイントークン
│   └── custom.css      # カスタムスタイル
└── js/
    └── custom.js       # カスタムスクリプト（必要な場合）
\`\`\`

## 完了の定義

- ローカル環境（http://localhost:8888）で表示が崩れていない
- `pnpm lint:php` がエラーなく完了する
- SWELL 親テーマのファイルが変更されていない
```

---

## 動作確認

- [ ] `pnpm wp:start` で `http://localhost:8888` が表示される
- [ ] SWELL 親テーマと SWELL CHILD が管理画面のテーマ一覧に表示される
- [ ] SWELL CHILD が有効化されている
- [ ] `css/custom.css` の変更がブラウザに反映される
- [ ] SWELL の CSS 変数（`--color_main` 等）が子テーマから参照できている
