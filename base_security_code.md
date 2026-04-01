# base_security_code

このファイルに書かれたルールは、コードを生成・編集するすべての場面で常に適用すること。
ユーザーから明示的に例外を求められた場合は、理由を確認してから対応する。

---

## 適用レベルの定義

- strict: このパターンは絶対に生成しない。代替実装を提示する
- warning: 警告を添えた上で代替案を提示する
- advisory: ベストプラクティスとして言及する

---

## 1. TypeScript 設定（strict）

tsconfig.json には必ず以下を含める。

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true
  }
}
```

`any` 型の使用は禁止。型が不明な外部データには `unknown` を使い、型ガードで絞り込む。

---

## 2. 入力バリデーション（strict）

すべての外部入力（リクエストボディ・クエリパラメータ・ヘッダー・環境変数）は
エントリーポイントで必ず Zod でバリデーションする。クライアントサイドのバリデーションのみは禁止。

```typescript
// DO
import { z } from 'zod'
const UserSchema = z.object({
  email: z.string().email().max(254),
  age: z.number().int().min(0).max(150),
})
const result = UserSchema.safeParse(req.body)
if (!result.success) return res.status(400).json({ error: 'Invalid input' })
const user = result.data

// DON'T
const user = req.body as User
```

---

## 3. SQL インジェクション（strict）

文字列結合・テンプレートリテラルによるクエリ構築は生成しない。
必ずパラメータ化クエリまたは ORM の安全なメソッドを使う。

```typescript
// DO
const result = await db.query('SELECT * FROM users WHERE id = $1', [userId])
// Prisma
const user = await prisma.user.findUnique({ where: { id: userId } })

// DON'T
const query = `SELECT * FROM users WHERE id = ${userId}`
await db.query(query)
```

---

## 4. コマンドインジェクション（strict）

`exec` / `execSync` にユーザー入力を渡すのは禁止。
`spawn` / `spawnSync` を使い、引数を配列で渡す。`shell: true` は使わない。
`eval` / `new Function(string)` の生成は禁止。

```typescript
// DO
import { spawnSync } from 'child_process'
spawnSync('echo', [userInput])

// DON'T
import { exec } from 'child_process'
exec(`echo ${userInput}`)
eval(userInput)
```

---

## 5. プロトタイプ汚染（strict）

`Object.assign` / `lodash.merge` / `_.merge` に外部入力を渡すのは禁止。
JSON.parse した外部データをオブジェクトにマージする前に、`__proto__` / `constructor` / `prototype` キーを除去する。

```typescript
// DO: キーのサニタイズ
function safeMerge(target: object, source: unknown): void {
  if (typeof source !== 'object' || source === null) return
  for (const [key, value] of Object.entries(source)) {
    if (['__proto__', 'constructor', 'prototype'].includes(key)) continue
    ;(target as Record<string, unknown>)[key] = value
  }
}

// または Object.create(null) で prototype のないオブジェクトを使う
const safe = Object.create(null)

// DON'T
Object.assign(config, JSON.parse(untrustedInput))
```

---

## 6. XSS（strict）

HTML に挿入するすべての文字列は DOMPurify でサニタイズする。
`innerHTML` への直接代入は禁止。テンプレートリテラルで HTML を組み立てるのは禁止。
サーバーサイドレンダリングでは `textContent` を使うか、テンプレートエンジンの自動エスケープに頼る。

```typescript
// DO
import DOMPurify from 'dompurify'
element.innerHTML = DOMPurify.sanitize(userContent)
element.textContent = userInput

// DON'T
element.innerHTML = userInput
document.write(userInput)
```

---

## 7. パストラバーサル（strict）

ユーザー入力をファイルパスに含める場合は、必ず `path.resolve` で正規化した後、
ベースディレクトリ内に収まっているか確認する。

```typescript
// DO
import path from 'path'
const BASE = path.resolve('./uploads')
const requested = path.resolve(BASE, userFilename)
if (!requested.startsWith(BASE + path.sep)) {
  throw new Error('Path traversal detected')
}

// DON'T
const filePath = `./uploads/${req.params.filename}`
fs.readFile(filePath, ...)
```

---

## 8. 認証・パスワード（strict）

パスワードのハッシュは `argon2` を使う。`bcrypt` は可。`md5` / `sha1` / `sha256` は禁止。
比較には `crypto.timingSafeEqual` を使い、タイミング攻撃を防ぐ。

```typescript
// DO
import argon2 from 'argon2'
const hash = await argon2.hash(password)
const valid = await argon2.verify(hash, password)

