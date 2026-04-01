# base_skill_md_prompt

以下をClaudeCodeへの指示としてそのまま使う。
`{skill_name}` と `{対象タスクの説明}` を実際の内容に置き換えて使うこと。

---

`{skill_name}` というスキルのSKILL.mdを新規作成してください。
以下の設計方針（Progressive Disclosure）に従って構成してください。

## 設計方針: Progressive Disclosure

「必要な情報を、必要なタイミングで、必要な分だけ読み込む」構造にすること。
SKILL.md単体にすべての情報を詰め込まず、詳細は references/ 以下のファイルに分離する。

これによりSKILL.mdのトークン消費を抑え、レスポンス速度と精度を改善する。

## ファイル構成

```
{skill_name}/
├── SKILL.md          # エントリーポイント（軽量に保つ）
└── references/
    ├── overview.md   # 詳細な概要・背景知識
    ├── examples.md   # 具体的なコード例・サンプル
    └── troubleshooting.md  # エラー対処・よくある問題
```

references/ 以下のファイル名はスキルの内容に応じて適切に変えてよい。
ただし troubleshooting.md は基本的に含めること（自律対処力のため）。

## SKILL.mdに書くこと（軽量エントリーポイント）

- スキルの目的・使いどころを1〜3文で説明
- 最も頻繁に使うコマンド・手順のみ（Quick Start相当）
- 各 references/ ファイルへの参照と、どんなときに読むかの案内
- 全体で200〜400トークン程度に収めることを意識する

## references/ に書くこと（詳細・オンデマンド）

- overview.md: 詳細な仕様・オプション・設計判断の背景など
- examples.md: ユースケース別のコード例・サンプル集
- troubleshooting.md: よくあるエラーと対処法、デバッグ手順

## SKILL.mdのフォーマット例

```markdown
---
name: {skill_name}
description: {対象タスクの説明}
---

# {skill_name}

## 概要

（1〜3文でスキルの目的を説明）

## Quick Start

（最も基本的な使い方のみ。コードブロック1〜2個まで）

## 詳細情報

より詳しい情報は以下を参照:
- 詳細な仕様・オプション → references/overview.md
- ユースケース別のサンプル → references/examples.md
- エラー対処・デバッグ → references/troubleshooting.md
```

## 作成後の確認事項

- SKILL.md単体が軽量に保たれているか（詳細が references/ に分離されているか）
- troubleshooting.md が含まれているか
- references/ の各ファイルへの案内がSKILL.mdに明記されているか
- 作成したファイル一覧と、各ファイルの役割を簡潔に報告すること
