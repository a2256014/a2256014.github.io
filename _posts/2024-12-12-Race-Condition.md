---
layout: post
title: Race Condition
subtitle: 다중 쓰레드 혹은 프로세스 간 공유 자원 싸움
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
  - Last-Byte_Sync
  - Single_Packet_Attack
sidebar:
---
## Intro
Race Condition은 다중 프로세스 혹은 쓰레드가 하나의 공유 자원에 대한 접근 및 수정을 동시에 해서 한 쪽 결과가 무시 당하거나, 잘못된 결과가 도출되는 취약점입니다.   
![](assets/images/posts/2024-12-12-Race-Condition/4a5981ed1eef80144ef6c9deabb8240b_MD5.jpeg)   

### 공격 원리
#### HTTP/1 - Last-Byte Sync
HTTP/1의 경우 HTTP 요청이 완료 되어야 서버에서 요청을 처리하기 때문에 각 패킷의 Last-Byte을 남겨둔 채로 전송한 뒤 모았던 Last-Byte들을 한번에 보내 동시에 요청을 처리하도록 하게할 수 있습니다.   
![](/assets/images/posts/2024-12-12-Race-Condition/637bb68e80eecd3651050e9413eb5300_MD5.jpeg)   

#### HTTP/2 - Single Packet Attack
HTTP/2의 경우 하나의 패킷에 여러 개의 요청을 전송할 수 있기 때문에 아래와 같은 모습으로 한번에 요청을 보내버립니다.😄   

![](/assets/images/posts/2024-12-12-Race-Condition/f4a02b5957531e299cb0525831491fc9_MD5.jpeg)   

>☑️한번에 요청을 보내기 때문에 시간 차이가 짧아 HTTP/2가 HTTP/1보다 Race Condition을 더 잘 구현할 수 있게됩니다.    

## Detect & Exploit 
### Detect
#### Limit overrun
로그인 실패 횟수 제한, 일회성 할인 코드, reCAPTCHA 등 횟수 제한이 있는 로직의 경우 Race Condition을 시도할 Race Window가 존재할 가능성이 있습니다. - [Turbo Intruder](https://portswigger.net/bappstore/9abaa233088242e8be252cd4ff534988) 사용    

#### Single End Point
비밀번호 초기화, 이메일 인증 등과 같은 기능에서 발생할 수 있고, 해당 기능이 비정상적으로 빠르다면 Race Window가 존재할 가능성이 있습니다.   

그 이유는 만약에 이메일 인증의 경우 해당 이메일에 인증 코드가 담긴 메일을 보내게 되는데, 이러한 과정이 비정상적으로 빠르다면 이메일 인증 요청(패킷)을 처리하는 스레드와 메일을 보내는 스레드가 다르다는 것을 유추할 수 있기 때문입니다.   

예를 들어 아래와 같이 두 요청을 동시에 보내게 됐을 때, 인증 코드가 담긴 메일이 꼬일 수 있습니다.   
```
# single packet 1
POST /email-auth HTTP/2
Host: victim.com

email=g3rm@attacker.com

# single packet 2
POST /email-auth HTTP/2
Host: victim.com

email=g3rm@victim.com
```   

```
받는이 : g3rm@attacker.com
제목 : 이메일 인증 코드
내용 : g3rm@victim.com 이메일의 인증코드는 1234 입니다.
```   

#### Multi End Point
가령 쇼핑몰과 같은 서비스가 있고 해당 서비스의 구매 로직이 `지불확인` -> `장바구니 확인` 순서로 이루어져 있다면, 해당 과정 사이에 Race Window가 발생할 수 있게 됩니다.    
![](/assets/images/posts/2024-12-12-Race-Condition/26b41a0d3526c8f0ed4d6f98e64db2c6_MD5.jpeg)   

>☑️Race Condition은 제가 설명한 부분 외 자원에 동시에 접근하면 발생하기에 수많은 취약점을 도출할 수 있습니다. 물론 이것을 실제 상황에서 발견하는 건 감각적인 요소일 것 같습니다. - 뭔가 동시에 접근하는 로직일 거 같은데....?🤣    

### Exploit
EQSTLab에 좋은 문제가 있어서 해당 문제 풀이로 Exploit을 적겠습니다 - [EQSTLab Race_Condition](https://github.com/EQSTLab/Race_Condition)     
간략하게 코드에 대한 소개를 하겠습니다.   
1. 결제 완료 시 정상적으로 결제가 되었는지 장바구니에 있는 물건들의 금액을 더한다. **[지불확인]**   
	```php
	# kcp_api_pay.php 13 line
	$stmt = $conn->prepare("SELECT sum(good_mny) AS total FROM orders WHERE buyr_name = ?");
	```   
	
2. 결제가 된 것을 확인 후 자신의 DB에 유저가 산 물건에 대한 정보를 장바구니에서 얻는다. **[장바구니 확인]**    
	```php
	# kcp_api_pay.php 249 line (RACE CONDITION POINT)
	 $stmt = $conn->prepare("UPDATE payments SET pay_method = ?, tno = ?, amount = (SELECT sum(good_mny) FROM orders WHERE buyr_name = ? ) WHERE buyr_name = ? ");
	```   

>☑️결제 시, 정상적으로 결제가 되었는지 확인하는 로직과 확인 후 DB에 업데이트하는 로직에서 둘 다 장바구니를 확인[공유 자원 사용]하기에 공격 포인트가 생성됩니다.    

---

단순하게 결제 서비스를 통해 결제가 끝난 후 **결제를 확인하는 패킷**과 **장바구니에 물건을 담는 패킷**을 동시에 보내면 됩니다.   

>⚠️ php의 경우 session을 파일로 관리하기에 해당 파일에 대한 Race condition을 막기 위해 phplock이 존재합니다. 따라서 세션을 두 개 발급 받아서 사용하셔야 됩니다.   

1. 10원으로 결제가 완료된 모습
	![](/assets/images/posts/2024-12-12-Race-Condition/866c4291afd09a5d4f81ea8ff56a35ee_MD5.jpeg)   
2. DB에는 10000010원으로 물건 2개가 결제된 것으로 입력된 모습
	![](/assets/images/posts/2024-12-12-Race-Condition/6455dfec128512767972d686309e7b28_MD5.jpeg)   

## Security Measures
### Thread의 동기화
공유 자원에 대한 접근을 제어하여 하나의 Thread만 접근할 수 있도록 해야 합니다.   
예를 들어 Java의 synchronized와 같은 동기화를 사용하여 공유 자원에 대해서 충돌이 일어나지 않도록 개발하는 것이 좋습니다.    
### Message 기반 
Thread들이 서로 메세지를 주고 받는 통신 구조를 통해 자원 공유를 피하는 방법이 있습니다.   
   
>😅보안 대책이 개발쪽과 관련이 깊다 보니 자세히 적지 못한 점 이해 바랍니다.

## References
[race-condition](https://www.imperva.com/learn/application-security/race-condition/)   
[PortSwigger Race conditions](https://portswigger.net/web-security/race-conditions)   
[PortSwigger Research](https://portswigger.net/research/smashing-the-state-machine)   