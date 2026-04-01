# ux_checklist_medium

タイポグラフィ、フォーム、チャートに関するルール。
該当する実装を行うときにオンデマンドで参照する。
出典: Apple HIG, Material Design, WCAG

---

## Typography & Color（MEDIUM）

- `line-height` — 本文テキストの行の高さは1.5〜1.75
- `line-length` — 1行あたり65〜75文字に制限
- `font-pairing` — 見出しと本文のフォントの性格を合わせる
- `font-scale` — 一貫したタイプスケールを使用（例: 12 14 16 18 24 32）
- `contrast-readability` — ライト背景には暗いテキスト（例: slate-900 on white）
- `text-styles-system` — プラットフォームのタイプシステムを使用（iOS Dynamic Type / Material 5 type roles）
- `weight-hierarchy` — font-weightで階層を強化。見出し600〜700、本文400、ラベル500
- `color-semantic` — セマンティックカラートークンを定義（primary, secondary, error, surface等）。コンポーネントに直接hexを書かない
- `color-dark-mode` — ダークモードではデサチュレーション/明るいトーナルバリアント。色の反転ではない
- `color-accessible-pairs` — 前景/背景ペアのコントラスト比がAA（4.5:1）またはAAA（7:1）を満たすこと
- `color-not-decorative-only` — 機能的な色（エラー赤、成功緑）にはアイコン/テキストを併用
- `truncation-strategy` — 切り捨てより折り返しを優先。切り捨てる場合は省略記号+ツールチップ/展開で全文を提供
- `letter-spacing` — プラットフォームデフォルトのletter-spacingを尊重。本文テキストのタイトなトラッキングは避ける
- `number-tabular` — データ列、価格、タイマーにはtabular/等幅数字を使用してレイアウトシフトを防止
- `whitespace-balance` — 余白を意図的に使って関連項目をグループ化し、セクションを分離。視覚的な雑然さを避ける

## Forms & Feedback（MEDIUM）

- `input-labels` — 入力ごとに可視ラベルを設置（プレースホルダーのみは禁止）
- `error-placement` — エラーは関連フィールドの下に表示
- `submit-feedback` — 送信時にローディング→成功/エラー状態を表示
- `required-indicators` — 必須フィールドをマーク（例: アスタリスク）
- `empty-states` — コンテンツがないときは有用なメッセージとアクションを表示
- `toast-dismiss` — トーストは3〜5秒で自動消去
- `confirmation-dialogs` — 破壊的アクションの前に確認ダイアログを表示
- `input-helper-text` — 複雑な入力の下に永続的なヘルパーテキストを提供（プレースホルダーだけでなく）
- `disabled-states` — 無効要素は不透明度を下げ（0.38〜0.5）+カーソル変更+セマンティック属性
- `progressive-disclosure` — 複雑なオプションを段階的に表示。最初からユーザーを圧倒しない
- `inline-validation` — blur時にバリデーション（キー入力ごとではない）。入力完了後にエラーを表示
- `input-type-keyboard` — セマンティックinputタイプ（email, tel, number）で適切なモバイルキーボードを起動
- `password-toggle` — パスワードフィールドに表示/非表示トグルを提供
- `autofill-support` — autocomplete / textContentType属性でシステムの自動入力をサポート
- `undo-support` — 破壊的/一括アクションに取り消しを許可（例: 「削除を取り消し」トースト）
- `success-feedback` — 完了アクションを簡潔な視覚フィードバック（チェックマーク、トースト、色フラッシュ）で確認
- `error-recovery` — エラーメッセージには明確なリカバリーパス（再試行、編集、ヘルプリンク）を含める
- `multi-step-progress` — 複数ステップのフローにはステップインジケーターまたはプログレスバーを表示。戻るナビゲーションを許可
- `form-autosave` — 長いフォームは意図しない解除によるデータ損失を防ぐため下書きを自動保存
- `sheet-dismiss-confirm` — 未保存の変更があるシート/モーダルの解除前に確認
- `error-clarity` — エラーメッセージは原因+修正方法を述べる（「無効な入力」だけでは不十分）
- `field-grouping` — 関連フィールドを論理的にグループ化（fieldset/legendまたは視覚的グループ化）
- `read-only-distinction` — 読み取り専用状態は無効状態と視覚的・セマンティックに異なること
- `focus-management` — 送信エラー後、最初の無効フィールドに自動フォーカス
- `error-summary` — 複数エラーの場合、各フィールドへのアンカーリンク付きサマリーをトップに表示
- `touch-friendly-input` — モバイルのinput高さは44px以上（タッチターゲット要件を満たす）
- `destructive-emphasis` — 破壊的アクションはセマンティックな危険色（赤）を使い、プライマリアクションから視覚的に分離
- `toast-accessibility` — トーストはフォーカスを奪わない。スクリーンリーダー通知に`aria-live="polite"`を使用
- `aria-live-errors` — フォームエラーは`aria-live`リージョンまたは`role="alert"`でスクリーンリーダーに通知
- `contrast-feedback` — エラー/成功状態の色はコントラスト比4.5:1を満たすこと
- `timeout-feedback` — リクエストタイムアウト時は再試行オプション付きの明確なフィードバックを表示

