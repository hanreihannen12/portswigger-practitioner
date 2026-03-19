

# 🧩 Inconsistent handling of exceptional input  
## — このラボでやったことの技術まとめ（Markdown）

## ## 🎯 ラボの目的
**「メールアドレスの扱いが “表示用” と “内部処理用” で不一致になる脆弱性を悪用し、  
内部だけ社員扱いにさせて admin パネルへ入る」**

---

# ## 🧠 脆弱性の正体（コア部分）
アプリはメールアドレスを次のように処理していた：

| 処理 | 内容 |
|------|------|
| **① 表示用** | 入力したメールアドレスをそのまま表示 |
| **② 内部の社員判定用** | **255文字で切り捨ててからドメイン判定** |

つまり、

> **見た目のメールアドレスと、内部で使われるメールアドレスが一致しない**

これが脆弱性の本質。

---

# ## 🚀 攻略の流れ（安全に説明）

## ### ① `/admin` の存在を確認
- Burp Suite Pro の “Discover content” が使えない環境だったので  
  **URL に直接 `/admin` を入力してアクセス**
- 結果：  
  **「DontWannaCry 社員のみアクセス可能」** と表示される  
  → 社員ドメインが鍵だと分かる

---

## ### ② メールクライアントで自分のメールドメインを確認
例：  
`@abcd1234.web-security-academy.net`

---

## ### ③ 長いメールアドレスで登録して挙動を観察
- まずは適当に長いメールで登録  
- My account を見ると **255文字で切り捨てられている**ことが分かる  
→ **このラボの核心：255文字で切れる**

---

## ### ④ 255文字目に社員ドメインを来させるように細工
ここが最大のポイント。

- メールアドレスを **255文字以上** にする  
- **255文字目が “@dontwannacry.com” の “m” になるように調整**

すると内部処理では：

```
...bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb@dontwannacry.com
```

→ **内部では社員メールに見える！**

---

## ### ⑤ メール認証 → ログイン
- メールクライアントに届いたリンクを踏んで登録完了  
- ログインすると…

👉 **Admin panel が表示される**

---

## ### ⑥ Admin panel で carlos を削除  
- `/admin` にアクセス  
- ユーザー一覧から **carlos を削除**  
- → ラボクリア

---

# ## 🎉 最終まとめ（超短縮版）

- `/admin` は社員だけアクセス可能  
- メールアドレスは **255文字で切り捨てられる**  
- 切り捨て後に **@dontwannacry.com** に見えるように細工  
- 内部処理だけ社員扱いになる  
- admin パネルに入り、carlos を削除してクリア

