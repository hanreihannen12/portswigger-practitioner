

# 🔐 Lab: 2FA Broken Logic（Cookie 書き換え＋ブルートフォース）

## 🧭 全体の攻撃フロー
1. wiener でログインして verify パラメータを確認  
2. GET /login2 の verify を carlos に書き換えて送信（carlos の 2FA コード生成）  
3. POST /login2 の verify を carlos に書き換えて 4 桁コードを Intruder で総当たり  
4. 成功したリクエストは 302 を返す  
5. 302 のレスポンスをブラウザで開いて carlos としてログイン完了  

---

# 🛠️ Intruder を使った実際の手順（詳細）

## Step1: wiener でログインし、/login2 を確認
1. `wiener:peter` でログイン  
2. `/login2` にリダイレクトされる  
3. Burp Proxy → HTTP history で `/login2` を確認  
4. Cookie に `account=wiener`、パラメータに `verify=wiener` があることを確認

---

## Step2: GET /login2 の verify を carlos に書き換える
1. `/login2` の **GET リクエスト**を右クリック → **Send to Repeater**  
2. Repeater で以下のように書き換える：

```
GET /login2?verify=carlos HTTP/1.1
Cookie: session=xxxx; account=carlos
```

3. Send  
→ これで **サーバーが carlos の 2FA コードを生成**する

---

## Step3: POST /login2 を Intruder に送ってブルートフォース

### ① POST /login2 を Intruder に送る
1. `/login2` の **POST リクエスト**を右クリック → **Send to Intruder**

---

### ② Intruder → Positions タブで設定
- `verify=wiener` → **verify=carlos に書き換える**
- `mfa-code=xxxx` の部分だけを選択して **§payload position§** にする  
  （他の位置は **Clear** で全部消す）

例：

```
verify=carlos
mfa-code=§0000§
```

---

### ③ Attack type を選ぶ
- Attack type → **Sniper**（1 箇所だけ総当たりするのでこれでOK）

---

### ④ Payloads タブで 0000〜9999 を設定
1. Payload type → **Numbers**  
2. From: `0`  
3. To: `9999`  
4. Step: `1`  
5. **Min length: 4**（ゼロ埋めのため）  
6. **Max length: 4**

→ これで `0000`〜`9999` が自動生成される

---

### ⑤ Intruder → Start attack
- 10,000 リクエストが走る  
- レスポンスの **Status** または **Length** を見て異常値を探す  
- 成功すると **302** が返る

---

## Step4: 成功したリクエスト（302）を確認
- 302 のレスポンスをクリック  
- `Location: /my-account` のような遷移先がある  
- これが **正しい 2FA コード**

---

## Step5: 302 のレスポンスをブラウザで開く
- そのリクエストを右クリック → **Copy URL**  
- ブラウザで開く  
→ **carlos としてログイン成功**

---

# 💡 補足ポイント

## 🔸 なぜ GET /login2 を carlos に書き換える？
- サーバーに **「carlos の 2FA コードを生成して」** と依頼するため  
- これをしないと carlos のコードが存在しない  
- つまりブルートフォースが成立しない

## 🔸 ブルートフォースが成立する理由
- 4 桁 → 10,000 通り  
- レートリミットなし  
- CAPTCHA なし  
- Intruder が高速で総当たりできる

---

# 🧩 1. どーいう脆弱性？
## **2FA Broken Logic（認証フローの不備）**
- 2FA の対象ユーザーを **Cookie の `account` と `verify` パラメータだけで識別**している  
- つまり **攻撃者が Cookie を書き換えるだけで他人の 2FA コードを生成できる**  
- さらに 2FA にレートリミットが無く、4桁コードを **総当たり可能**

---

# 🕵️ 2. どう気づいて？
- `/login2` のリクエストを Burp で確認すると  
  - `verify=wiener`  
  - Cookie: `account=wiener`  
  が **クライアント側で自由に変更可能**  
- GET `/login2?verify=carlos` を送ると **エラーにならず正常応答**  
→ **サーバーが carlos の 2FA コードを生成している**と判明  
- POST `/login2` も同じく verify を書き換え可能  
→ **他人の 2FA をブルートフォースできる構造**だと気づく

---

# 🔨 3. どう攻撃して？
1. wiener でログインして `/login2` を取得  
2. GET `/login2?verify=carlos` を送信  
   → サーバーが carlos の 2FA コードを生成  
3. POST `/login2` の  
   - Cookie: `account=carlos`  
   - Body: `verify=carlos`  
   に書き換え  
4. `mfa-code=0000〜9999` を Burp Intruder で総当たり  
5. 正しいコードのときだけ **302** が返る  
6. そのレスポンスをブラウザで開くと **carlos としてログイン成功**

---

# 🛡️ 4. どう対策する？
- **2FA の対象ユーザーをサーバー側セッションで管理する**  
  - クライアントの Cookie やパラメータに依存しない  
- **2FA コード生成はログイン済みユーザーにのみ紐づける**  
- **レートリミット・ロックアウトを導入**  
- **2FA コードは一度生成したら1回しか使えないようにする**  
- **IP/UA の異常検知**  
- **ログインフローを完全にサーバー側で制御**

---

# ⏱️ 5. 30秒面接用まとめ（あなた用に最適化）

```
この脆弱性は 2FA の対象ユーザーを Cookie の account や verify パラメータで
クライアント側に依存していたため、攻撃者が値を書き換えるだけで
他人の 2FA コードを生成できる Broken 2FA です。

GET /login2 を carlos に書き換えるとサーバーが carlos の 2FA を発行し、
POST /login2 も同様に書き換えられるため、4桁コードを Intruder で
0000〜9999 まで総当たりして突破できます。

対策は、2FA の対象ユーザーをサーバー側セッションで固定し、
クライアントのパラメータに依存しないこと、さらにレートリミットを
導入することです。
```

