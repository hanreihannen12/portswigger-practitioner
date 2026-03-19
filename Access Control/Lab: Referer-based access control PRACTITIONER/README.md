
# STEP 1: 準備

1. **Burp Suite を起動する**
2. **Burp のブラウザを開く**
   - Burp Suite の上のメニューから **「Proxy」** タブをクリック
   - **「Open Browser」** ボタンをクリック
   - Burp のブラウザが開く

3. **Intercept を OFF にする**
   - Proxy タブの中の **「Intercept」** タブをクリック
   - **「Intercept is on」** というボタンが見えたら**クリックして OFF にする**
   - ボタンが **「Intercept is off」** になったらOK

---

# STEP 2: Administrator でログイン

4. **Burp のブラウザのアドレスバーに Lab の URL を入力してEnter**

5. **「My account」をクリック**

6. ログインページで：
   - Username に **administrator** と入力
   - Password に **admin** と入力
   - **「Log in」をクリック**

---

# STEP 3: Admin Panel でCarlos を Upgrade してキャプチャ

7. **「Admin panel」をクリック**
   - ページ上部にリンクがあるはず

8. **Intercept を ON にする**
   - Burp に戻って **「Proxy」→「Intercept」タブ**
   - **「Intercept is off」ボタンをクリック** → **「Intercept is on」** になる

9. **ブラウザに戻って** Carlos の横にある **「Upgrade」をクリック**

10. **Burp に戻る** → リクエストがキャプチャされてる！

こんな感じのリクエストが見えるはず：
```
GET /admin-roles?username=carlos&action=upgrade HTTP/2
Host: xxxxx.web-security-academy.net
Cookie: session=ここがadministratorのセッション
Referer: https://xxxxx.web-security-academy.net/admin
```

11. **このリクエストを Repeater に送る**
    - キャプチャされたリクエストの上で **右クリック**
    - **「Send to Repeater」** をクリック

12. **Intercept を OFF に戻す**
    - **「Intercept is on」ボタンをクリック** → OFF にする

13. **「Forward」ボタンをクリック**してリクエストを通す
    - これでブラウザ側が止まらなくなる

---

# STEP 4: Wiener のセッションクッキーを取得

14. **パソコンで新しいシークレットウィンドウを開く**
    - Chrome なら **Ctrl+Shift+N**
    - Firefox なら **Ctrl+Shift+P**
    - ⚠️ Burp のブラウザじゃなくて普通のブラウザのシークレット！

15. シークレットウィンドウで **Lab の URL を入力してEnter**

16. **「My account」をクリック**

17. ログインページで：
    - Username に **wiener** と入力
    - Password に **peter** と入力
    - **「Log in」をクリック**

18. **ログインできたら F12 を押す**（DevTools が開く）

19. DevTools の上のメニューから **「Application」タブをクリック**
    - 見つからない場合は **「>>」** みたいなボタンを押すと隠れてるタブが出てくる

20. 左側のメニューから **「Cookies」→ Lab の URLをクリック**

21. **「session」という名前の行を探す**

22. **session の Value（値）の部分をトリプルクリックで全選択してコピー**
    - 長い文字列がコピーできる
    - 例: `abc123xyz456...`

---

# STEP 5: Burp Repeater で攻撃

23. **Burp に戻る**

24. 上のタブから **「Repeater」をクリック**

25. STEP 11 で送ったリクエストが表示されてる

26. リクエストの中の **`username=carlos`** の部分を **`username=wiener`** に変更
    - `carlos` をダブルクリックで選択
    - **`wiener`** と入力して上書き

27. リクエストの中の **`Cookie: session=`** の後ろの値を全部消して、**STEP 22 でコピーした wiener のセッションに貼り付ける**
    - 既存の session の値をトリプルクリックで選択
    - Ctrl+V で貼り付け

28. **Referer ヘッダーはそのまま触らない！**
    - `Referer: https://xxxxx.web-security-academy.net/admin` のまま

29. リクエストが最終的にこうなってるか確認：
```
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: xxxxx.web-security-academy.net
Cookie: session=wiener_のセッション値
Referer: https://xxxxx.web-security-academy.net/admin
```

30. **「Send」ボタンをクリック**（Repeater の左上らへんにある）

31. 右側のレスポンスを見る
    - **`302 Found`** が返ってきたら成功！

---

# STEP 6: 確認

32. **シークレットウィンドウに戻る**（wiener でログインしてるやつ）

33. **ページをリロードする（F5）**

34. **「Admin panel」リンクが表示されてたら Lab クリア！** 🎉

---



# 🔥 **このラボの脆弱性：アクセス制御の欠落（Broken Access Control）**
正確には **「2ステップ操作のうち、2番目のリクエストだけアクセス制御が欠落している」** という典型的なロジックフローのミス。

- 管理者 UI は  
  ① Upgrade ボタン →  
  ② Yes（確認）  
  の2段階になっている。
- **②のリクエストだけが「管理者かどうか」をチェックしていない。**

つまり：

> **“管理者が押すはずのリクエスト”を、一般ユーザーが送っても通ってしまう**

これが本質。

---

# 🔍 **どう気づく？（気づきのポイント）**

1. Admin panel の Upgrade 操作が **2ステップ**になっている  
2. Burp でキャプチャすると、  
   - 1つ目のリクエストは admin しか送れない  
   - 2つ目のリクエストは **ただの GET /admin-roles?username=xxx&action=upgrade**
3. **Referer が admin ページを指しているだけで、Cookie の権限チェックが甘い**

→ **「これ、セッションだけ差し替えたら通るのでは？」**  
と気づける。

---

# 🎯 **攻撃の流れ（要点だけ）**

1. 管理者でログインして、Upgrade ボタンを押す  
2. **2つ目の Upgrade リクエスト（確認用）を Burp でキャプチャして Repeater に送る**
3. Repeater で：
   - `username=carlos` → `username=wiener` に変更
   - Cookie を **wiener のセッション**に差し替える
   - Referer はそのまま
4. 送信すると **302 Found** → 成功  
5. wiener の画面をリロードすると Admin panel が出る

---

# 🛡 **どう対策する？（面接で評価される答え）**

- **サーバー側で必ず権限チェックを行う**  
  UI のステップ数に依存しない
- **重要操作は POST + CSRF トークン + 権限チェック**  
- **Referer に頼らない**  
- **IDOR / ロジックフローの検証を徹底**

---

# ⚡ **まとめ**

```
この問題は、管理者の2ステップ操作のうち、
2つ目の「Upgrade 確認リクエスト」だけアクセス制御が欠落している
Broken Access Control 。

管理者で正規操作を行い、その2つ目のリクエストを Burp で取得。
Repeater で username を wiener に書き換え、
Cookie を wiener のセッションに差し替えて送ると、
権限チェックが無いため一般ユーザーでも管理者化できる。

対策は、全ての重要操作でサーバー側の権限チェックを行い、
UI のステップ数に依存しないこと。
```

