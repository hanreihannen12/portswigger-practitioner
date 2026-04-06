XSSを理解・攻撃・防御・実戦する
│
├── 1. どこに入力が反映されるか？（反射点の特定）
│       ├── HTML
│       ├── 属性
│       ├── JS文字列
│       ├── URL
│       └── DOM操作
│
├── 2. どのコンテキストか？（攻撃パターン分岐）
│       ├── HTMLコンテキスト
│       ├── 属性コンテキスト
│       ├── JSコンテキスト
│       ├── URLコンテキスト
│       └── コメント/テンプレート
│
├── 3. どのsinkに到達するか？（DOM XSS）
│       ├── innerHTML
│       ├── outerHTML
│       ├── document.write
│       ├── eval
│       ├── setTimeout
│       ├── setInterval
│       └── srcdoc
│
├── 4. どのsourceから来ているか？
│       ├── location.search
│       ├── location.hash
│       ├── document.cookie
│       └── postMessage
│
├── 5. フィルタ・サニタイザはあるか？
│       ├── HTMLエンティティ化
│       ├── タグ除去
│       ├── 属性除去
│       ├── イベント除去
│       ├── WAF
│       └── bypass可能か？
│               ├── タグ分割
│               ├── エンティティ
│               ├── SVG
│               ├── data:URL
│               └── JSプロトコル
│
├── 6. 実行可能なpayloadは何か？
│       ├── <script>
│       ├── <img onerror>
│       ├── <svg/onload>
│       ├── 属性break
│       ├── JS break
│       └── DOM-based payload
│
├── 7. 実行後に何ができるか？（攻撃チェーン）
│       ├── Cookie Theft
│       ├── Session Hijack
│       ├── CSRF token奪取
│       ├── 権限操作
│       ├── Keylogger
│       └── 管理画面XSS → 権限昇格
│
├── 8. 防御はどうなっているか？（実務で必須）
│       ├── エスケープ（HTML/JS/URL/CSS）
│       ├── CSP
│       ├── HttpOnly
│       ├── SameSite
│       ├── Sanitizer（DOMPurify）
│       └── フレームワークの自動エスケープ
│
├── 9. 実装側の理解（開発者視点）
│       ├── テンプレートエンジンの仕様
│       ├── React/Vue/AngularのXSS対策
│       ├── SSR/CSRの違い
│       ├── Markdown → HTML変換
│       └── WYSIWYGエディタの危険性
│
├── 10. 調査手法（どう見つけるか）
│       ├── Burp（Param Miner / DOM Invader）
│       ├── DevTools（DOM Breakpoints）
│       ├── JSコード読解
│       ├── レスポンス差分分析
│       └── 入力反映ポイントの列挙
│
├── 11. ハンズオン（手を動かす）
│       ├── PortSwigger全XSSラボ
│       ├── DVWA / Juice Shop
│       ├── CTF（picoCTF / RootMe）
│       └── Writeup分析
│
└── 12. 実戦（バグバウンティ）
        ├── XSSが起きやすい場所を優先
        │       ├── プロフィール編集
        │       ├── コメント欄
        │       ├── 管理画面
        │       ├── Markdown
        │       └── JSON → innerHTML
        ├── 再現性のあるテンプレ化
        └── レポート作成
