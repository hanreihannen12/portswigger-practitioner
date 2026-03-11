# 🔥 URLマッチング不一致（URL-matching discrepancies）  
**アクセス制御をすり抜けるために “URLの形を少し変えて送るだけ” の攻撃**

## ✅ 手順まとめ（最短版）

### **1. 通常リクエストを送る**
```
GET /admin/deleteUser
→ 401 / 403 が返ることを確認
```

### **2. Burp Repeater に送る**

### **3. URLのバリエーションを試す**
以下を順番に送信：

```
/ADMIN/DELETEUSER
/Admin/DeleteUser
/admin/deleteuser
/admin/deleteUser.jpg
/admin/deleteUser.anything
/admin/deleteUser/
/admin/deleteUser//
```

### **4. どれかで 200 OK が返ったら脆弱性あり**

---

# 🔥 水平権限昇格（Horizontal Privilege Escalation / IDOR）

**URLパラメータの ID を変えて他人の情報が見えるか確認するだけ**

## ✅ 手順まとめ（最短版）

### **1. 自分のアカウントページのリクエストを取得**
```
GET /myaccount?id=123
→ 200 OK（自分の情報）
```

### **2. Repeater に送る**

### **3. ID を変更して送信**
```
id=123 → id=124
```

### **4. レスポンスを確認**
- **他人の情報が返る → IDOR / 水平権限昇格**
- **403 Forbidden → 正しく防御されている**

---

# 🧩 まとめ（さらに短く）

## URLマッチング不一致
```
1. 正常リクエスト → 403 を確認
2. URLの大文字/拡張子/スラッシュを変えて送る
3. 200 OK が返ればバイパス成功
```

## 水平権限昇格（IDOR）
```
1. 自分のページのリクエストを Repeater に送る
2. id を他人の値に変更
3. 他人の情報が返れば脆弱性
```

---

必要なら **「面接用の15秒説明」** も作るよ。
