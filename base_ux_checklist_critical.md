# ux_checklist_critical

UI/UXの最重要ルール。UIを作成・編集するすべての場面で適用する。
出典: Apple HIG, Material Design, WCAG

---

## Accessibility（CRITICAL）

- `color-contrast` — テキストのコントラスト比は最低4.5:1（大文字テキストは3:1）
- `focus-states` — インタラクティブ要素に2〜4pxの可視フォーカスリングをつける
- `alt-text` — 意味のある画像にはすべて説明的なalt textをつける
- `aria-labels` — アイコンのみのボタンにはaria-label（ネイティブではaccessibilityLabel）を設定
- `keyboard-nav` — Tab順序がビジュアル順序と一致すること。完全なキーボード操作をサポート
- `form-labels` — inputにはfor属性つきのlabelを使う
- `skip-links` — キーボードユーザー向けに「メインコンテンツへスキップ」リンクを設置
- `heading-hierarchy` — h1→h6を順序通り使う。レベルの飛ばしは禁止
- `color-not-only` — 色だけで情報を伝えない。アイコンやテキストを併用する
- `dynamic-type` — システムのテキストサイズ変更をサポート。拡大時に切り捨てが起きないこと
- `reduced-motion` — `prefers-reduced-motion` を尊重し、要求時にアニメーションを抑制・無効化
- `voiceover-sr` — 意味のあるaccessibilityLabel/Hintを設定。論理的な読み上げ順序を確保
- `escape-routes` — モーダルや複数ステップのフローにキャンセル/戻るを必ず設ける
- `keyboard-shortcuts` — システムやa11yのショートカットを上書きしない。D&Dにはキーボード代替を提供

## Touch & Interaction（CRITICAL）

- `touch-target-size` — 最小44×44pt（Apple）/ 48×48dp（Material）。視覚サイズが小さい場合はヒットエリアを拡張
- `touch-spacing` — タッチターゲット間に最低8px/8dpの間隔を確保
- `hover-vs-tap` — 主要操作はclick/tapで。hoverだけに依存しない
- `loading-buttons` — 非同期処理中はボタンを無効化し、スピナーまたはプログレスを表示
- `error-feedback` — エラーメッセージは問題箇所の近くに明確に表示
- `cursor-pointer` — クリック可能な要素にcursor-pointerを設定（Web）
- `gesture-conflicts` — メインコンテンツで水平スワイプを避ける。縦スクロールを優先
- `standard-gestures` — プラットフォーム標準のジェスチャーを一貫して使用。再定義しない
- `system-gestures` — システムジェスチャー（コントロールセンター、戻るスワイプ等）をブロックしない
- `gesture-alternative` — ジェスチャーのみの操作に頼らない。重要な操作には可視コントロールを提供
- `safe-area-awareness` — 主要タッチターゲットをノッチ、Dynamic Island、ジェスチャーバー、画面端から離す
- `no-precision-required` — 小さなアイコンや細い端へのピクセル精度のタップを要求しない
