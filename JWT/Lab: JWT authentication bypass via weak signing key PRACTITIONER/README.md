

### 1. Burp で JWT を取る（ここまではできてる）

1. **`wiener:peter` でログイン**
2. **Proxy → HTTP history** で `/my-account` を開く
3. リクエストヘッダの `Cookie` を見る：

   ```http
   Cookie: session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ3aWVuZXIifQ.x6BLkxlHWd0fL2u790Rt4sBNTrHh0qw_8u_X3cWzT68
   ```

4. `session=` の **右側全部** をコピーする：

   ```text
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ3aWVuZXIifQ.x6BLkxlHWd0fL2u790Rt4sBNTrHh0qw_8u_X3cWzT68
   ```

ここからが hashcat パート。

---

### 2. JWT をファイルに保存する（jwt.hash）

1. 適当にフォルダを作る（例）：

   ```text
   C:\jwt-lab\
   ```

2. その中に `jwt.hash` というファイルを作る
3. 中身を **さっきコピーした JWT をそのまま1行** だけ書く：

   ```text
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ3aWVuZXIifQ.x6BLkxlHWd0fL2u790Rt4sBNTrHh0qw_8u_X3cWzT68
   ```

※デコードしない・分割しない・改行入れない。

---

### 3. ワードリストを用意する（jwt.secrets.list）

1. 同じ `C:\jwt-lab\` に `jwt.secrets.list` を作る
2. とりあえずテスト用にこう書いておく：

   ```text
   password
   123456
   secret
   secret1
   admin
   ```

実際のラボではもっと長いリストが配られてるはずだけど、仕組み理解ならこれで十分。

---

### 4. PowerShell を開いて hashcat のフォルダへ移動

1. Windowsキー →「powershell」→ **Windows PowerShell** を開く
2. hashcat を解凍した場所に移動する（例）：

   ```powershell
   cd "C:\Users\あなたの名前\Downloads\hashcat-6.2.6"
   ```

ここで `.\hashcat.exe` が見えてる状態にする。

---

### 5. hashcat を実行するコマンド

PowerShell でこう打つ：

```powershell
.\hashcat.exe -m 16500 -a 0 C:\jwt-lab\jwt.hash C:\jwt-lab\jwt.secrets.list
```

- **`-m 16500`** → JWT HS256 モード
- **`-a 0`** → 辞書攻撃
- `C:\jwt-lab\jwt.hash` → 割りたい JWT
- `C:\jwt-lab\jwt.secrets.list` → 試す秘密鍵候補

---

### 6. 結果の確認

うまくいくと、実行中か `--show` でこう出る：

```powershell
.\hashcat.exe -m 16500 --show C:\jwt-lab\jwt.hash
```

出力例：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9....:secret1
```

この `secret1` が「見つかった秘密鍵」。

---

### 7. あとは今日やった通り Burp に戻る

1. `secret1` を Base64 → `c2VjcmV0MQ==`
2. JWT Editor → Keys → New Symmetric Key → `k` を `c2VjcmV0MQ==` に書き換え
3. `"sub": "wiener"` → `"sub": "administrator"` にして Sign → Send

---

ここまでが **「Burp で JWT コピーしたあと、実際に hashcat を叩いて secret1 を見つけるまで」** の一本の線。
<br>
<br>
<br>
<br>

一気に“面接でそのまま使える形”までまとめるね。  
和志の理解レベルなら、技術的にも現実的にも説得力ある説明ができる。

---

## 🔐 どんな脆弱性か（本質）
**JWTの署名に使われている秘密鍵が弱すぎて、総当たりで割れてしまう脆弱性。**

- JWTは「ヘッダー.ペイロード.署名」で構成される  
- 署名は **秘密鍵（HS256など）** を使って作られる  
- 秘密鍵が `secret1` のような弱い文字列だと、  
  **攻撃者が手元で総当たりして秘密鍵を特定できる**

結果として：

> **攻撃者が自分で好きな内容のJWTを作り、正規ユーザーや管理者になりすませる**

---

