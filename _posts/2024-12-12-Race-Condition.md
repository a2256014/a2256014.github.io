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
Race Condition은 다중 프로세스 혹은 쓰레드가 하나의 공유 자원에 대한 접근 및 수정을 동시에 해서 한 쪽 결과가 무시 당하거나, 잘못된 결과가 도출되는 취약점입니다.   
![](assets/images/posts/2024-12-12-Race-Condition/4a5981ed1eef80144ef6c9deabb8240b_MD5.jpeg)   
## Detect & Exploit 
### Detect
사실 진단자 입장에서는 눈으로 확인할 수는 없을 것 같습니다. 제 경험에 의하면 주로 복수 이벤트 참여, 비밀번호 오류 횟수 초과 등과 같이 횟수에 의존하는 로직들에서 주로 발생했던 것 같습니다.   
   
>☑️코드를 보지 않고 발견하려면 많은 경험에 의한 혹은 Race Condition이 발생한 CVE 등을 본 지식에 의한 감각적인 요소일 것 같습니다. - 아~ 이럴 때 발생할 수 있겠구나🤣    

### Exploit
EQSTLab에 좋은 문제가 있어서 해당 문제 풀이로 Exploit을 적겠습니다 - [EQSTLab Race_Condition](https://github.com/EQSTLab/Race_Condition)     
간략하게 코드에 대한 소개를 하겠습니다.   
1. 결제 요청 시 장바구니에 있는 물건들의 금액을 더한다.   
	```php
	# kcp_api_pay.php 13 line
	$stmt = $conn->prepare("SELECT sum(good_mny) AS total FROM orders WHERE buyr_name = ?");
	```   
2. 결제 방법에 따른 결제를 한 뒤 쇼핑몰 업체는 자신의 DB에 유저가 산 물건에 대한 정보를 장바구니에서 얻는다   
	```php
	# kcp_api_pay.php 249 line (RACE CONDITION POINT)
	 $stmt = $conn->prepare("UPDATE payments SET pay_method = ?, tno = ?, amount = (SELECT sum(good_mny) FROM orders WHERE buyr_name = ? ) WHERE buyr_name = ? ");
	```   

---
>☑️결제 시 KCP와 쇼핑몰에서 각각 장바구니를 조회하여 저장하는 로직인데, 여기서 공격 포인트가 생성됩니다.    




>⚠️ 해당 취약점으로 발생할 수 있는 문제들이 다양하기에 제가 가져온 문제 외 다양한 결과를 도출할 수 있습니다. 하지만, 공격 포인트 자체는 동일합니다.(**동시에 자원에 접근하는 지점을 찾는 것**)   

## Security Measures
### Thread의 동기화
공유 자원에 대한 접근을 제어하여 하나의 Thread만 접근할 수 있도록 해야 합니다.   
예를 들어 Java의 synchronized와 같은 동기화를 사용하여 공유 자원에 대해서 충돌이 일어나지 않도록 개발하는 것이 좋습니다.    
### Message 기반 
Thread들이 서로 메세지를 주고 받는 통신 구조를 통해 자원 공유를 피하는 방법이 있습니다.   
   
>😅보안 대책이 개발쪽과 관련이 깊다 보니 자세히 적지 못한 점 이해 바랍니다.

## References
[race-condition](https://www.imperva.com/learn/application-security/race-condition/)   
