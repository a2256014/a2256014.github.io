---
layout: post
title: Command Injection
subtitle: Command Injection
author: g3rm
categories: 
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - example
  - markdown
sidebar:
---
## Intro
사용자가 입력 값 조작을 통해 원하는 시스템 명령어를 실행 시킬 수 있는 취약점으로 시스템을 ㅇ  

## Detect & Exploit 
### Detect
서버측 코드에서 아래와 같은 함수를 사용하면 사용자 입력 값이 시스템 명령어로 활용될 수 있다.

|언어|함수|
|---|---|
|Java|System.*, 특히 System.runtime 취약, Runtime.exec()|
|C/C++|system(), exec(), ShellExecute()|
|python|exec(), eval(), os.system(), os.popen(), subprocess.popen(), subprocess.call()|
|Perl|open(), sysopen(), system(), glob()|
|php|exec(), system(), passthru(), popen(), rquire(), include(), eval(), preg_replace(), shell_exec(), proc_open(), eval()|

사용자 입력 값이 들어가는 부분이 명령어 뒷 부분이라면 아래 메터문자를 통해 체인을 만들어 한 줄로 여러 명령어를 실행할 수 있다.

| 메타문자 | 설명                                                                                             |
| ---- | ---------------------------------------------------------------------------------------------- |
| ``   | **명령어 치환** `` 안에 들어있는 명령어를 실행한 결과로 치환                                                          |
| $()  | **명령어 치환** `$()` 안에 들어있는 명령어를 실행한 결과로 치환, 중복 사용 가능                                             |
| &&   | **명령어 연속 실행** 한 줄에 여러 명령어를 사용하고 싶을 때 사용. 앞 명령어에서 에러가 발생하지 않아야 뒷 명령어 실행                         |
| \|   | **명령어 연속 실행** 한 줄에 여러 명령어를 사용하고 싶을 때 사용. 앞 명령어에서 에러가 발생해야 뒷 명령어 실행                             |
| ;    | **명령어 구분자** 한 줄에 여러 명령어를 사용하고 싶을 때 사용. `;` 은 단순히 명령어 구분을 위해 사용하며, 앞 명령어의 에러 유무와 관계 없이 뒷 명령어 실행 |
| \|   | **파이프** 앞 명령어 결과가 뒷 명령어 입력으로 들어간다.                                                             |
| 그 외  | `.`, >, >>, &>, >&, <, {}, ?, *, ~                                                             |

### Exploit

#### Bypass
https://sonarsource.github.io/argument-injection-vectors/

인수 주입 벡터로 만약 서버측에서 보안조치를 `& # ; ' | * ? ~ < > ^ ( ) [ ] { } $ \ , \x0A \xFF` 와 같은 특수문자로만 했다면, 명령어 옵션을 활용하여 우회할 수 있는 기법이다.

## Security Measures