## 🔍 どう気づくか（攻撃者視点の思考プロセス）
1. **CookieにJWTが入っているのを確認**  
   `eyJ…` で始まる長い文字列 → JWT と判断。

2. **JWTをデコードして中身を確認**  
   Base64なので誰でも読める。  
   `"sub": "wiener"` などのユーザー情報が見える。

3. **署名アルゴリズムがHS256（共通鍵方式）であることを確認**  
   → 秘密鍵が弱いと危険。

4. **署名を検証するために秘密鍵を総当たり**  
   hashcat や jwtcrack などで  
   「どの鍵で署名したら一致するか？」を試す。

5. **秘密鍵が割れたら改ざん可能と判断**

---

## 💥 どう攻撃するか（実際の攻撃フロー）
1. JWTをデコードしてペイロードを改ざん  
   `"sub": "wiener"` → `"sub": "administrator"`

2. 割れた秘密鍵（例：`secret1`）で再署名

3. 改ざんJWTをCookieにセットして送信

4. サーバーは署名が正しいと思い込み、  
   **攻撃者を管理者として扱ってしまう**

---

## 🛡 どう対策するか（現実の開発・運用で必要なこと）
- **強力な秘密鍵を使う（最低32バイト以上のランダム値）**
- **秘密鍵を定期的にローテーション**
- **HS256ではなくRS256（公開鍵暗号方式）を採用**  
  → 秘密鍵はサーバーだけが持ち、公開鍵は漏れても安全
- **JWTに重要な権限情報を入れない**  
  → 権限はサーバー側で再チェック
- **JWTの有効期限を短くする（数分〜数十分）**
- **署名検証を必ず強制（alg=none攻撃を防ぐ）**

---

## 🧩 30秒で話せる面接用まとめ（暗記用）
> **「JWTの署名鍵が弱いことで発生する脆弱性です。JWTはHS256などの共通鍵方式を使う場合、秘密鍵が短いと総当たりで割られてしまいます。攻撃者は秘密鍵を特定すると、ペイロードを書き換えて管理者ユーザーのJWTを自分で作成し、正規の署名としてサーバーに受け入れられてしまいます。対策としては、十分に長いランダムな秘密鍵の使用、鍵のローテーション、RS256のような公開鍵方式への移行、権限情報をJWTに直接入れないことなどが重要です。」**

---



🎯 まず答えだけ言うとこうなる

✔ signature は

「header + payload を秘密鍵で署名したもの」

✔ JWT 全体は

header.payload.signature の3つでできてる

つまり、

JWT = header + "." + payload + "." + signature


---

🔥 じゃあ、signature と JWT の関係を超シンプルに言うと？

✔ signature は「答え」

✔ JWT は「問題 + 答えのセット」

こう考えると一気にわかる。

---

🍎 図で見ると一瞬で理解できる

▼ JWT の構造

header      payload        signature
   |           |               |
   |           |               |
   |           |         HMAC(header + payload, 秘密鍵)
   |           |               |
   +-----------+---------------+
           これ全部で JWT


つまり、

👉 JWT = (header + payload) + (それを秘密鍵で署名した signature)

---

🔥 あなたの質問に直接答える

signatureはヘッダーとペイロードと秘密鍵でできてるんやんな？

→ その通り。100%正しい。

signature = HMAC(header + payload, secret_key)


---

jwtはなにでできてるん？

→ JWT は “header + payload + signature” の3つでできてる。

JWT = header.payload.signature


---

🧠 じゃあ攻撃者は何を総当たりしてるの？

ここが混乱ポイントだったよね。

❌ 署名そのものを総当たりしてる

→ これは違う

✔ 秘密鍵を総当たりしてる

→ これが正しい

理由：

• 秘密鍵が当たれば
header + payload を使って同じ signature が作れる
• だから攻撃者は
秘密鍵候補を1つずつ試して、署名が一致するか確認してる


---

🍀 もっと噛み砕くとこう

署名は「答え」

秘密鍵は「答えを作るための道具」

攻撃者は

「この道具（秘密鍵）を使ったら同じ答え（署名）が作れるか？」

を試してるだけ。

