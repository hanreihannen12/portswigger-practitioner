## 完全版 手順書（修正済み）

---

## STEP 1：ダミーログインでリクエストをキャプチャ

1. Burp Suite を起動
2. **Proxy タブ → Intercept → 「Intercept is on」** になってることを確認
3. ブラウザでラボのログインページを開く
4. Username に `test`、Password に `test` と入力して **「Log in」** をクリック
5. Burp の Intercept 画面にリクエストが止まる
6. リクエスト上で**右クリック → 「Send to Intruder」**
7. **「Forward」ボタン**を押してリクエストを流す

---

## STEP 2：Intruder で Username を Payload に設定

1. Burp 上部の **「Intruder」タブ** をクリック
2. **「Positions」タブ** を開く
3. 右側の **「Clear §」ボタン** をクリック（既存マーカーを全消し）
4. 一番下の行にある `username=test&password=test` を確認
5. `test`（username の値）だけをマウスで選択
6. **「Add §」ボタン** をクリック
   - `username=§test§&password=test` になればOK
7. Attack type は **「Sniper」** のままでOK

---

## STEP 3：Username の Wordlist を貼り付け

1. 画面右側パネルの **「Payloads」** をクリック
2. Payload type が **「Simple list」** になってることを確認
3. PortSwigger のラボページにある **「Candidate usernames」** リンクを開く
4. 全部コピーして Burp の **「Paste」ボタン** でペースト
5. リストに username が並んでいることを確認

---

## STEP 4：Grep - Extract の設定

1. 画面右端の縦に並んだアイコンの中から**ギアマーク（⚙）** をクリック
   - Settings パネルが開く
2. 下にスクロールして **「Grep - Extract」** セクションを見つける
3. **「Add」ボタン** をクリック → ダイアログが開く
4. ダイアログ下部にレスポンスの HTML が**すでに表示されている**（Fetch 済みの状態）
5. 下にスクロールして **54行目** を探す
6. `Invalid username or password.` という文字列を**マウスでドラッグして選択**
   - `<p class=is-warning>` や `</p>` タグは含めず、テキストのみ選択
   - 選択すると「Start after expression」と「End at delimiter」が**自動で入力される**
7. ダイアログ右下の **「OK」** をクリック

---

## STEP 5：Username 列挙の攻撃を開始

1. **「Start attack」ボタン** をクリック
2. 攻撃結果ウィンドウが開く（数分かかる）
3. 完了したら Grep - Extract で設定した**エラーメッセージのカラムをクリックしてソート**
4. ほとんどは `Invalid username or password.` と表示される
5. **1つだけ末尾にスペースがある（ピリオドがない）** やつを探す
   - 例：`Invalid username or password ` ← スペースで終わってる
6. そのリクエストの Username の値を**メモ**（例：`apache`）

---

## STEP 6：Password ブルートフォースの準備

1. 攻撃結果ウィンドウを**閉じる**
2. Intruder の **「Positions」タブ** に戻る
3. `username=` の値をメモしたユーザー名に書き換える
   - 例：`username=apache&password=test`
4. **「Clear §」ボタン** をクリック
5. `test`（password の値）をマウスで選択
6. **「Add §」ボタン** をクリック
   - `username=apache&password=§test§` になればOK

---

## STEP 7：Password の Wordlist を設定

1. 右側パネルの **「Payloads」** をクリック
2. 今のリストを全選択 → **「Remove」ボタン** で全消去
3. PortSwigger のラボページにある **「Candidate passwords」** リンクを開く
4. 全部コピーして **「Paste」ボタン** でペースト

---

## STEP 8：Password ブルートフォースを開始

1. **「Start attack」ボタン** をクリック
2. 攻撃結果ウィンドウの **「Status」カラムをクリック** してソート
3. ほとんどが `200` のところ **`302`** になってるやつが1つある
4. そのリクエストの Password の値を**メモ**

---

## STEP 9：ログインしてラボ完了

1. ブラウザのログインページに戻る
2. Username：STEP 5 でメモしたユーザー名
3. Password：STEP 8 でメモしたパスワード
4. **「Log in」** をクリック
5. `/my-account` ページに遷移できたら **ラボ完了！**
	2つのラボの違い

前のラボ（簡単バージョン）
エラーメッセージがはっきり違うやつやった。
状況エラーメッセージ存在しないユーザーInvalid username存在するユーザーIncorrect password
メッセージが全然違うから、Length（レスポンスの長さ）でソートするだけで一発でわかった。

