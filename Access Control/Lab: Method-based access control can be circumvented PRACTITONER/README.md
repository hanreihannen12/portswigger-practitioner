## Phase 1: 管理者リクエストを手に入れる

### 1. ラボを開く
- **PortSwiggerのラボページ**で「**ACCESS THE LAB**」をクリック  
- 新しいタブでショッピングサイトが開く（これがターゲット）

### 2. 管理者でログイン
- 右上の **「My account」** をクリック  
- ログインフォームに入力:
  - **Username:** `administrator`
  - **Password:** `admin`
- 「**Log in**」をクリック → 管理者としてログイン成功  
- 画面上部に **「Admin panel」リンク**が出ていることを確認

### 3. Admin panel を開く
- 「**Admin panel**」リンクをクリック  
- 画面に以下のようなUIが出る:
  - `Upgrade user: [carlos ▼] [Upgrade user] [Downgrade user]`

### 4. BurpのProxyを準備
- Burp Suite を起動  
- **Proxy → Intercept** タブを開く  
- ボタンが **「Intercept is on」** になっていることを確認  
- ブラウザが Burp プロキシを使うように設定（FoxyProxyなど）

### 5. 「wiener を upgrade するリクエスト」をキャプチャ
- ブラウザの Admin panel 画面で:
  - ドロップダウンを **「wiener」** に変更  
  - **「Upgrade user」** ボタンをクリック
- Burp の **Proxy → Intercept** にリクエストが止まる  
  例:
  ```http
  POST /admin-roles HTTP/1.1
  Host: ...
  Cookie: session=管理者のセッション
  Content-Type: application/x-www-form-urlencoded

  username=wiener&action=upgrade
  ```

### 6. Repeater に送る & 実リクエストは捨てる
- Intercept 画面でそのリクエストを選択 → 右クリック → **「Send to Repeater」**  
- そのまま **「Drop」** ボタンを押して、このリクエストはサーバーに送らない  
- 上のボタンで **「Intercept is off」** に戻す

---
<br>
<br>
<br>


## Phase 2: 一般ユーザー wiener のセッションを取る

### 7. 管理者からログアウト
- ブラウザに戻って右上の **「Log out」** をクリック  
- トップページに戻る → 管理者ログアウト完了

### 8. wiener でログイン
- 右上の **「My account」** をクリック  
- ログインフォームに入力:
  - **Username:** `wiener`
  - **Password:** `peter`
- 「**Log in**」をクリック → wiener のマイアカウントページが表示される  
- この時点では **「Admin panel」リンクはない**

### 9. wiener のセッションをキャプチャ
- Burp の **Proxy → Intercept** を **「on」** にする  
- ブラウザで **F5** でページをリロード  
- Intercept に例えばこんなリクエストが止まる:
  ```http
  GET /my-account HTTP/1.1
  Host: ...
  Cookie: session=Bv2Jk9Tx5...
  ```
- `Cookie: session=` の値（`session=`の後ろ）を選択してコピー  
  - 例: `Bv2Jk9Tx5Cm8Qn1Wd3Yf7Hp6Ls4Rg0Zm`  
- メモ帳などに貼っておく（「wiener のセッション」として）

### 10. このリクエストは捨てて通常状態に戻す
- **「Drop」** を押す  
- **「Intercept is off」** に戻す

---
<br>
<br>
<br>
<br>

## Phase 3: メソッド変更でアクセス制御をすり抜ける

### 11. Repeater の管理者リクエストを開く
- Burp の **「Repeater」タブ**をクリック  
- さっき送った `/admin-roles` の **POST リクエスト**があるはず:
  ```http
  POST /admin-roles HTTP/1.1
  Cookie: session=管理者のセッション
  ...
  username=wiener&action=upgrade
  ```

### 12. Cookie を wiener のセッションに差し替える
- `Cookie: session=...` の値を、さっきコピーした **wiener のセッション値**に書き換える  
  例:
  ```http
  Cookie: session=Bv2Jk9Tx5Cm8Qn1Wd3Yf7Hp6Ls4Rg0Zm
  ```

### 13. まずは POST のまま送って「ブロックされる」ことを確認
- **「Send」** をクリック  
- Response に:
  ```http
  HTTP/1.1 401 Unauthorized
  Unauthorized
  ```
  → **POST ではちゃんとアクセス制御が効いている**ことを確認

### 14. HTTPメソッドを GET に変換
- Request ペイン内で右クリック → **「Change request method」** を選択  
- 自動でこう変わる:
  ```http
  GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
  Host: ...
  Cookie: session=Bv2Jk9Tx5Cm8Qn1Wd3Yf7Hp6Ls4Rg0Zm
  ```
- `Content-Type` や `Content-Length` は消えていてOK

### 15. GET リクエストを送信（ここが本番）
- **「Send」** をクリック  
- Response に:
  ```http
  HTTP/1.1 302 Found
  Location: /admin
  ```
  → **401 ではなく 302** → つまり **wiener で権限変更が通った**

