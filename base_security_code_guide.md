# base_security_code_guide

Claude Code が書くコードに脆弱性を混入させないための対策。
Layer 1（Claude Code 自体の制御）とは独立した問題。

> Endor Labs の調査によると、AI が生成したコードは25〜75%の確率で脆弱性を含む。
> モデルはセキュアでないコードを大量に学習しているため、指示なしでは同じパターンを再現する。

---

## セットアップ方法

`CLAUDE_SECURE_CODING_TS.md` を `CLAUDE.md` に組み込むか、
セッション冒頭で読ませることでルールが適用される。

```
> CLAUDE_SECURE_CODING_TS.md を読んでセキュアコーディングルールを適用してください
```

---

## Layer 1 との違い

| | Layer 1 | Layer 2 |
|--|---------|---------|
| 何を守るか | Claude Code の実行環境 | Claude Code が書くコード |
| 脅威 | プロンプトインジェクション・権限悪用 | SQL インジェクション・XSS・認証不備 など |
| 対策の場所 | settings.json・フック・.claudeignore | CLAUDE.md のコーディングルール・SAST |
| 効果が出るタイミング | Claude Code の起動時 | コード生成時 |

---

## OWASP Top 10 2025 とのマッピング

| OWASP カテゴリ | TypeScript / Node.js での主な脅威 | 対応するルール |
|--------------|----------------------------------|--------------|
| A01 アクセス制御の不備 | RBAC の抜け漏れ・認可チェックの欠落 | JWT・認証セクション |
| A02 暗号化の失敗 | MD5/SHA1 でのパスワードハッシュ・平文保存 | パスワードハッシュ |
| A03 インジェクション | SQL インジェクション・コマンドインジェクション・XSS | セクション 3・4・6 |
| A04 安全でない設計 | バリデーション欠落・信頼境界の未定義 | セクション 2 |
| A05 セキュリティの設定ミス | helmet 未適用・CORS `*`・エラー情報漏洩 | セクション 11・13 |
| A06 脆弱なコンポーネント | npm 脆弱パッケージ・lockfile なし | セクション 14 |
| A07 認証の失敗 | JWT の alg:none・タイミング攻撃 | セクション 8・9 |
| A09 ログ・監視の失敗 | スタックトレースの公開・機密情報のログ出力 | セクション 13 |

---

## 各ルールの解説

### 1. TypeScript strict モード

`strict: true` を設定しないと、`null` / `undefined` の扱いやインデックスアクセスが型チェックされず、
実行時エラーや予期しない型変換が起きやすくなる。
特に `noUncheckedIndexedAccess` は配列・オブジェクトアクセスで `undefined` を考慮させるため、
インジェクション対策の前提として重要。

### 2. 入力バリデーション（Zod）

すべての脆弱性の根本原因は「外部入力を信頼すること」。
TypeScript の型は実行時には消えるため、`req.body as User` は何の保護にもならない。
Zod の `safeParse` でエントリーポイントで確実に検証することで、
後続の処理全体を型安全に保てる。

### 3. SQL インジェクション

文字列結合でクエリを組み立てると、ユーザー入力が SQL として解釈される。

```
入力: ' OR '1'='1
クエリ: SELECT * FROM users WHERE id = '' OR '1'='1'  → 全件取得
```

パラメータ化クエリを使うと、入力は常に「値」として扱われ SQL として解釈されない。

### 4. コマンドインジェクション

`exec` は文字列をシェルに渡すため、`;` や `&&` で追加コマンドを注入できる。
`spawn` に配列で引数を渡すとシェルを経由しないため安全。

```
exec(`echo ${input}`)  → input が "hello; rm -rf /" でも実行される
spawnSync('echo', [input])  → input はエコーされるだけ
```

`eval` と `new Function(string)` はユーザー入力を JavaScript として実行するため、
RCE（リモートコード実行）に直結する。

### 5. プロトタイプ汚染（Prototype Pollution）

JavaScript のすべてのオブジェクトは `Object.prototype` を継承している。
`__proto__` キーを含む JSON を unsafe なマージ関数に渡すと、
グローバルなプロトタイプが汚染され、アプリ全体の挙動が変わる。

```javascript
// 攻撃リクエスト
POST /api/settings
{ "__proto__": { "isAdmin": true } }

// 汚染後
const user = {}
user.isAdmin  // true（本来は undefined）
```

Node.js では `__proto__` 汚染からさらに `NODE_OPTIONS` 環境変数経由で RCE に発展した事例もある。
外部 JSON をオブジェクトにマージする際は必ずキーのサニタイズを挟む。

