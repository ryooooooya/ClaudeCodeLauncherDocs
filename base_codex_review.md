# base_codex_review

Claude Code（実装）と OpenAI Codex（レビュー）を組み合わせた開発フローの設定。
実装計画レビューとコードレビューの2つのフローをカバーする。

---

## Codex の概要

クラウドベースのソフトウェアエンジニアリングエージェント。リポジトリ全体を読み・依存関係を推論し・テストを実行した上で指摘を出す点で、静的解析ツールより深いレイヤーの問題を検出できる。モデルは gpt-5.3-codex。ChatGPT Plus 以上のプランに含まれる。

### Skill（CLI）経由を使う理由

MCP 経由では進捗が見えず、長時間の無応答が発生する。Skill 経由（スラッシュコマンド）にすると実行ログがリアルタイムで表示され、中断の判断もしやすい。

---

## CLAUDE.md に追記するセクション

以下の内容を CLAUDE.md の適切なセクションに追記する。
既存のレビュー・コミット関連セクションがあれば統合すること。

---

### 実装計画レビュー（Codex 連携）

実装計画をユーザーに提示する前に、必ず Codex でレビューを行う。

手順:

1. 計画をファイルに書き出す
2. Codex に致命的な問題のみ指摘させる（スタイルや軽微な点は無視させる）
3. 指摘があれば計画を修正し、再レビューする
4. OK が出たらユーザーに提示する

初回レビュー:

```bash
codex exec -m gpt-5.3-codex "このプランをレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {plan_full_path} (ref: {CLAUDE.md full_path})"
```

修正後の再レビュー（文脈を引き継ぐため `resume --last` を必ずつける）:

```bash
codex exec resume --last -m gpt-5.3-codex "プランを更新したからレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {plan_full_path} (ref: {CLAUDE.md full_path})"
```

---

### コードレビュー（Codex 連携）

実装完了後、ユーザーに報告する前に Codex でコードレビューを行う。

手順:

1. 実装をコミットする
2. コミットハッシュを指定して Codex にレビューさせる
3. 指摘の内容を自分で判断し、対応が必要なものは修正する
4. 修正後は `resume --last` で再レビューする
5. 問題がなくなったらユーザーに報告する

単一コミットのレビュー:

```bash
codex exec -m gpt-5.3-codex "このコードをレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {commit_hash} (ref: {CLAUDE.md full_path})"
```

範囲指定:

```bash
codex exec -m gpt-5.3-codex "このコードをレビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {start_hash}..{end_hash} (ref: {CLAUDE.md full_path})"
```

修正後の再レビュー:

```bash
codex exec resume --last -m gpt-5.3-codex "修正したから再レビューして。瑣末な点へのクソリプはしないで。致命的な点だけ指摘して: {commit_hash} (ref: {CLAUDE.md full_path})"
```

レビュー指摘の判断基準:

- セキュリティ上の問題 → 必ず修正
- ロジックの誤り → 必ず修正
- パフォーマンス上の重大な問題 → 修正
- スタイル・命名・コメント等 → 無視（瑣末な点として扱う）

オシレーション検出:

同じ箇所への指摘が2回以上反復した場合（A→B→A のパターン）、両方のアプローチを比較して優れた方を directive として固定し、以降はそれに従う。directive はコミットメッセージに `directive: {内容}` として記録する。

---

## GitHub PR での @codex review（オプション）

Skill（CLI）経由とは別に、GitHub 連携経由のレビュー機能も使える。

セットアップ:

1. Codex cloud を有効化（ChatGPT アカウントと GitHub リポジトリを接続）
2. [Codex の設定画面](https://chatgpt.com/codex/settings/code-review)でリポジトリの "Code review" をオン

PR のコメント欄に `@codex review` と書くだけで起動する。"Automatic reviews" をオンにすると PR オープン時に自動で走る。

AGENTS.md にプロジェクト固有のレビュー観点を書くと Codex がそれに従う:

```markdown
## Review guidelines

- Prisma トランザクションが必要な箇所に抜け漏れがないか確認する
- ファイルアップロード処理でマジックナンバー検証が行われているか
- PII のログ出力がないか確認する
```

### Skill（CLI）と @codex review の比較

| 観点 | Skill（CLI） | @codex review（GitHub） |
|---|---|---|
| トリガー | `/codex` コマンド | PR コメント |
| 進捗の可視性 | リアルタイムで見える | 見えない |
| 自動化 | 手動呼び出し | PR ごとに自動化できる |
| カスタマイズ | Skill の指示文 | AGENTS.md |
| 実装途中での利用 | できる | PR ベースなので難しい |

---

## CLAUDE.md への追記方法

このファイルの内容を Claude Code に渡す場合の指示:

```
現在の CLAUDE.md を読んで、base_codex_review.md の「CLAUDE.md に追記するセクション」の内容を
既存の構成に自然に馴染む形で追記してください。
既存のレビュー・コミット関連セクションがあればそこに統合し、
なければ新規セクションとして追加してください。
追記後、どこに何を追加したかを簡潔に報告してください。
```