## Charts & Data（LOW）

- `chart-type` — チャートタイプをデータタイプに合わせる（トレンド→折れ線、比較→棒、割合→円/ドーナツ）
- `color-guidance` — アクセシブルなカラーパレットを使用。赤/緑のみのペアは色覚障害者向けに避ける
- `data-table` — アクセシビリティのためテーブル代替を提供。チャートだけではスクリーンリーダーに不親切
- `pattern-texture` — 色に加えてパターン、テクスチャ、形状を使い、色なしでもデータが区別可能にする
- `legend-visible` — 凡例を常に表示。チャートの近くに配置し、スクロールフォールドの下に離さない
- `tooltip-on-interact` — ホバー（Web）またはタップ（モバイル）で正確な値を表示するツールチップ/データラベルを提供
- `axis-labels` — 軸に単位と読みやすいスケールのラベルを設置。モバイルで切り捨てや回転したラベルは避ける
- `responsive-chart` — チャートは小画面でリフローまたは簡略化（例: 横棒に切替、ティック削減）
- `empty-data-state` — データがないとき意味のある空状態を表示（「まだデータがありません」+ガイダンス）。空のチャートは禁止
- `loading-chart` — チャートデータ読み込み中はスケルトン/シマープレースホルダーを使用。空の軸フレームを表示しない
- `animation-optional` — チャートの入場アニメーションは`prefers-reduced-motion`を尊重。データは即座に読めること
- `large-dataset` — 1000件以上のデータポイントは集約またはサンプリング。詳細にはドリルダウンを提供
- `number-formatting` — 軸とラベルにロケール対応の数値/日付/通貨フォーマットを使用
- `touch-target-chart` — インタラクティブなチャート要素（ポイント、セグメント）は44pt以上のタップエリア、またはタッチで拡大
- `no-pie-overuse` — 5カテゴリ超の円/ドーナツは避ける。明瞭さのため棒グラフに切り替え
- `contrast-data` — データ線/棒 vs 背景は3:1以上、データテキストラベルは4.5:1以上
- `legend-interactive` — 凡例はクリックでシリーズの表示/非表示を切り替え可能にする
- `direct-labeling` — 小規模データセットでは値をチャートに直接ラベル付けして視線移動を削減
- `tooltip-keyboard` — ツールチップはキーボードで到達可能であること。ホバーだけに依存しない
- `sortable-table` — データテーブルはソート対応とし、`aria-sort`で現在のソート状態を示す
- `axis-readability` — 軸ティックが詰まらないこと。読みやすい間隔を維持し、小画面では自動スキップ
- `data-density` — チャートあたりの情報密度を制限して認知負荷を避ける。必要なら複数チャートに分割
- `trend-emphasis` — 装飾よりデータトレンドを強調。データを覆い隠す重いグラデーション/シャドウは避ける
- `gridline-subtle` — グリッド線はローコントラスト（例: gray-200）でデータと競合しないように
- `focusable-elements` — インタラクティブなチャート要素（ポイント、棒、スライス）はキーボードでナビゲーション可能
- `screen-reader-summary` — チャートの主要な洞察を説明するテキストサマリーまたはaria-labelをスクリーンリーダー向けに提供
- `error-state-chart` — データ読み込み失敗時は再試行アクション付きのエラーメッセージを表示。壊れた/空のチャートは禁止
- `export-option` — データ量の多いプロダクトではチャートデータのCSV/画像エクスポートを提供
- `drill-down-consistency` — ドリルダウン操作は明確な戻りパスと階層パンくずを維持
- `time-scale-clarity` — 時系列チャートは時間粒度（日/週/月）を明確にラベル付けし、切替を許可
