Lab: Username enumeration via account lock

ほな、めっちゃ細かく手順説明するで！

---

## ステップ1：ラボを開いてBurpでリクエストをキャッチする

1. ラボを開いて、Burp SuiteのInterceptをONにする
2. ログインページで**でたらめなユーザー名・パスワード**を入力してログインボタンを押す
   - 例：username = `test`、password = `test`
3.HTTP historyで `POST /login` のリクエストを右クリック
4. **「Send to Intruder」** をクリック

---

## ステップ2：Intruderの設定（ユーザー名列挙）

### Positionsタブ

1. Intruder → **Positionsタブ**を開く
2. Attack typeを **「Cluster bomb」** に変更
3. まず右上の **「Clear §」** ボタンを押して既存のペイロードポジションを全部消す
4. `username=` の値の部分を選択して **「Add §」** をクリック
   - こうなる：`username=§test§`
5. `password=test` の最後（`test`の後ろ）にカーソルを置いて **「Add §」** をクリック
   - こうなる：`username=§test§&password=test§§`
   - ⚠️ passwordの値自体は変えない。`test`の後ろに**空のポジション**を追加するだけ

---

## ステップ3：Payloadsタブの設定

### Payload set 1（ユーザー名リスト）

1. **Payload set: 1** を選択
2. Payload type: **Simple list**
3. 「Paste」または「Load」でPortSwiggerの**candidateユーザー名リスト**を全部貼り付ける

### Payload set 2（Null payloads × 5回）

1. **Payload set: 2** を選択
2. Payload type: **Null payloads** に変更
3. 「Generate **5** payloads」を選択

> 💡 これで各ユーザー名が**5回ずつ**送信される = アカウントロックを意図的に発動させる

---

## ステップ4：攻撃スタート＆結果確認

1. 右上の **「Start attack」** をクリック
2. 結果ウィンドウで **「Length」** の列をクリックしてソート
3. **他より明らかにLengthが長いユーザー名**を探す
4. そのリクエストをクリックしてResponseを見ると：
   ```
   You have made too many incorrect login attempts.
   ```
   というメッセージが出てるはず
5. そのユーザー名をメモしておく ✅

---

## ステップ5：新しいIntruder攻撃（パスワードブルートフォース）

1. BurpのProxy → HTTP historyで `POST /login` を再度 **「Send to Intruder」**
2. Attack typeを **「Sniper」** に変更
3. 「Clear §」で全ポジションを消す
4. `username=` の値を**さっき見つけたユーザー名**に直接書き換える
   - 例：`username=identified-user`
5. `password=` の値を選択して **「Add §」** をクリック
   - `username=identified-user&password=§test§`

---

## ステップ6：Payloads＆Grep設定

### Payloadsタブ

1. Payload set: 1、Payload type: **Simple list**
2. **candidateパスワードリスト**を貼り付ける

### Settingsタブ（Grep - Extract）

1. 上部の **「Settings」タブ** を開く
2. **「Grep - Extract」** セクションまでスクロール
3. **「Add」** をクリック
4. 「Fetch response」をクリック → レスポンスが表示される
5. エラーメッセージのテキスト（例：`Invalid username or password`）を選択
6. **「OK」** をクリック

---

## ステップ7：攻撃スタート＆パスワード特定

1. **「Start attack」** をクリック
2. 結果ウィンドウで **Grep extractの列**を確認
3. ほとんどのリクエストは `Invalid username or password` か `too many attempts` のメッセージ
4. **エラーメッセージが空（何も入ってない）** 行を探す ← これが正しいパスワード ✅

---

## ステップ8：1分待ってからログイン

> ⚠️ アカウントがまだロックされてるので、**1分待つ**のが重要！

1. 1分後にログインページへ
2. 特定した **ユーザー名** と **パスワード** でログイン
3. **My account** ページにアクセス → ラボ完了！🎉

---

## 今回のロジックフロー（logic flaw）の正体

```
ロック発動 → でも正しいパスワードなら
ロックされていてもログイン成功してしまう！
```

これがこのラボの「logic flaw」やで。アカウントロックのチェックより先にパスワード検証が走ってる、または正しい認証情報にはロックが適用されない実装になってるってこと。

---

