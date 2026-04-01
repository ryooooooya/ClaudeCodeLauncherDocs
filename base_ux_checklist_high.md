# ux_checklist_high

UIの構造・レイアウト・パフォーマンスに関するルール。
UI実装時やレビュー時に参照する。
出典: Apple HIG, Material Design, WCAG, Core Web Vitals

---

## Performance（HIGH）

- `image-optimization` — WebP/AVIFを使用。レスポンシブ画像（srcset/sizes）、非重要アセットは遅延読み込み
- `image-dimension` — width/heightまたはaspect-ratioを宣言してレイアウトシフトを防止（CLS対策）
- `font-loading` — `font-display: swap/optional` でFOITを回避。レイアウトシフト軽減のためスペースを予約
- `font-preload` — クリティカルなフォントのみpreload。全バリアントのpreloadは避ける
- `critical-css` — ファーストビューのCSSを優先（インラインまたは早期読み込み）
- `lazy-loading` — 非ヒーローコンポーネントはdynamic import/ルートレベル分割で遅延読み込み
- `bundle-splitting` — ルート/機能単位でコード分割して初期読み込みとTTIを削減
- `third-party-scripts` — サードパーティスクリプトはasync/deferで読み込み。不要なものは削除
- `reduce-reflows` — レイアウトの読み書きを頻繁に交互に行わない。DOM読み取りをまとめてから書き込む
- `content-jumping` — 非同期コンテンツのスペースを事前に確保してレイアウトジャンプを防止（CLS対策）
- `lazy-load-below-fold` — ファーストビュー外の画像や重いメディアに`loading="lazy"`を使用
- `virtualize-lists` — 50件以上のリストは仮想化してメモリ効率とスクロール性能を改善
- `main-thread-budget` — フレームあたりの処理を約16ms以下に。重い処理はメインスレッドから分離
- `progressive-loading` — 1秒以上の操作にはスケルトンスクリーン/シマーを使用。長いスピナーは避ける
- `input-latency` — タップ/スクロールの入力遅延を約100ms以下に保つ
- `tap-feedback-speed` — タップから100ms以内に視覚フィードバックを提供
- `debounce-throttle` — scroll、resize、inputなど高頻度イベントにはdebounce/throttleを使用
- `offline-support` — オフライン状態のメッセージングと基本的なフォールバックを提供（PWA/モバイル）
- `network-fallback` — 低速ネットワーク向けに劣化モードを提供（低解像度画像、アニメーション削減）

## Style Selection（HIGH）

- `style-match` — スタイルをプロダクトタイプに合わせる
- `consistency` — 全ページで同一スタイルを使用
- `no-emoji-icons` — SVGアイコン（Heroicons、Lucide等）を使用。絵文字をアイコンとして使わない
- `color-palette-from-product` — プロダクト/業界からパレットを選択
- `effects-match-style` — シャドウ、ブラー、角丸を選択したスタイルに合わせる
- `platform-adaptive` — プラットフォームの慣習を尊重（iOS HIG vs Material）
- `state-clarity` — hover/pressed/disabledの状態を視覚的に区別しつつスタイルに合わせる
- `elevation-consistent` — カード、シート、モーダルに一貫したelevation/shadowスケールを使用
- `dark-mode-pairing` — ライト/ダークバリアントを一緒に設計してブランドとコントラストの一貫性を保つ
- `icon-style-consistent` — 1つのアイコンセット/ビジュアル言語を製品全体で使用
- `system-controls` — ブランド要件がない限りネイティブ/システムコントロールを優先
- `blur-purpose` — ブラーはモーダル等の背景を示すために使用。装飾には使わない
- `primary-action` — 各画面にプライマリCTAは1つだけ。セカンダリは視覚的に控えめに

## Layout & Responsive（HIGH）

