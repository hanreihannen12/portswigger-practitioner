

## 📍 完全ステップバイステップガイド

### **Step 1: ラボを開く**
1. PortSwiggerのラボページで **"ACCESS THE LAB"** ボタンをクリック
2. 新しいタブでラボサイトが開く

---

### **Step 2: ラボサイトで検索機能を探す**
1. ページ上部に**検索ボックス**(Search box)がある
2. とりあえず `test` と入力して検索ボタン(虫眼鏡アイコン)をクリック

---

### **Step 3: DevToolsで調査**
1. 検索ボックスの上で **右クリック**
2. メニューから **"検証"** または **"Inspect"** をクリック
3. 画面下半分にDevToolsが開く

---

### **Step 4: JavaScriptコードを確認**
DevToolsの中で:
1. **"Elements"タブ**を見る
2. `Ctrl+F` (Mac: `Cmd+F`)を押す
3. 検索窓に `document.write` と入力
4. こんなコードが見つかる:
```javascript
document.write('<img src="/resources/images/tracker.gif?searchTerms='+query+'">');
```
これが脆弱なコード!

---

### **Step 5: 攻撃ペイロードを入力**

**方法1: 検索ボックスから**
1. 検索ボックスをクリック
2. 中身を全削除
3. 以下をコピペ:
```
"><svg onload=alert(1)>
```
4. 検索ボタン(虫眼鏡)をクリック

**方法2: URLから直接(こっちが確実)**
1. ブラウザのアドレスバーを見る
2. URLの最後が `?search=test` みたいになってる
3. その部分を選択して削除
4. 以下を追加:
```
?search="><svg onload=alert(1)>
```
5. **Enter**キーを押す

---

### **Step 6: アラートが出る**
- `alert(1)` のポップアップが表示される
- **"OK"** をクリック

---

### **Step 7: ラボ完了確認**
1. ラボページ(最初のタブ)に戻る
2. 緑のバナーで **"Congratulations, you solved the lab!"** と表示される
3. または画面上部が **"LAB SOLVED"** になる

---


### 🧨 どーいう脆弱性？

**DOMベースXSS（DOM XSS）**。  
具体的には：

- `location.search`（URLのクエリ）からユーザー入力を取ってきて  
- それを **エスケープせずにそのまま `document.write()` に突っ込んでいる**

```javascript
document.write('<img src="/resources/images/tracker.gif?searchTerms='+query+'">');
```

ここで `query` が攻撃者コントロールなので、  
`">...` みたいに **HTML構造を壊して任意のタグを差し込める** → XSS成立。

---

### 🔍 どう気づいてる？

1. 検索機能を適当に使ってみる（`test` とか）  
2. DevTools で `Inspect` → `Ctrl+F` で `document.write` を検索  
3. `location.search` 由来の値が `document.write()` に直結しているコードを発見  
4. さらに、検索語がレスポンスHTMLではなく **クライアント側JSで処理されている** ので  
   → これは **DOM XSS のパターン** と判断

---

### 💥 どう攻撃してる？

攻撃の本質は：

> `document.write('<img ... '+query+'">')`  
> の `query` に `">` で抜けて新しいタグを差し込む。

だからペイロードは：

```text
"><svg onload=alert(1)>
```

URLとしては：

```text
?search="><svg onload=alert(1)>
```

ブラウザ側での流れ：

1. `location.search` から `search` パラメータを取得  
2. そのまま `document.write()` に連結  
3. 結果として HTML がこうなる：

```html
<img src="/resources/images/tracker.gif?searchTerms="><svg onload=alert(1)>
```

4. `<svg onload=alert(1)>` がブラウザで実行 → XSS成立

---

### 🛡️ どう対策する？

**DOM XSS 対策の基本セット：**

- **`document.write()` を使わない**
  - 必要なら `textContent` / `innerText` / `setAttribute` など、  
    「HTMLとして解釈されない」APIを使う
- どうしてもHTMLを生成するなら：
  - 信頼できない入力は **エスケープしてから**埋め込む
  - もしくはテンプレートエンジンやフレームワークのサニタイズ機能を使う
- `location.search` や `document.URL` など **完全に外部入力な値を、そのままDOMに突っ込まない**

---

### 🎤 30秒面接用まとめ

> **このラボは、URLのクエリパラメータを `location.search` から取得して、そのまま `document.write()` に連結しているために発生するDOMベースXSSです。**  
> DevToolsでコードを確認すると、検索語がサーバー側ではなくクライアント側の `document.write` でHTMLに埋め込まれているのを見つけました。そこで、`?search="><svg onload=alert(1)>` のようにHTML構造を壊すペイロードを入れることで、任意のタグを挿入してXSSを成立させました。  
> 対策としては、`document.write` の使用を避け、`textContent` などの安全なAPIを使うこと、また `location.search` などの外部入力をHTMLとして直接扱わないことが重要です。

