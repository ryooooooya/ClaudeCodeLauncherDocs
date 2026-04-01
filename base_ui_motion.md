# base_ui_motion

UIの触感・質感に関するルール。アニメーション、インタラクションフィードバック、ジェスチャー応答を対象とする。
アニメーションを実装・レビューするときに参照する。
出典: Emil Kowalski（animations.dev, Sonner, Vaul）, Apple HIG, Material Design

---

## 原則

アニメーションは「見せるもの」ではなく「感じさせるもの」。うまくいっているとき、ユーザーはアニメーションに気づかず、インターフェースが気持ちいいと感じる。アニメーションが目立ったら、やりすぎのサイン。

- すべてのアニメーションは因果関係を表現する。装飾だけのアニメーションは禁止
- ユーザーを待たせるためにアニメーションを使わない。遅いコードをアニメーションで隠さない
- `prefers-reduced-motion` は常に尊重する（CRITICAL と同等）

---

## Timing & Duration

- `duration-micro` — マイクロインタラクション（hover, toggle, press）は150〜250ms
- `duration-standard` — モーダル開閉、パネルスライド、コンテンツ表示は200〜350ms
- `duration-complex` — ページ遷移など複雑なオーケストレーションは400〜600ms。1秒超は禁止
- `exit-faster-than-enter` — 退出アニメーションは入場の60〜70%の長さにして応答感を出す
- `stagger-sequence` — リスト/グリッドアイテムの入場を1項目あたり30〜60msずらす。100ms超はスライドショーに見える
- `stagger-group-size` — staggerグループは3〜7アイテムまで。それ以上は最後のアイテムの待ち時間が長くなりすぎる
- `excessive-motion` — ビューごとにアニメーションするキー要素は1〜2個まで。同時に動くものを増やさない

---

## Easing & Physics

- `easing-enter` — 入場はease-out（自然な減速）。`cubic-bezier(0.22, 1, 0.36, 1)` が汎用的
- `easing-exit` — 退場はease-in（自然な加速）。入場と同じ曲線を逆にしない
- `easing-reposition` — 画面内で位置を移動する要素にはease-in-out
- `no-linear` — UIトランジションにlinearは使わない。プログレスバーとループアニメーション専用
- `spring-physics` — 自然な感触のためにスプリング/物理ベースの曲線を優先。CSS変数として定義する

```css
:root {
  --ease-spring-bounce: cubic-bezier(0.34, 1.56, 0.64, 1);
  --ease-spring-smooth: cubic-bezier(0.22, 1, 0.36, 1);
  --ease-spring-snappy: cubic-bezier(0.16, 1, 0.3, 1);
}
```

- `easing-consistency` — 関連して動く複数の要素には同一の曲線を使う（モーダルと背景オーバーレイ等）
- `motion-consistency` — duration/easingトークンをグローバルに統一。プロジェクト全体で同じリズムを共有

---

## What to Animate

- `transform-only` — transform と opacity のみアニメーション。width/height/top/left/margin/padding は禁止（レイアウトスラッシング）
- `layout-shift-avoid` — アニメーションがレイアウトリフローやCLSを起こさないこと。位置変更には transform を使用
- `will-change` — アニメーション開始前に付与し、終了後に必ず外す。常時 on は禁止
- `mobile-no-layout` — モバイルブラウザはレイアウトアニメーションが苦手。transform と opacity に限定する
- `no-scroll-animation` — スクロール連動アニメーションはジャンクの原因になる。scroll-timeline や Intersection Observer は控えめに

---

## Entrance & Exit Patterns

- `fade-rise` — コンテンツ出現の基本パターン: `opacity: 0, y: 8px` → `opacity: 1, y: 0`
- `movement-distance` — 移動量は小さく保つ。マイクロインタラクションは4〜16px、大きな出現は20〜40px。それ以上はアニメーションでなくカートゥーンになる
- `scale-origin` — ドロップダウンはトリガー位置からscale（transform-originをトリガーに合わせる）。モーダルは中央からscale
- `modal-motion` — モーダル/シート: `scale(0.96) + opacity` で開く。`scale(0)` スタートは安っぽく見える
- `toast-motion` — トースト: 出現する端からスライド+フェード。退場は出現した方向に戻す
- `fade-crossfade` — 同一コンテナ内のコンテンツ置換にはクロスフェード。スナップ切替は禁止
- `opacity-threshold` — フェードする要素はopacity 0.2以下に留まらない。完全にフェードするか可視のまま
- `continuity` — ページ/画面遷移は空間的な連続性を維持（共有要素トランジション、方向スライド）
- `navigation-direction` — 前進ナビゲーションは左/上へ、後退は右/下へ。方向を論理的に一貫させる
- `hierarchy-motion` — 階層の深さを方向で表現。下から入場=深い階層へ進む、上へ退場=戻る
- `shared-element-transition` — 画面間の視覚的連続性のために共有要素/ヒーロートランジションを使用