今のラボ（難しいバージョン）
エラーメッセージがほぼ同じやつ。
状況エラーメッセージ存在しないユーザーInvalid username or password.存在するユーザーInvalid username or password 
違いがピリオドの有無だけやから、Length でソートしても気づきにくい。だから Grep - Extract でエラーメッセージを抽出して直接比較する必要があった。

まとめ

前のラボ → メッセージが違うから簡単に見つかる
今のラボ → メッセージがほぼ同じやから Grep - Extract で細かく比較する必要がある

「Subtly different（微妙に違う）」 ってラボ名の意味がそこにあるで！でもlengthのとこクリックしたらすぺーすのやつ一番上に北出9:54ユーザーがソート機能で同じ結果が得られることに気づいた。The user is saying that even in the current lab, clicking the Length column to sort showed the different response at the top (because the trailing space vs period makes a 1-character length difference). So they're questioning why Grep - Extract is needed if Length sorting works too.完了それでも正解や！
Length でソートしても見つかる理由は、末尾のスペースと ピリオドで 1文字分だけ長さが違うから。

じゃあ Grep - Extract いらんの？

このラボなら Length でも解ける。
でも現実のペネトレーションテストでは…
状況LengthGrep - Extract1文字の違い△ ギリ気づける◎ 確実にわかる広告やランダムな要素でLength がバラバラ✕ 全然使えない◎ 確実にわかるレスポンスの内容で判断したい✕ 無理◎ 確実にわかる

つまり

Length でたまたま見つかったのはラッキーなケースで、Grep - Extract は確実に見つけるための方法やで！



<br>

---

# 🧩 **このラボの本質：Subtle Username Enumeration（微妙なユーザー名列挙）**

---

# ✅ **1. どんな脆弱性？（What）**

### **認証エラーメッセージの微妙な差異による Username Enumeration**
- 存在しないユーザー  
  → `Invalid username or password.`（ピリオドあり）
- 存在するユーザー  
  → `Invalid username or password `（末尾スペースで終わる）

**見た目は同じだが、実際は1文字だけ違う。**  
この差が攻撃者に「ユーザーが存在するかどうか」を教えてしまう。

---

# ✅ **2. どう気づく？（How to detect）**

### **レスポンスの差異を比較する**
- Length ソート → 1文字だけ長さが違うレスポンスが出る  
- Grep - Extract → エラーメッセージを抽出して比較すると確実に判別できる

**「同じエラーに見えるのに、レスポンスが微妙に違う」**  
ここに気づくのがポイント。

---

# ✅ **3. どう攻撃する？（How to exploit）**

### **攻撃フェーズは2段階**

---

## **Phase 1：Username Enumeration**
1. Intruder Sniper で username をペイロード化  
2. 候補リストを流す  
3. レスポンスの差異（ピリオド vs スペース）で **存在するユーザー名を特定**

---

## **Phase 2：Password Brute-force**
1. 見つけた username を固定  
2. password をペイロード化  
3. 候補パスワードを流す  
4. **302 リダイレクト** が成功の合図  
5. 正しいパスワードを取得してログイン成功

---

# ✅ **4. どう対策する？（Mitigation）**

### **認証まわりのベストプラクティス**

---

## **① エラーメッセージを完全に統一する**
- 「Invalid username or password」  
- ピリオド・スペース・HTML構造も含めて完全一致させる

---

## **② レートリミット / IP ブロック**
- 連続ログイン失敗を制限  
- 1ユーザーあたりの試行回数を制限

---

## **③ アカウントロックアウト（時間制限付き）**
- 5回失敗 → 5分ロックなど

---

## **④ CAPTCHA の導入**
- 自動化攻撃を困難にする

---

## **⑤ 認証ログの監視**
- 異常な試行回数を検知してアラート

---

# 🎤 **30秒面接回答（そのまま言えるやつ）**

> この脆弱性は、ログインエラーのメッセージが微妙に異なることでユーザー名が列挙できてしまう問題です。  
> 実際には「ピリオドの有無」や「末尾スペース」など1文字の差で、レスポンスの長さや抽出したメッセージを比較すると存在するユーザーを特定できます。  
> 攻撃者はまずこの差異でユーザー名を割り出し、その後パスワードのブルートフォースを行ってログインを突破します。  
> 対策としては、エラーメッセージを完全に統一すること、レートリミットやCAPTCHAの導入、アカウントロックアウトなどが有効です。

---

# 🎤 **さらに短い15秒版**

> エラーメッセージの微妙な差異でユーザー名が判別できる脆弱性です。  
> レスポンスの長さや内容を比較して存在するユーザーを特定し、その後パスワードを総当たりします。  
> 対策はエラーメッセージの完全統一とレートリミットです。

---