- `viewport-meta` — `width=device-width initial-scale=1`（ズーム無効化は禁止）
- `mobile-first` — モバイルファーストで設計し、タブレット→デスクトップへスケールアップ
- `breakpoint-consistency` — 体系的なブレークポイントを使用（例: 375 / 768 / 1024 / 1440）
- `readable-font-size` — モバイルの本文テキストは最低16px（iOS自動ズーム回避）
- `line-length-control` — モバイル35〜60文字/行、デスクトップ60〜75文字/行
- `horizontal-scroll` — モバイルで水平スクロールを発生させない
- `spacing-scale` — 4pt/8dp刻みのスペーシングシステムを使用
- `touch-density` — コンポーネント間隔をタッチ操作に適した密度に保つ
- `container-width` — デスクトップで一貫したmax-width（max-w-6xl / 7xl）
- `z-index-management` — 階層化されたz-indexスケールを定義（例: 0 / 10 / 20 / 40 / 100 / 1000）
- `fixed-element-offset` — 固定ナビバー/ボトムバーの下にコンテンツ用のsafe paddingを確保
- `scroll-behavior` — メインスクロールを妨げるネストされたスクロール領域を避ける
- `viewport-units` — モバイルでは100vhより`min-h-dvh`を優先
- `orientation-support` — ランドスケープモードでもレイアウトが読みやすく操作可能であること
- `content-priority` — モバイルではコアコンテンツを最初に表示。セカンダリは折り畳みまたは非表示
- `visual-hierarchy` — サイズ、スペーシング、コントラストで階層を確立。色だけに頼らない

## Navigation Patterns（HIGH）

- `bottom-nav-limit` — ボトムナビゲーションは最大5項目。アイコンとラベルを両方表示
- `drawer-usage` — ドロワー/サイドバーはセカンダリナビゲーション用。プライマリアクションには使わない
- `back-behavior` — 戻るナビゲーションは予測可能で一貫していること。スクロール/状態を保持
- `deep-linking` — 主要画面はすべてディープリンク/URLで到達可能にする
- `tab-bar-ios` — iOS: トップレベルナビゲーションにはボトムタブバーを使用
- `top-app-bar-android` — Android: トップアプリバーとナビゲーションアイコンで構造化
- `nav-label-icon` — ナビゲーション項目にはアイコンとテキストラベルの両方を表示
- `nav-state-active` — 現在地をナビゲーション内で視覚的にハイライト（色、太さ、インジケーター）
- `nav-hierarchy` — プライマリナビ（タブ/ボトムバー）とセカンダリナビ（ドロワー/設定）を明確に分離
- `modal-escape` — モーダル/シートには明確な閉じる/解除のアフォーダンスを提供。モバイルでは下スワイプ
- `search-accessible` — 検索は簡単に到達可能にする（トップバーまたはタブ）。最近/おすすめクエリを提供
- `breadcrumb-web` — Web: 3階層以上のネストにはパンくずリストを使用
- `state-preservation` — 戻る操作でスクロール位置、フィルター状態、入力を復元
- `gesture-nav-support` — システムジェスチャーナビゲーション（iOS戻るスワイプ、Android予測戻る）と競合しない
- `tab-badge` — ナビ項目のバッジは控えめに使用。ユーザーが訪問したらクリア
- `overflow-menu` — アクションがスペースを超える場合、詰め込まずオーバーフロー/もっとメニューを使用
- `bottom-nav-top-level` — ボトムナビはトップレベル画面専用。サブナビゲーションをネストしない
- `adaptive-navigation` — 大画面（1024px以上）はサイドバー、小画面はボトム/トップナビを優先
- `back-stack-integrity` — ナビゲーションスタックを暗黙にリセットしたり、予期せずホームに飛ばない
- `navigation-consistency` — ナビゲーション配置は全ページで同一にする
- `avoid-mixed-patterns` — 同一階層でタブ + サイドバー + ボトムナビを混在させない
- `modal-vs-navigation` — モーダルをプライマリナビゲーションフローに使わない
- `focus-on-route-change` — ページ遷移後、スクリーンリーダーユーザーのためにメインコンテンツ領域にフォーカスを移動
- `persistent-nav` — コアナビゲーションは深い階層からも到達可能であること。サブフローで完全に隠さない
- `destructive-nav-separation` — 危険なアクション（アカウント削除、ログアウト）は通常のナビ項目から視覚的・空間的に分離
- `empty-nav-state` — ナビゲーション先が利用不可の場合、静かに隠すのではなく理由を説明
