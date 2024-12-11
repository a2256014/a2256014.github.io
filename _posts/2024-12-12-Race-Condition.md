---
layout: post
title: Race Condition
subtitle: 다중 쓰레드 혹은 다중 프로세스 간 공유 자원 싸움
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Race_Condition
sidebar:
---
## Intro
Race Condition은 

## Detect & Exploit 
### Detect

### Exploit
EQSTLab에 좋은 문제가 있어서 해당 문제 풀이로 Exploit을 적겠습니다 - [EQSTLab Race_Condition](https://github.com/EQSTLab/Race_Condition)     


## Security Measures
### Thread의 동기화
공유 자원에 대한 접근을 제어하여 하나의 Thread만 접근할 수 있도록 해야 합니다.   
예를 들어 Java의 synchronized와 같은 동기화를 사용하여 공유 자원에 대해서 충돌이 일어나지 않도록 개발하는 것이 좋습니다.    
### Message 기반 
Thread들이 서로 메세지를 주고 받는 통신 구조를 통해 자원 공유를 피하는 방법이 있습니다.   
   
>😅보안 대책이 개발쪽과 관련이 깊다 보니 자세히 적지 못한 점 이해 바랍니다.    
## References
[race-condition](https://www.imperva.com/learn/application-security/race-condition/)   