
# 🛠️ Broken brute-force protection, IP block  
## Burp Suite を使った 2段階攻撃（ユーザー名列挙 → パスワードブルートフォース）

---

## **Step 1: ログインリクエストをキャプチャ**

1. Burp Suite を起動  
2. **Proxy → Intercept is on** を確認  
3. ブラウザでラボのログインページを開く  
4. Username: `test` / Password: `test` を入力して Login  
5. Burp の Proxy に `POST /login` が止まる  
6. **Intercept is off** にしてリクエストを通す  
7. **HTTP history → POST /login → 右クリック → Send to Intruder**

---

## **Step 2: Intruder 設定（Phase 1 - ユーザー名列挙）**

### Positions タブ

1. **Clear §** を押して全ペイロード位置を削除  
2. ヘッダーに以下を追加：

```
X-Forwarded-For: 0
```

3. `0` を選択 → **Add §**  
   → `X-Forwarded-For: §0§`  
4. `username=test` の `test` を選択 → **Add §**  
   → `username=§test§`  
5. `password=` の値を 100 文字の適当な文字列に変更（ペイロード化しない）

```
password=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

6. Attack type → **Pitchfork**

---

## **Step 3: Payloads タブ（Phase 1）**

### Payload set 1（X-Forwarded-For 用）

- Payload set: **1**
- Payload type: **Numbers**
- 設定：

```
From: 1
To: 100
Step: 1
Max fraction digits: 0
```

### Payload set 2（ユーザー名リスト）

- Payload set: **2**
- Payload type: **Simple list**
- ラボの Candidate usernames をコピーして貼り付け

---

## **Step 4: 攻撃開始（Phase 1）**

1. **Start attack**  
2. 攻撃完了後、上部の **Columns** をクリック  
3. **Response received** と **Response completed** にチェック  
4. Response received を降順ソート  
5. 一番時間が長いリクエストの **username** をメモ 📝

---

## **Step 5: Intruder 設定（Phase 2 - パスワードブルートフォース）**

1. 攻撃ウィンドウを閉じる  
2. Intruder → Positions  
3. **Clear §**  
4. `X-Forwarded-For: §0§` はそのまま  
5. `username=` を Phase 1 で見つけたユーザー名に変更（§なし）  
6. `password=` の値を適当な文字列に変更 → 選択 → **Add §**  
   → `password=§dummy§`

---

## **Step 6: Payloads タブ（Phase 2）**

### Payload set 1（X-Forwarded-For）

Phase 1 と同じ：

```
Numbers, 1〜100, step 1
```

### Payload set 2（パスワードリスト）

- ラボの Candidate passwords を貼り付け

---

## **Step 7: 攻撃開始（Phase 2）**

1. **Start attack**  
2. **Status 列を確認**  
3. **302** のリクエストを探す 🎯  
4. そのリクエストの **password** をメモ

---

## **Step 8: ログイン**

1. ブラウザでログインページへ戻る  
2. 見つけた username + password でログイン  
3. My account ページへアクセスしてラボ完了 🎉


---

# 🔥 Broken brute-force protection（IP ブロック回避）  
## 1. **どーいう脆弱性？（What）**

- アプリが **ログイン試行回数を IP アドレスでしか制限していない**  
- しかし、攻撃者は **X-Forwarded-For ヘッダーを偽装** することで、  
  *毎回違う IP からのアクセスに見せかけて無限にログイン試行できる*  
- 結果：  
  **ユーザー名列挙 → パスワードブルートフォースが成立**

---

## 2. **どう気づく？（How to detect）**

- ログイン失敗を繰り返しても **アカウントロックされない**  
- `X-Forwarded-For` を変えると **レスポンスが変化しない**（制限が効いていない）  
- Burp Intruder で  
  - **ユーザー名ごとにレスポンス時間が変わる** → ユーザー名列挙  
  - **特定のパスワードで 302 が返る** → 認証成功

---

## 3. **どう攻撃する？（Attack）**

### Phase 1：ユーザー名列挙  
- `X-Forwarded-For` を 1〜100 で変化させる  
- username 候補を同時に送る（Pitchfork）  
- **レスポンス時間が長い username が「存在するユーザー」**

### Phase 2：パスワードブルートフォース  
- 見つけた username を固定  
- `X-Forwarded-For` を毎回変える  
- パスワードリストを総当たり  
- **302 が返ったパスワードが正解**

---

## 4. **どう対策する？（Mitigation）**

### ❌ 悪い対策（今回破られたやつ）
- IP アドレスだけでレート制限  
- X-Forwarded-For を信頼してしまう

### ✅ 有効な対策
- **IP ではなくアカウント単位でロックアウト**  
- **X-Forwarded-For を信頼しない**（信頼できるリバースプロキシのみ）  
- **CAPTCHA / MFA の導入**  
- **レスポンス時間・メッセージを統一して列挙を防ぐ**  
- **WAF で異常な試行を検知**

---

# 🎤 30秒面接用まとめ（あなたの面接スタイル向け）

> **この脆弱性は、ログイン試行の制限を IP アドレスだけで行っているため、攻撃者が X-Forwarded-For を偽装すると無限に試行できる問題です。まず Intruder で IP を変えながらユーザー名候補を送ると、レスポンス時間の差から存在するユーザーを特定できます。次にそのユーザー名を固定してパスワードリストを総当たりすると、302 のレスポンスで正しいパスワードが分かります。対策としては、IP ではなくアカウント単位のロックアウト、X-Forwarded-For を信頼しない設定、CAPTCHA や MFA の導入が有効です。**