---

🌟 まとめ（超シンプル版）

• signature は
header + payload を秘密鍵で署名したもの
• JWT は
header.payload.signature の3つでできてる
• hashcat が総当たりしてるのは
署名ではなく秘密鍵
• 秘密鍵が当たれば
同じ signature を作れる → JWT を偽造できる
---






# 🔥 結論：signature を「合わせる」とは  
**“同じ材料（header + payload + 秘密鍵）を使って、同じ計算をして、同じ結果を作ること”** だよ。

もっと噛み砕くと：

> **サーバーが作った署名と、攻撃者が手元で作った署名が一致する状態にすること**

これが「signature を合わせる」という意味。

---

# 🧩 どうやって一致させるのか（仕組みだけ説明）

JWT の署名はこう作られる：

\[
signature = HMAC(header + payload, secret\_key)
\]

つまり **秘密鍵が同じなら、誰が計算しても同じ signature ができる**。

---

## ✔ ステップの流れ（安全に説明できる範囲）

1. **header と payload は誰でも読める**  
   Base64 だから Decoder で見える。

2. **攻撃者は秘密鍵を推測する（弱いと割れる）**

3. **推測した秘密鍵を使って、同じ計算をする**  
   HMAC(header + payload, secret_key)

4. **サーバーの署名と一致したら「鍵が当たった」状態になる**

5. その鍵を使えば、改ざんした payload でも  
   **“正しい署名” を作れるようになる**

これが「signature を合わせる」ということ。

---

# 🎨 イメージで言うと

- サーバーが押したスタンプ（署名）がある  
- そのスタンプを押したハンコ（秘密鍵）が弱いと、同じハンコを手に入れられる  
- 同じハンコを持っていれば、  
  **自分で作った紙にも“本物と同じスタンプ”を押せる**

だから signature が一致する。

---

# 🌟 一言でまとめると

**秘密鍵が同じなら、同じ署名が作れる。  
だから秘密鍵を当てると signature を“合わせられる”。**

---


  
**「計算するって何？」**  
ここを理解すると JWT の署名が一気に腹落ちする。

結論から言うと：

---

# 🔥 「計算する」の正体は **ただの関数に数字を入れて答えを出すこと**

数学の「2 + 3 = 5」と同じ構造。

ただし JWT の場合は：

\[
signature = HMAC\_SHA256(header + "." + payload, secret\_key)
\]

これだけ。

---

# 🍎 もっと噛み砕く（超わかりやすい版）

### ✔ “計算”＝「材料を入れたら結果が出る機械に通す」だけ

- 材料①：header  
- 材料②：payload  
- 材料③：secret_key（秘密鍵）

この3つを **HMAC-SHA256 という機械** に入れる。

すると **signature（署名）** が出てくる。

---

# 🎮 ゲームで例えると

「名前 + レベル + 秘密のパスワード」を入力すると  
ゲームが「あなたのキャラID」を計算して出してくれる。

同じ入力なら、誰がやっても同じキャラIDになる。

JWT も同じ。

---

# 🧪 実験イメージ（危険じゃない範囲で説明）

```
HMAC("header.payload", "secret1")
→ x6BLkxlHWd0fL2u790Rt4sBNTrHh0qw_8u_X3cWzT68
```

これが signature。

つまり **秘密鍵が同じなら、誰が計算しても同じ signature が出る**。

---

# 🧠 だから攻撃者はこう考える

「この署名を作れる秘密鍵はどれや？」

→ 候補の鍵を1つずつ入れて **同じ signature が出るか確認する**  
→ 出たら「鍵が当たった！」

これが「計算して signature を合わせる」という意味。

---

# 🌟 一言でまとめると

**“計算する”とは、  
header + payload + 秘密鍵 を HMAC という関数に入れて  
signature を作ること。**

数学の「2 + 3 = 5」と同じで、  
ただの **入力 → 出力** の仕組み。

---

和志、ここまででまだ引っかかる部分ある？  
「HMACって何？」とか「なんで同じ結果になるん？」とか、  
どこでも深掘りできるよ。
