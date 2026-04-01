# base_a11y

Next.js + GitHub Actions + Playwright + Jest/Vitest 環境でのアクセシビリティ担保セットアップガイド。
プロジェクト初期に入れておくべきものをレイヤーごとに整理する。

---

## Layer 1: 静的解析（書いた瞬間に検出）

### eslint-plugin-jsx-a11y

```bash
npm install -D eslint-plugin-jsx-a11y
```

`.eslintrc.json` または `eslint.config.js` に追加:

```json
{
  "extends": [
    "next/core-web-vitals",
    "plugin:jsx-a11y/recommended"
  ],
  "plugins": ["jsx-a11y"]
}
```

検出できるもの: imgのalt欠落、labelとinputの未紐付け、インタラクティブ要素への不適切なARIAなど。
エディタ（VS Code + ESLint拡張）と組み合わせると書いた瞬間に警告が出る。

### 限界

静的解析なので動的に変化するARIA属性や、実行時の状態に依存する問題は検出できない。

---

## Layer 2: コンポーネントテスト（jest-axe）

```bash
npm install -D jest-axe @types/jest-axe
```

各コンポーネントのテストファイルに追加:

```typescript
import { render } from '@testing-library/react'
import { axe, toHaveNoViolations } from 'jest-axe'
import { Button } from './Button'

expect.extend(toHaveNoViolations)

describe('Button', () => {
  it('should have no axe violations', async () => {
    const { container } = render(<Button>送信</Button>)
    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })
})
```

新規コンポーネントを作るたびにこのテストを追加するのをルールにする。
既存コンポーネントに後から追加すると大量の違反が出るので、最初から入れておく。

### 限界

コンポーネント単体のDOMを検査するため、ページ全体のランドマーク構造や見出し階層の問題は見えない。
axe-coreが検出できるのはWCAG違反の30〜40%程度という点はjest-axeでも同じ。

---

## Layer 3: E2Eテスト（Playwright + axe-core）

```bash
npm install -D @axe-core/playwright
```

`playwright.config.ts` は既存のものを使いつつ、a11yチェック用のテストを追加:

```typescript
// tests/a11y/pages.spec.ts
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

const pages = [
  { name: 'トップ', path: '/' },
  { name: 'ログイン', path: '/login' },
  { name: '検索結果', path: '/search?q=test' },
]

for (const page of pages) {
  test(`${page.name}ページ: axe違反がないこと`, async ({ page: p }) => {
    await p.goto(page.path)

    const results = await new AxeBuilder({ page: p })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa', 'wcag22aa'])
      .analyze()

    expect(results.violations).toEqual([])
  })
}
```

### キーボード操作の確認テスト

自動検出できないキーボード操作の問題を補うために、主要フローには手動確認の代わりとしてPlaywrightで操作テストを書く:

```typescript
test('モーダル: Escキーで閉じられること', async ({ page }) => {
  await page.goto('/some-page')
  await page.click('[data-testid="open-modal"]')
  await page.keyboard.press('Escape')
  await expect(page.locator('[role="dialog"]')).not.toBeVisible()
})

test('ナビゲーション: Tabキーで全リンクにフォーカスできること', async ({ page }) => {
  await page.goto('/')
  await page.keyboard.press('Tab')
  const focused = page.locator(':focus')
  await expect(focused).toBeVisible()
})
```

### 限界

axe-coreで検出できない問題（フォーカス順序の論理的な妥当性、スクリーンリーダーの読み上げ内容の質など）はPlaywrightでも検出できない。

---

## Layer 4: CI（GitHub Actions）

`.github/workflows/a11y.yml`:

```yaml
name: Accessibility

on:
  pull_request:
    branches: [main, develop]

jobs:
  jest-axe:
    name: Component a11y tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm test -- --testPathPattern=a11y

  playwright-axe:
    name: Page a11y tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npx playwright test tests/a11y/
        env:
          BASE_URL: http://localhost:3000
```

PRがマージできない状態を作ることで、「後で直す」が起きにくくなる。

---

## Layer 5: PRレビュープロセス

`.github/pull_request_template.md`:

```markdown
## 変更内容

<!-- 何を変更したか -->

## a11yチェック

UIの変更を含む場合、以下を確認してチェックを入れる。

- [ ] キーボードのみで全操作ができる
- [ ] フォーカスインジケータが視覚的に確認できる
- [ ] 画像・アイコンに適切なaltまたはaria-labelがある
- [ ] フォームのラベルとinputが紐づいている
- [ ] 色だけで情報を伝えていない
- [ ] スクリーンリーダーで主要な操作を確認した（できれば）

UIの変更がない場合: `N/A`
```

自動検出できない問題をレビュー時に拾うための最後の網。
ただしチーム全員がa11yの基礎知識を持っていないと機能しないので、チェック項目の意味を別途共有しておく。

---

## Layer 6: コンポーネント設計の標準化

自分でコンポーネントライブラリを作る場合、以下を最初から設計に入れておく。

### Buttonコンポーネントの例

```typescript
// src/components/ui/Button.tsx
type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> & {
  isLoading?: boolean
}

export function Button({ isLoading, children, disabled, ...props }: ButtonProps) {
  return (
    <button
      {...props}
      disabled={disabled || isLoading}
      aria-busy={isLoading}
      aria-disabled={disabled || isLoading}
    >
      {isLoading && <span aria-hidden="true">…</span>}
      {children}
    </button>
  )
}
```

`div` や `span` にクリックハンドラをつけない、`button` を使う、ARIAはデフォルトで適切な状態にする、という方針をベースコンポーネントに焼き込む。

### フォームコンポーネントの例

```typescript
// src/components/ui/FormField.tsx
type FormFieldProps = {
  id: string
  label: string
  error?: string
  children: React.ReactElement
}

export function FormField({ id, label, error, children }: FormFieldProps) {
  const errorId = `${id}-error`
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      {React.cloneElement(children, {
        id,
        'aria-describedby': error ? errorId : undefined,
        'aria-invalid': error ? true : undefined,
      })}
      {error && (
        <p id={errorId} role="alert">
          {error}
        </p>
      )}
    </div>
  )
}
```

labelとinputの紐付け、エラー時のaria-invalid/aria-describedby、role="alert" によるエラー通知をコンポーネントで強制する。

---

## 優先順位

時間がないとき、最低限これだけは初日に入れる順序:

1. `eslint-plugin-jsx-a11y` — 設定5分、即効性が高い
2. `jest-axe` を新規コンポーネントのテストテンプレートに含める
3. GitHub ActionsにCI追加
4. PRテンプレート追加

Playwright + axe-core は、主要ページの数が決まってきたタイミングで追加する。

---

## 自動検出できない問題への対処

以下はどのツールでも検出できないため、手動確認が必要:

- スクリーンリーダーの読み上げが意味をなしているか（VoiceOver / NVDA）
- フォーカス順序が視覚的な順序と一致しているか
- エラーメッセージが具体的で理解できるか
- 認知アクセシビリティ（複雑すぎる操作、わかりにくいラベル）

主要フローについて、月1回程度VoiceOverで実際に操作してみるのが現実的な補完手段。