---

## Interaction Feedback（触感・質感）

- `press-feedback` — タップ/クリック時の視覚フィードバック（リップル/ハイライト/スケール）を提供
- `scale-feedback` — タップ可能なカード・ボタンのプレス時に微妙なスケール（0.97〜0.98）。0.95以下はカートゥーン
- `hover-instant-on` — hoverは即座に反応（transition-duration: 0ms）。離脱時だけ150msでease-out
- `gesture-feedback` — ドラッグ、スワイプ、ピンチは指の動きをリアルタイムで追跡する視覚応答を提供
- `haptic-feedback` — 確認や重要なアクションにハプティクを使用。多用しない
- `tap-delay` — `touch-action: manipulation` で300ms遅延を解消（Web）
- `swipe-clarity` — スワイプ操作には明確なアフォーダンスまたはヒント（シェブロン、ラベル等）を表示
- `drag-threshold` — 誤ドラッグを防ぐため、ドラッグ開始前に移動閾値を設ける
- `disabled-no-animation` — 無効要素にはhoverエフェクトもアニメーションも付けない。無効は無効
- `loading-pulse` — ローディング状態にはスケルトンシマーまたはsubtleなパルス。スピナーは最終手段
- `state-transition` — 状態変化（hover/active/expanded/collapsed）はスナップせずスムーズにアニメーション

---

## Orchestration

- `stagger-order` — staggerシーケンスはメインコンテンツを先に動かす。背景→コンテナ→コンテンツ→アクションの順
- `stagger-exit` — 退場はstaggerを逆順にするか、一括で動かす。どちらかに統一する
- `interruptible` — アニメーションは中断可能であること。ユーザーの操作で進行中のアニメーションを即座にキャンセルできる
- `no-blocking-animation` — アニメーション中にユーザー入力をブロックしない。UIはインタラクティブであり続ける

---

## When NOT to Animate

以下の場面ではアニメーションを付けない。

- フォームバリデーションエラー（色・アイコンの変化で十分）
- 重大なエラー状態（悪い知らせをアニメーションで遅らせない）
- ユーザーが能動的に読んでいるコンテンツ
- ライブデータ、タイマーなど高頻度の更新
- ユーザーがセッション中に何百回も目にする要素

---

## Framer Motion パターン（Next.js / React）

### フェード+ライズ（基本）

```tsx
<motion.div
  initial={{ opacity: 0, y: 8 }}
  animate={{ opacity: 1, y: 0 }}
  exit={{ opacity: 0, y: 4 }}
  transition={{ duration: 0.2, ease: [0.22, 1, 0.36, 1] }}
/>
```

### スプリング（ボタン/カード）

```tsx
<motion.div
  whileTap={{ scale: 0.97 }}
  transition={{ type: "spring", stiffness: 400, damping: 25 }}
/>
```

### stagger（リスト）

```tsx
<motion.ul
  initial="hidden"
  animate="visible"
  variants={{
    visible: { transition: { staggerChildren: 0.04 } },
  }}
>
  {items.map(item => (
    <motion.li
      key={item.id}
      variants={{
        hidden: { opacity: 0, y: 8 },
        visible: { opacity: 1, y: 0 },
      }}
    />
  ))}
</motion.ul>
```

### AnimatePresence（exit対応）

```tsx
<AnimatePresence mode="wait">
  <motion.div
    key={currentView}
    initial={{ opacity: 0 }}
    animate={{ opacity: 1 }}
    exit={{ opacity: 0 }}
    transition={{ duration: 0.15 }}
  />
</AnimatePresence>
```

### reduced-motion（CSS）

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## よくあるミス

- bouncy everything — バウンスは成功・祝福のフィードバック専用。メニューやモーダルに使わない
- slow fades — opacity変化が200ms超えると「ラグ」に見える
- scale(0)スタート — 無から湧いて出るように見える。0.95以上からスタートする
- 方向の不統一 — モーダルが下から入るなら下へ退場する。方向を決めたら全体で一貫させる
- exitアニメーションの省略 — 突然消えるのは不自然。入場と同じだけ退場を考える
- 同時に動かしすぎ — フォーカルアニメーションは1つ。他は静止か控えめにする
- マウント時の無条件アニメーション — 初回ロードはあり得る。再レンダリングのたびに動かさない