了解。ここまでの流れを **攻撃者視点 → 実装バグ → どう気づく → どう攻撃 → どう直す** の順で、面接でそのまま使えるレベルで整理するよ。  
あなたの面接スタイル（論理・実務・図解イメージ）に合わせて、**色分けイメージの構造**でまとめる。

---

# 🔥 これは何の脆弱性か？

## 🎯 **脆弱性名（複合）**
| 種類 | 内容 |
|------|------|
| **Username Enumeration** | ロック時のエラーメッセージ差異でユーザー存在がバレる |
| **Broken Brute-force Protection** | ロックアウトが正しく機能していない |
| **Authentication Logic Flaw** | ロック状態でも正しいパスワードならログイン成功してしまう |

**＝認証周りのロジックが破綻している複合脆弱性**

---

# 🧠 どう気づくのか（気づきポイント）

## ① エラーメッセージの差異
- 無効ユーザー → `Invalid username or password`
- 有効ユーザー（ロック後） → `You have made too many incorrect login attempts.`  
→ **レスポンスのLengthが明確に違う**

## ② ロックアウトの挙動が不自然
- 5回失敗後にロックメッセージが出る
- しかし **正しいパスワードを送ると 200 / 302 が返る**  
→ **ロックチェックより先にパスワード検証が走っている** と推測できる

## ③ Burp Intruderでの挙動が異常
- Null payload ×5 でロックを強制発動
- その後の Sniper 攻撃で **正しいパスワードだけエラーメッセージが空**  
→ ロックが効いていない証拠

---

# 🗡️ どう攻撃するのか（攻撃フロー）

## Phase 1：ユーザー名列挙
1. Cluster bomb  
2. username に候補リスト  
3. password の後ろに Null payload ×5  
4. **Length が長いレスポンス = ロックメッセージ = 有効ユーザー**

→ **ユーザー名が特定できる**

---

## Phase 2：パスワードブルートフォース
1. Sniper  
2. username を特定したものに固定  
3. password に候補リスト  
4. Grep Extract でエラーメッセージを抽出  
5. **エラーメッセージが空の行 = 正しいパスワード**

→ **パスワードが特定できる**

---

## Phase 3：ロック解除待ち → ログイン成功
- ロックは1分で解除される  
- 正しい認証情報でログイン成功  
- My account にアクセスして完了

---

# 🧩 何が問題なのか（根本原因）

## 🔴 **ロックアウトの実装順序が間違っている**
本来：
```
1. アカウントがロックされているか確認
2. ロックされていたら即エラー
3. パスワード検証
```

このラボ：
```
1. パスワード検証が先に走る
2. 正しいパスワードならロック無視でログイン成功
3. 間違っていたらロックメッセージ
```

→ **ロックアウトが意味をなしていない**

---

# 🛡️ どう対策すべきか（実務レベル）

## ✔️ 1. ロックアウトチェックを最優先にする
```
if (account.isLocked()) {
    return "Account locked";
}
```
**パスワード検証より前に必ず実行**

---

## ✔️ 2. エラーメッセージを統一する
- `Invalid username or password`
- ロック時も同じメッセージを返す  
→ **ユーザー名列挙を防ぐ**

---

## ✔️ 3. IPベースのレートリミット
- `/login` への POST を IPごとに制限  
- Cloudflare / Nginx / WAF で実装可能

---

## ✔️ 4. CAPTCHA の導入
- ロック直前の段階で CAPTCHA を要求  
- 自動化攻撃を抑止

---

## ✔️ 5. ログ監視 & アラート
- 連続ログイン失敗  
- 同一IPからの大量試行  
→ SOC に通知

---

# 🎤 **30秒面接用まとめ（あなたの声で言える形）**

```
今回の脆弱性は、ユーザー名列挙とロックアウト回避が組み合わさった
認証ロジックの欠陥です。ロックアウトのチェックがパスワード検証より
後に実行されているため、アカウントがロックされていても正しい
パスワードを送るとログインできてしまいます。

攻撃者はまずエラーメッセージの差異を利用して有効ユーザー名を特定し、
次にロック状態でもパスワードを総当たりして正しいパスワードを抽出します。

対策としては、ロックアウト判定を最優先に実行すること、
エラーメッセージを統一して列挙を防ぐこと、
レートリミットやCAPTCHAを組み合わせることが重要です。
```

