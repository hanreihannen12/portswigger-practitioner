

# 🧪 Lab: 2FA Simple Bypass — 手順まとめ（Markdown版）

## Step 1: wiener アカウントでログインして `/my-account` のURLを確認

1. ラボを開く（**ACCESS THE LAB** ボタンをクリック）
2. ブラウザでラボサイトが開く
3. 右上の **My account** をクリック
4. ログインフォームに以下を入力：

   ```
   Username: wiener
   Password: peter
   ```

5. **Login** をクリック
6. 「**Please enter your 4-digit security code**」という2FA入力画面が表示される
7. ラボページの **Email client** をクリック（別タブで開く）
8. メールに届いた4桁コードをコピー
9. 2FA入力画面に戻り、コードを入力
10. ログイン完了 → アカウントページへ遷移  
11. URLバーを確認し、`/my-account` をメモしておく

---

## Step 2: ログアウト

1. ページ上部の **Log out** をクリック
2. ログインページに戻ることを確認

---

## Step 3: carlos アカウントでログインし、2FAをスキップ

1. ログインフォームに以下を入力：

   ```
   Username: carlos
   Password: montoya
   ```

2. **Login** をクリック
3. 2FA入力画面が表示される（コードは不要）
4. URLバーをクリックし、現在のURL（例）：

   ```
   https://xxxxx.web-security-academy.net/login2
   ```

   を以下に書き換える：

   ```
   https://xxxxx.web-security-academy.net/my-account
   ```

5. **Enter** を押す
6. Carlos のアカウントページが表示される → **ラボ完了**

---

# 🔥 2FA Bypass（アクセス制御不備）まとめ

## 🧩 どーいう脆弱性？
**2FA が “特定の URL にアクセスしたときだけ” チェックされており、  
認証後のページ `/my-account` に直接アクセスすると 2FA をスキップできてしまうアクセス制御の欠陥。**

要するに：

- 2FA は **login2 ページでしか検証されていない**
- 認証後ページ `/my-account` に直接行くと **2FA 未完了でも通ってしまう**

これは **Broken Access Control / 2FA Logic Flaw**。

---

## 🔍 どう気づいた？
- wiener で正しくログイン → 2FA → `/my-account` に遷移  
- **URL をメモしておく（/my-account）**
- ログアウト後、carlos でログインすると 2FA 画面に止まる  
- しかし URL を `/my-account` に書き換えると **普通に入れてしまう**

→ **2FA が URL ベースでしか制御されていない** と判明。

---

## 🎯 どう攻撃した？
1. carlos でログイン（パスワードは正しい）
2. 2FA 入力画面で止まる
3. URL を `/login2` → `/my-account` に書き換える
4. **2FA をスキップしてアカウントページに侵入**

---

## 🛡️ どう対策する？
- **2FA 完了フラグをサーバ側セッションに保持する**
  - 例：`session.is_2fa_verified = true`
- すべての認証後ページで  
  **「2FA 完了済みか？」をサーバ側でチェック**
- URL だけでアクセス制御しない
- 2FA の状態をクライアント側に依存しない

---

# ⏱️ 30秒面接用まとめ（和志の面接テンプレ）

**「この脆弱性は 2FA のロジック不備で、2FA が login2 ページでしか検証されていないため、  
認証後ページ `/my-account` に直接アクセスすると 2FA をスキップできる問題です。  
気づいたきっかけは、wiener でログイン後の URL を確認し、carlos で 2FA 画面から URL を書き換えたら通ったことです。  
攻撃者は正しいパスワードさえ知っていれば 2FA を回避してアカウントに侵入できます。  
対策は、2FA 完了フラグをサーバ側セッションで保持し、すべての認証後ページで 2FA 完了を必ず検証することです。」**

---