### 6. XSS（クロスサイトスクリプティング）

`innerHTML` にユーザー入力を渡すと、`<script>` タグや `onerror` などが実行される。
`textContent` は文字列として挿入するため安全。
HTML として挿入が必要な場合は DOMPurify でサニタイズする。

CSP（Content Security Policy）ヘッダーを設定しておくと、
XSS が成功してもスクリプトの外部送信を防ぐ多層防御になる（helmet で設定可能）。

### 7. パストラバーサル

`../` を含むパスをそのまま使うと、想定外のディレクトリのファイルを読み書きできる。

```
GET /files?name=../../etc/passwd
→ ./uploads/../../etc/passwd が読まれる
```

`path.resolve` で正規化した後、ベースディレクトリ内に収まっているか確認する。

### 8・9. 認証・JWT

MD5・SHA1 はレインボーテーブル攻撃で破られる。argon2 は計算コストが調整可能で現時点で最も安全。

JWT の `alg: "none"` は「署名なし」を意味する。
`jwt.verify` でアルゴリズムを明示しないと、攻撃者が `alg: "none"` を指定したトークンを受け入れてしまう。

文字列の `===` 比較はタイミング攻撃に脆弱（一致する文字数が多いほど比較時間が長くなる）。
`crypto.timingSafeEqual` は常に一定時間で比較するため安全。

### 10. シークレット管理

ハードコードされた認証情報は git 履歴に残り、公開リポジトリになった瞬間に漏洩する。
環境変数を起動時に `envalid` で検証すると、設定漏れを早期に検出できる。

### 11〜13. HTTP セキュリティ・レート制限・エラーハンドリング

`helmet` は10以上のセキュリティヘッダーを一括で設定する。
スタックトレースをレスポンスに含めると、攻撃者にフレームワーク・パス・コード構造を教えることになる。

---

## AI 生成コードに特有のリスク

### ハルシネーションされた依存パッケージ

Claude が実在しない npm パッケージ名を使うことがある。
攻撃者がそのパッケージ名で悪意あるパッケージを公開した場合、インストール時に攻撃が成立する（スクワッティング攻撃）。
新しいパッケージを追加する際は必ず npm のページで実在確認をすること。

### 古いパターンの再現

Claude は古い・脆弱なパターンを学習しているため、指示なしでは MD5 を使ったり、
`exec` に文字列を渡したりするコードを生成することがある。
`CLAUDE_SECURE_CODING_TS.md` を CLAUDE.md に組み込むことでこれを防ぐ。

### `/security-review` の限界

Claude Code の `/security-review` は一次スクリーニングとして有用だが、
ビジネスロジックの認可チェック漏れ・複数処理をまたいだデータフローの問題は見落とすことがある。
Semgrep（SAST）や OWASP ZAP（DAST）との組み合わせが推奨される。

---

## 推奨ツールスタック

| カテゴリ | ツール | 用途 |
|----------|--------|------|
| スキーマバリデーション | Zod | 入力検証 |
| パスワードハッシュ | argon2 | 認証 |
| HTTP ヘッダー | helmet | セキュリティヘッダー |
| XSS サニタイズ | DOMPurify | フロントエンド |
| 環境変数検証 | envalid | シークレット管理 |
| SAST | Semgrep | 静的解析 |
| 依存関係監査 | npm audit / Snyk | 脆弱パッケージ検出 |
| DAST | OWASP ZAP | 動的解析 |

---

## CI/CD への組み込み

PR 時に自動で以下を走らせると、生成コードの問題を早期に検出できる。

```yaml
# .github/workflows/security.yml
- name: npm audit
  run: npm audit --audit-level=high

- name: Semgrep
  uses: semgrep/semgrep-action@v1
  with:
    config: p/owasp-top-ten p/typescript
```

---

## 参考

- [OWASP Top 10 2025](https://owasp.org/Top10/)
- [Node.js 公式セキュリティベストプラクティス](https://nodejs.org/en/learn/getting-started/security-best-practices)
- [TikiTribe/claude-secure-coding-rules](https://github.com/TikiTribe/claude-secure-coding-rules)
- [joacod/secure-node-typescript skill](https://playbooks.com/skills/joacod/skills/secure-node-typescript)
- [awesome-nodejs-security](https://github.com/lirantal/awesome-nodejs-security)
- [PortSwigger: Prototype Pollution](https://portswigger.net/web-security/prototype-pollution)
