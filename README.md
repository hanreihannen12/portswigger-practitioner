# PortSwigger Practitioner 学習ログ

PortSwigger Web Security Academy のラボをカテゴリごとに整理し、
単なる解法ではなく「なぜそうなるのか」「実務でどう活きるか」をまとめています。

## 🔐 カテゴリ構成

- **Access Control**  
  メソッド・URL・Referer など実装依存のアクセス制御の破綻と、その見つけ方・攻撃パターン。

- **Authentification**  
  ブルートフォース、2FA ロジック破綻、クッキーの不備など、認証まわりの実践的な攻撃と防御。

- **Business logic vulnerabilities**  
  状態遷移の破綻や例外入力の扱いミスによる認可バイパスなど、仕様レベルの脆弱性。

- **CSRF**  
  トークンの紐付け不備、SameSite、メソッド依存など、実装パターンごとの CSRF 防御の崩れ方。

- **File upload vulnerabilities**  
  拡張子・Content-Type チェックの bypass など、ファイルアップロード機能の典型的な落とし穴。

- **JWT**  
  jku/jwk インジェクション、弱い署名鍵など、JWT の設計・実装上の問題を利用した認証回避。

- **SQL injection**  
  UNION / Blind / Time-based / OOB など、主要パターンを通じた DB 情報の抽出と攻撃チェーン。

- **XSS**  
  DOM / Reflected / Stored に加え、XSS を起点にした CSRF bypass や Cookie 盗難などの応用。

## 📌 代表的なまとめ（例）

- **Access Control / Method-based access control can be circumvented**  
  → HTTP メソッドごとの認可実装の差異を突いたアクセス制御 bypass。

- **Authentification / 2FA broken logic**  
  → 2FA の検証タイミングとセッション管理の不整合を利用した認証回避。

- **SQL injection / Blind SQL injection with time delays**  
  → レスポンス時間を利用した Blind SQLi による情報抽出の手順とパターン整理。

## 🧰 使用ツール・スタック

- Burp Suite, browser devtools
- 基本的な HTTP / Web アプリケーションの理解
- 各ラボごとに README で攻撃チェーン・学び・実務での教訓を記載