// タイミングセーフな比較
import { timingSafeEqual, createHash } from 'crypto'
const a = createHash('sha256').update(tokenA).digest()
const b = createHash('sha256').update(tokenB).digest()
const match = a.length === b.length && timingSafeEqual(a, b)

// DON'T
const hash = require('crypto').createHash('md5').update(password).digest('hex')
if (storedToken === inputToken) { ... }  // タイミング攻撃に脆弱
```

---

## 9. JWT（strict）

`alg: "none"` を受け入れる設定は禁止。アルゴリズムを明示的に指定して検証する。
シークレットは環境変数から取得し、コードにハードコードしない。

```typescript
// DO
import jwt from 'jsonwebtoken'
const decoded = jwt.verify(token, process.env.JWT_SECRET!, {
  algorithms: ['HS256'],
})

// DON'T
const decoded = jwt.decode(token)  // 署名検証なし
jwt.verify(token, secret)  // algorithms 未指定（alg:none 攻撃に脆弱）
```

---

## 10. シークレット管理（strict）

API キー・パスワード・接続文字列をコードにハードコードするのは禁止。
すべて環境変数から取得し、envalid などで起動時に検証する。

```typescript
// DO
import { cleanEnv, str } from 'envalid'
const env = cleanEnv(process.env, {
  DATABASE_URL: str(),
  JWT_SECRET: str(),
})

// DON'T
const secret = 'hardcoded-secret-key'
const dbUrl = 'postgresql://user:pass@localhost/db'
```

---

## 11. HTTP セキュリティヘッダー（warning）

Express / Fastify には必ず `helmet` を適用する。
CORS は許可するオリジンを明示的に列挙し、`origin: '*'` は禁止。

```typescript
// DO
import helmet from 'helmet'
import cors from 'cors'
app.use(helmet())
app.use(cors({ origin: ['https://example.com'] }))

// DON'T
app.use(cors())  // 全オリジン許可
```

---

## 12. レート制限（warning）

認証エンドポイントには必ずレート制限を設ける。

```typescript
// DO
import rateLimit from 'express-rate-limit'
app.use('/auth', rateLimit({ windowMs: 15 * 60 * 1000, max: 20 }))
```

---

## 13. エラーハンドリング（warning）

スタックトレース・内部パス・DBエラーをクライアントに返すのは禁止。
エラーログはサーバーサイドにのみ記録し、クライアントには汎用メッセージを返す。

```typescript
// DO
app.use((err: Error, req: Request, res: Response, _next: NextFunction) => {
  console.error(err)  // サーバーサイドにのみ記録
  res.status(500).json({ error: 'Internal server error' })
})

// DON'T
res.status(500).json({ error: err.message, stack: err.stack })
```

---

## 14. 依存関係（advisory）

- 新しいパッケージを追加する前に `npm audit` を実行して確認を得る
- `npm install` / `npm update` 前後に差分を提示する
- lockfile（package-lock.json / yarn.lock）はコミットする
- `postinstall` スクリプトを含むパッケージを追加する際は中身を提示してユーザーの承認を得る

パッケージの追加・更新・緊急対応の詳細手順は `base_security_npm.md` を参照すること。

---

## 15. セキュリティレビューの実施

以下のタイミングで `/security-review` を実行してユーザーに報告する。

- 認証・認可に関わるコードを書いたとき
- 外部入力をDBや shell に渡す処理を書いたとき
- 新しい API エンドポイントを追加したとき
- 依存パッケージを追加・更新したとき

`/security-review` はあくまで補助。Semgrep などの SAST ツールとの併用を推奨として伝える。

---

## 禁止パターン一覧（即時拒否）

以下のコードは生成しない。代替を提示すること。

```
eval(...)
new Function(string)
exec(`...${userInput}...`)
innerHTML = userInput
Object.assign(target, JSON.parse(untrustedInput))
alg: 'none'
md5 / sha1 でのパスワードハッシュ
const secret = '...'  // ハードコードされた認証情報
`SELECT ... WHERE id = ${userInput}`  // 文字列結合クエリ
```
