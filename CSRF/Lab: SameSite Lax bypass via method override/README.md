
### **Step 1: ログインしてメール変更をキャプチャ**

1. Burp Suite起動、ブラウザのプロキシをBurpに向ける
2. LabのURLにアクセスして `wiener:peter` でログイン
3. My Accountページでメールアドレスを適当に変更（例：`test@test.com`）
4. Burp の **Proxy > HTTP history** を開く

---

### **Step 2: リクエストを確認**

HTTP historyで `POST /my-account/change-email` を見つけて確認：

```
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=xxxxx

email=test%40test.com
```

- CSRFトークンがない ✓
- Set-CookieにSameSite指定なし（=Laxデフォルト）✓

---

### **Step 3: メソッドオーバーライドを確認**

そのリクエストを **右クリック → Send to Repeater**

Repeaterで右クリック → **Change request method** でGETに変換：

```
GET /my-account/change-email?email=test%40test.com HTTP/1.1
```

Sendすると → **405 Method Not Allowed** が返ってくる

次に `_method=POST` を追加：

```
GET /my-account/change-email?email=test%40test.com&_method=POST HTTP/1.1
```

Sendすると → **302リダイレクト（成功）** が返ってくる ✓

---

### **Step 4: ブラウザで確認**

My Accountページをリロードしてメールアドレスが変わってるか確認 ✓

---

### **Step 5: Exploit Serverでペイロード作成**

Labページのトップにある **Go to exploit server** をクリック

**Body** 欄に以下を入力：

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=pwned@evil.com&_method=POST";
</script>
```

⚠️ `YOUR-LAB-ID` は自分のLabのIDに置き換える

---

### **Step 6: 自分でテスト**

**Store** → **View exploit** をクリック

My Accountページでメールが `pwned@evil.com` に変わってたらOK ✓

---

### **Step 7: Victimに届ける**

メールアドレスを自分のものと被らないように変更（例：`hacked@evil.com`）

**Deliver to victim** をクリック → Lab完了！ 🎉


---

# 🔥 どーいう脆弱性？
このラボは **CSRFトークンが存在しない or 検証されていない** うえに、  
**HTTPメソッドオーバーライド（`_method=POST`）がCSRF保護なしで通る** という複合的な欠陥。

つまり：

- 本来 POST でしか実行できない「メール変更」が  
- GET リクエストに `_method=POST` を付けるだけで  
- **CSRFトークンなしで実行できてしまう**

＝ **完全にCSRF防御が無効化されている状態**。

---

# 🔍 どう気づいた？

1. wiener でメール変更 → Burp でリクエスト確認  
2. **CSRFトークンが存在しない** と判明  
3. Repeaterで **POST → GET に変換**  
4. GETだと405 → `_method=POST` を付けると **302で成功**  
5. My Account をリロードするとメールが変わっている

ここで確信できる：

> 「このアプリは `_method=POST` を信じて状態変更を許可している」  
> 「しかもCSRFトークンがないので、誰でも勝手に実行できる」

---

# 💥 どう攻撃した？
攻撃はめちゃシンプル。

1. 攻撃者は exploit server に以下の JS を置く：

```html
<script>
  document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=hacked@evil.com&_method=POST";
</script>
```

2. 被害者がこのページを踏む  
3. 被害者ブラウザが **GET リクエスト** を送る  
4. `_method=POST` によりサーバーは **POST として処理**  
5. 被害者の Cookie（session）が自動送信される  
6. **被害者のメールアドレスが攻撃者の指定した値に変更される**

つまり：

> 「GET で勝手に POST を実行できる」  
> ＋  
> 「CSRFトークンがない」

これだけで完全に乗っ取れる。

---

# 🛡️ どう対策する？
対策は明確で、どれか1つ欠けてもダメ。

### ✔ CSRFトークンを必須にする  
- POST/PUT/DELETE には **毎回CSRFトークンを検証**

### ✔ メソッドオーバーライドを無効化 or 制限  
- `_method=POST` を GET で受け付けない  
- 受け付けるなら **CSRF検証必須**

### ✔ GET で状態変更を絶対にしない  
- GET は「参照専用」  
- 状態変更は POST/PUT/DELETE のみ

### ✔ SameSite=Lax/Strict Cookie  
- クロスサイトからの自動送信を防ぐ

---

# 🎤 **30秒面接用まとめ（和志のスタイルで）**

> **この脆弱性は、CSRFトークンが存在しないうえに、GETリクエストに `_method=POST` を付けるだけで状態変更が実行できてしまう設計ミスです。**  
> BurpでPOSTをGETに変換し、`_method=POST` を付けるとメール変更が成功したため気づきました。  
> 攻撃は、攻撃者が用意したページを踏ませるだけで、被害者のブラウザが自動的にメール変更リクエストを送ってしまいます。  
> 対策は、CSRFトークンの導入、メソッドオーバーライドの制限、そしてGETで状態変更を行わないことです。

---


