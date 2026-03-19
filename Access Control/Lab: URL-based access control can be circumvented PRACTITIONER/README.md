### 1. /admin に直接アクセスしてブロックを確認

1. **ブラウザでラボを開く。**
2. **URLバーに `/admin` を付けてアクセス**  
   例:  
   `https://<lab-domain>/admin`
3. **ブロック画面を確認**  
   - シンプルなエラーページ（WAFやリバースプロキシっぽい）になっているはず。

ここで「フロント側でパスを見てブロックしている」ことを確認するイメージ。

---

### 2. リクエストを Burp Repeater に送る準備

1. **Burp を起動して、ブラウザのプロキシ設定を Burp に向ける。**
2. 再度 `/admin` にアクセスして、  
   **Burp Proxy → HTTP history** に `/admin` へのリクエストがあることを確認。
3. その `/admin` リクエストを右クリックして  
   **「Send to Repeater」** を選択。

---

### 3. Repeater で「X-Original-URL が効いているか」テスト

1. **Repeater タブを開いて、さっき送ったリクエストを表示。**
2. リクエストラインを次のように変更：

   元:
   ```http
   GET /admin HTTP/1.1
   ```
   変更後:
   ```http
   GET / HTTP/1.1
   ```

3. ヘッダに **X-Original-URL** を追加して、最初はダミー値を入れる：

   ```http
   X-Original-URL: /invalid
   ```

   全体イメージ:
   ```http
   GET / HTTP/1.1
   Host: <lab-domain>
   X-Original-URL: /invalid
   （他のヘッダはそのまま）
   ```

4. **「Send」ボタンを押してレスポンスを確認。**
   - レスポンスが「Not found」など、普通のアプリっぽいエラーになればOK。  
   → これは「バックエンドが X-Original-URL を本当のパスとして解釈している」サイン。

---

### 4. X-Original-URL を /admin にして管理画面に入る

1. 同じ Repeater リクエストで、ヘッダだけ変更：

   ```http
   X-Original-URL: /admin
   ```

2. もう一度 **「Send」**。
3. レスポンスボディに **admin パネルっぽい内容**（ユーザー一覧や「delete」リンクなど）が出ていれば成功。

---

### 5. carlos を削除するリクエストを作る

このラボのポイントは：

- **実際のリクエストラインのパス**（フロントが見る部分）
- **X-Original-URL のパス**（バックエンドが見る部分）

を分離して使うこと。

1. Repeater で、リクエストラインを次のように変更：

   ```http
   GET /?username=carlos HTTP/1.1
   ```

   つまり「表向きの URL（フロント側が見るクエリ）」に `?username=carlos` を付ける。

2. ヘッダの **X-Original-URL** を削除ではなく、次のように変更：

   ```http
   X-Original-URL: /admin/delete
   ```

   全体イメージ:
   ```http
   GET /?username=carlos HTTP/1.1
   Host: <lab-domain>
   X-Original-URL: /admin/delete
   （他のヘッダはそのまま）
   ```

3. **「Send」ボタンを押す。**
4. レスポンスを確認して、  
   - 「ユーザー carlos を削除しました」系のメッセージ  
   - またはラボの上部に「Solved」表示  
   が出ていればクリア。

---

### 6. 流れを一言でまとめると

- `/admin` はフロントでブロックされる。
- バックエンドは `X-Original-URL` を本当のパスとして扱う。
- 表向きのパスは `/` にしてブロックを避けつつ、  
  `X-Original-URL: /admin` で管理画面に入り、  
  `X-Original-URL: /admin/delete` ＋ `?username=carlos` で削除処理を実行する。



---

# 🔥 1. どーいう脆弱性？
## **フロントエンドとバックエンドのアクセス制御の不一致（Access Control Discrepancy）**
- フロント側（WAF/リバースプロキシ）が `/admin` をブロックしている  
- しかしバックエンドは **X-Original-URL** を優先してルーティング  
- 結果、**外部から本来禁止の管理パスにアクセスできてしまう**

つまり **「フロントはブロックしてるつもりでも、バックエンドが裏口を開けている」** 状態。

---

# 🔍 2. どう気づく？
### 気づきポイント（実際のラボと同じ流れ）
- `/admin` にアクセスすると **シンプルなブロック画面** → フロント側の制御っぽい
- Burp Repeater で  
  `GET /` に変更しつつ `X-Original-URL: /invalid` を付けると  
  → バックエンドが `/invalid` として処理している挙動が見える
- つまり **バックエンドが X-Original-URL を信じている** と判明

---

# 🎯 3. どう攻撃する？
### 実際の攻撃手順（ラボの流れ）
1. `/admin` はフロントでブロックされる
2. Repeater でリクエストを次のように改造：

```
GET / HTTP/1.1
X-Original-URL: /admin
```

→ バックエンドが `/admin` として処理し、管理画面に入れる

3. ユーザー削除は `/admin/delete?username=carlos`  
   しかしフロントに見える URL は `/` にしておく必要がある

```
GET /?username=carlos HTTP/1.1
X-Original-URL: /admin/delete
```

→ フロントは `/` しか見ないので通過  
→ バックエンドは `/admin/delete` として処理  
→ carlos 削除成功

---

# 🛡 4. どう対策する？
### **根本対策**
- **フロントとバックエンドでアクセス制御を統一する**
- バックエンドが **X-Original-URL / X-Rewrite-URL** を信頼しない
- これらのヘッダを **削除** してからバックエンドへ渡す
- もしくはバックエンド側で **認証・認可を必ず実施**

### 追加対策
- リバースプロキシで許可ヘッダをホワイトリスト化
- 管理画面は VPN / IP 制限 / MFA などで強化

---

# ⏱ 5. まとめ


> この脆弱性は、フロントエンドとバックエンドのアクセス制御が不一致なことで発生する。  
> フロント側は `/admin` をブロックしていますが、バックエンドは `X-Original-URL` を優先してルーティングするため、ヘッダを書き換えるだけで管理画面にアクセスできてしまう。  
> 攻撃者は `X-Original-URL: /admin/delete` のように送ることで管理操作を実行できる。  
> 対策は、バックエンドでも必ず認可チェックを行い、信頼できないヘッダを削除すること。

---

