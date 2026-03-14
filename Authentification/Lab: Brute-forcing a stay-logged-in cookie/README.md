

# ✅ **Stay‑logged‑in Cookie ブルートフォース攻撃：完全手順書**

---

# **1. 自分のアカウントで Cookie の構造を確認**
### ① Burp を起動した状態でログイン  
- ユーザー名：`wiener`  
- パスワード：`peter`  
- **「Stay logged in」チェックを ON**

### ② Cookie を確認  
HTTP history で
`POST /login` のレスポンスにある：

```
stay-logged-in=xxxxxx
```

### ③ Base64 デコード  
Inspector → Decode → Base64 decode  
結果：

```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

→ これは **MD5(password)** であることを確認。

---

# **2. Cookie の構造を理解**
```
stay-logged-in = base64(username + ":" + md5(password))
```

---

# **3. Intruder に攻撃準備**
### ① ログアウトする  
`/logout` を実行。

### ② `/my-account?id=wiener` のリクエストを Intruder に送る  
Burp → Proxy → HTTP history  
`GET /my-account?id=wiener` を右クリック → **Send to Intruder**

### ③ Intruder の Positions  
- `stay-logged-in` Cookie が自動で選択されていることを確認  
- Attack type：**Sniper**

---

# **4. Payload 設定（まず自分のパスワードでテスト）**
### ① Payloads → Payload set  
- **自分のパスワード `peter` を 1 行だけ追加**

### ② Payload Processing  
順番に以下を追加：

1. **Hash → MD5**
2. **Add prefix → `wiener:`**
3. **Encode → Base64**

---

# **5. 成功判定の設定**
### Grep - Match  
- Add → `Update email`

（この文字列がレスポンスにあれば成功）

---

# **6. 攻撃開始（自分の Cookie が生成できるか確認）**
Start attack  
→ `Update email` がヒットするはず  
→ Payload 処理が正しく動いていることを確認

---

# **7. Carlos の Cookie をブルートフォース**
ここから本番。

### ① Payload リストを置き換える  
- 自分のパスワード `peter` を削除  
- **提供されたパスワード候補リストを追加**

### ② リクエスト URL を変更  
`GET /my-account?id=wiener`  
　↓  
`GET /my-account?id=carlos`

### ③ Payload Processing の prefix を変更  
- `wiener:`  
　↓  
- `carlos:`

---

# **8. 攻撃開始**
Start attack

### 成功したリクエストは 1 件だけ  
- `Update email` を含むレスポンス  
- そのリクエストの Payload が **carlos の有効 Cookie**

→ **ラボ解決**

---


# 🔥 このラボの本質（超重要ポイント）
## 🎯 **脆弱性の正体**
### **「弱い永続ログイン Cookie（Stay‑logged‑in Cookie）の設計不備」**
- Cookie が **`base64(username:md5(password))`** という **予測可能な構造**  
- MD5 は高速でブルートフォースに弱い  
- サーバー側で Cookie の署名や秘密鍵による検証がない  
- つまり **攻撃者が Cookie を自作できてしまう**

---

# 🧨 どんな攻撃か？
## ✔ **パスワードを直接攻撃するのではなく、Cookie を生成してブルートフォースする攻撃**
攻撃者は以下を行う：

1. 自分の Cookie を観察し、構造を理解する  
2. `username:md5(password)` を Base64 にしただけだと気づく  
3. 被害者（carlos）のパスワード候補を MD5 化  
4. `carlos:<md5>` を Base64 化して Cookie を生成  
5. その Cookie を使って `/my-account?id=carlos` にアクセス  
6. 成功した Cookie だけが「Update email」ボタンを返す  
7. → **carlos のアカウントにログイン成功**

つまり **パスワードを当てるのではなく、Cookie を当てる** という発想。

---

# 🧩 何が問題なのか？
## ❌ **1. Cookie が秘密情報で署名されていない**
本来、永続ログイン Cookie は：

- サーバー側で生成したランダムトークン  
- DB に保存  
- 署名付き（HMAC）  
- 有効期限付き  
- 1 回使ったら無効化（ワンタイム）

であるべき。

しかしこのラボでは：

- username と md5(password) を Base64 しただけ  
- 誰でも再現可能  
- 秘密鍵なし  
- 署名なし  
- ランダム性ゼロ

→ **攻撃者が Cookie を自作できる＝認証破壊**

---

## ❌ **2. MD5 は高速すぎてブルートフォースに弱い**
- GPU で毎秒数十億回ハッシュ可能  
- ソルトなし  
- ストレッチングなし  
- しかもパスワード候補リストが与えられている

→ **攻撃者にとっては「ただの文字列変換」**

---

## ❌ **3. Cookie の値が「パスワードハッシュ」そのもの**
これは最悪。

- Cookie が漏れたらパスワードハッシュが漏れる  
- ハッシュを使ってログインできる  
- パスワード変更しても Cookie が無効化されない可能性がある

---

# 🛡 どう対策すればいい？
ここが面接で一番評価されるポイント。  
和志のスタイルに合わせて、**色分け・構造化**してまとめる。

---

## 🟩 **1. 永続ログイン Cookie はランダムトークンにする**
- `stay-logged-in = random_128bit_token`
- DB に保存（user_id, token_hash, expiry）
- Cookie には **生の token**  
- DB には **ハッシュ化した token** を保存

→ **攻撃者はトークンを生成できない**

---

## 🟦 **2. Cookie に署名（HMAC）を付ける**
```
cookie = base64(username + ":" + expiry + ":" + HMAC(secret, username+expiry))
```

→ **秘密鍵がないと偽造できない**

---

## 🟧 **3. パスワードハッシュは強力な KDF を使う**
- bcrypt  
- scrypt  
- Argon2  

→ **高速ブルートフォースを不可能にする**

---

## 🟥 **4. Cookie の有効期限を短くする**
- 30日 → 長すぎ  
- 7日 or 14日  
- 重要操作は再認証

---

## 🟪 **5. ログアウト時にトークンを無効化**
- DB の token を削除  
- Cookie を破棄

---

## 🟫 **6. Cookie の再利用を検知して無効化**
- 1 トークン 1 回限り  
- 使われたら新しいトークンを発行

---

# 🧠 まとめ（面接でそのまま言えるレベル）
```
この脆弱性は、永続ログイン Cookie が
base64(username:md5(password)) という予測可能な構造で、
署名もランダム性もないため、攻撃者が Cookie を自作できてしまう点にあります。

攻撃者はパスワード候補を MD5 化し、carlos:<md5> を Base64 化するだけで
有効な Cookie を生成でき、アカウントを乗っ取れます。

対策としては、永続ログイン Cookie をランダムトークン化し、
サーバー側でハッシュ化して保存する方式に変更し、
HMAC 署名、短い有効期限、トークンのワンタイム化を行うことが重要です。
```