---
<br>
<br>
<br>
<br>

## Phase 4: 昇格確認 & ラボ完了

### 16. wiener が管理者になったか確認
- ブラウザに戻って、今ログインしているのは wiener のまま  
- **F5** でページをリロード  
- 上部に **「Admin panel」リンク** が表示されていれば昇格成功

### 17. Admin panel に入る
- 「**Admin panel**」をクリック  
- 管理者用の画面（`Upgrade user: [carlos ▼] ...`）が見えればOK

### 18. ラボ完了の確認
- PortSwigger のラボページに戻る  
- 上部に緑のバナー:
  - `Congratulations, you solved the lab!`


了解。  
あなたの面接スタイルに合わせて、**「どーいう脆弱性で / どう気づいて / どう攻撃して / どう対策するか」**を、  
**URLマッチング不一致** と **メソッドベースアクセス制御バイパス** の2つをまとめて整理するよ。  
最後に **30秒面接用テンプレ** も付ける。

---

# 🔥 1. URLマッチング不一致（URL-matching discrepancies）

## 🧩 どーいう脆弱性？
**アクセス制御が “厳密なパス” を見ているのに、ルーティングは “ゆるいパス” を受け入れる不一致を突く脆弱性。**

例：
- アクセス制御 → `/admin/deleteUser` のみチェック  
- ルーティング → 大文字小文字無視、拡張子許可、スラッシュ許可

→ **攻撃者が URL を少し変えるだけでアクセス制御をスルーできる**

---

## 🔍 どう気づく？
Burp Repeater で以下を試すと気づける：

- `/ADMIN/DELETEUSER`
- `/admin/deleteUser.jpg`
- `/admin/deleteUser.anything`
- `/admin/deleteUser/`
- `/admin/deleteUser//`

**どれかが 200 OK なら不一致がある証拠。**

---

## 💥 どう攻撃する？
1. 正規リクエストは 401/403 でブロックされることを確認  
2. URLの大文字・拡張子・スラッシュを変えて送る  
3. 200 OK が返ればアクセス制御バイパス成功  
4. 管理者機能や機密データにアクセスできる

---

## 🛡 どう対策する？
- **ルーティングとアクセス制御を同じ基準で処理する**
- **大文字小文字を統一**
- **拡張子マッチングを無効化（Spring: useSuffixPatternMatch=false）**
- **スラッシュの有無を統一**
- **正規化したパスでアクセス制御を行う**

---

# 🔥 2. メソッドベースアクセス制御バイパス（Method-based access control）

## 🧩 どーいう脆弱性？
**POST のときだけアクセス制御を行い、GET ではチェックしない** という実装ミス。

例：
```python
if request.method == 'POST':
    check_admin()
# GET のときはチェックなし
```

→ **攻撃者は GET に変えるだけで管理者操作ができる**

---

## 🔍 どう気づく？
1. 管理者の正規リクエスト（POST）をキャプチャ  
2. 一般ユーザーのセッションに変えて送る → **401 Unauthorized**  
3. Repeater で **「Change request method」→ GET**  
4. GET で送ると **200 / 302** が返る

→ **メソッドによってアクセス制御が変わっている** と気づく。

---

## 💥 どう攻撃する？
1. 管理者の POST リクエストをコピー  
2. Cookie を一般ユーザーのセッションに差し替え  
3. Repeater で **POST → GET に変更**  
4. 送信するとアクセス制御をバイパス  
5. 権限昇格・管理者操作が可能になる

---

## 🛡 どう対策する？
- **メソッドに依存せず、全メソッドでアクセス制御を行う**
- **サーバー側で権限チェックを共通化**
- **クライアントから送られるメソッドを信用しない**
- **管理者操作は CSRF トークン＋認可チェックを必須にする**

---

# 🎤 30秒面接用まとめ（あなたの面接テンプレ）

## 🧩 URLマッチング不一致（30秒）
「URLマッチング不一致は、アクセス制御が厳密なパスを見ている一方で、ルーティングが大文字小文字や拡張子、スラッシュの違いを許容することで起きる脆弱性です。  
Burp で URL を変形して 200 が返れば気づけます。  
攻撃者は `/ADMIN/...` や `.jpg` を付けてアクセス制御をバイパスします。  
対策はパスの正規化と、ルーティングとアクセス制御を同じ基準で処理することです。」

---

## 🧩 メソッドベースアクセス制御バイパス（30秒）
「メソッドベースアクセス制御は、POST のときだけ権限チェックを行い、GET ではチェックしない実装ミスです。  
管理者の POST リクエストを GET に変えて送ると権限が通ることで気づけます。  
攻撃者はメソッドを変えるだけで管理者操作を実行できます。  
対策は全メソッドで統一した認可チェックを行い、メソッドに依存しないアクセス制御を実装することです。」

---

必要なら **図解版** や **15秒バージョン** も作るよ。
