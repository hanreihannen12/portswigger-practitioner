# daily-bugle-ctf-notes

root@ip-10-49-190-46:~# nmap --min-rate 5000 -T4 10.49.190.46
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




root@ip-10-49-107-31:~# gobuster dir -u http://10.49.190.46 -w /usr/share/wordlists/dirb/common.txt  

Gobuster v3.6  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  

[+] Url:                     http://10.49.134.85  
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
