
## 📝 実際の手順

### ステップ1: 商品ページに移動

1. ブラウザで商品一覧ページが表示される  
2. どれでもいいので **「View details」** ボタンをクリック  
3. 対象商品の詳細ページに移動する  

---

### ステップ2: 脆弱性の場所を確認

1. ページを下にスクロール  
2. 「**Check stock**」ボタンと、  
   ロケーション選択のドロップダウン（London / Paris / Milan）が見える  
3. ドロップダウンメニューを **右クリック → 「Inspect」/「要素を検証」**  
4. DevTools が開く  

---

### ステップ3: URLを変更してテスト

1. ブラウザのアドレスバーを見る  
2. 現在のURL：

   ```text
   https://YOUR-LAB-ID.web-security-academy.net/product?productId=1
   ```

3. このURLの末尾に以下を追加：

   ```text
   &storeId=test
   ```

4. 完成したURL：

   ```text
   https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&storeId=test
   ```

5. **Enterキー** を押してアクセス  

---

### ステップ4: 結果を確認

- ページがリロードされる  
- ドロップダウンメニューを見る  
- 一番上に **「test」** が追加されているのが確認できる ✅  

---

### ステップ5: XSS攻撃実行

1. 再びアドレスバーをクリック  
2. URLを次のように変更：

   ```text
   https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&storeId=test</select><img src="0" onerror="alert(1)">
   ```

3. **Enterキー** を押す  

---

### ステップ6: 成功!

- `alert(1)` のポップアップウィンドウが表示される  
- 「OK」をクリック  
- ラボが自動的に **Solved** になる 🎉  

---

## 🎯 何が起こったか（ざっくり）

- `storeId` パラメータの値が、そのまま **ドロップダウンのHTMLに埋め込まれている**  
- `test</select><img src="0" onerror="alert(1)">` によって  
  - `</select>` でセレクトボックスを強制終了  
  - `<img ... onerror="alert(1)">` で任意のJS実行  
- 結果として **DOM上でXSSが成立** した、という流れ。



---

# 🔥 どーいう脆弱性？
**DOMベースXSS（DOM XSS）**。

具体的には：

- `storeId` パラメータが URL から取得され  
- その値が **サーバーではなくクライアント側の JavaScript** によって  
- `<select>` タグの中に **エスケープなしで直接書き込まれている**

つまり：

```
<select>
  <option>London</option>
  <option>Paris</option>
  <option>Milan</option>
  <option>{攻撃者入力}</option>
</select>
```

攻撃者が `</select>` を入れたら **HTML構造を壊して外に出られる** → XSS成立。

---

# 🔍 どう気づいた？


1. 商品詳細ページを開く  
2. URL に `&storeId=test` を付けると  
3. **ドロップダウンの一番上に test が追加される**  
4. つまり storeId の値が **DOM にそのまま挿入されている** と判明  
5. `<select>` の中に入っているので、  
   `</select>` で抜けられると確信  
6. → XSS 可能と判断

---

# 💥 どう攻撃した？
攻撃の本質は：

> **`</select>` でタグを閉じて、任意のタグを挿入する**

使ったペイロード：

```
test</select><img src=0 onerror=alert(1)>
```

URL 全体：

```
https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&storeId=test</select><img src="0" onerror="alert(1)">
```

ブラウザ側での最終的なDOMはこうなる：

```html
<select>
  <option>test</option>
</select>
<img src="0" onerror="alert(1)">
```

→ `<img>` の onerror が実行されて XSS 成立。

---

# 🛡️ どう対策する？
DOM XSS の対策は明確。

### ✔ 1. ユーザー入力を HTML として扱わない
- `innerHTML` や `document.write` を使わない  
- `textContent` / `innerText` を使う

### ✔ 2. select 要素に値を入れるときは createElement を使う
```js
const opt = document.createElement("option");
opt.textContent = storeId;
select.appendChild(opt);
```

### ✔ 3. URL パラメータを直接 DOM に入れない
- 必ずサニタイズ  
- もしくはサーバー側で安全な値だけ返す

### ✔ 4. HTML構造を壊せる文字（`< > " '`）をエスケープする

---

# 🎤 まとめ

> **この脆弱性は、URLパラメータの `storeId` をクライアント側のJavaScriptがエスケープせずに `<select>` 要素へ挿入しているため発生するDOMベースXSSです。**  
> URLに `&storeId=test` を追加するとドロップダウンにそのまま表示されたため、`</select>` でタグを閉じて `<img onerror=alert(1)>` を挿入し、XSSを成立させました。  
> 対策としては、`document.write` や `innerHTML` を使わず、`textContent` や `createElement` を使ってDOM操作すること、そして外部入力をHTMLとして扱わないことが重要です。

---
