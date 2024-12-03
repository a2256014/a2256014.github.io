---
layout: post
title: S3 Bucket Takeover
subtitle: Subdomain Takeover 취약점 사례
author: g3rm
categories: 
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - WEB
  - AWS
  - S3
  - CNAME
sidebar:
---


## Intro
S3 Bucket Takeover 취약점은 Subdomain Takeover 취약점이 Cloud 서비스가 많은 현재 상황에 맞게 적용되어 나온 취약점으로 이름만 다를 뿐 같은 취약점이라고 봐도 무방합니다.   

주로 서비스의 Subdomain에 S3 Bucket 주소가 매핑되어 있거나, 서비스 내에서 S3 Bucket 주소를 사용하고 있을 때 확인해볼 만한 취약점입니다.    

resource.victim.com 도메인이 AWS에서 제공한 S3 엔드포인트 주소와 매핑되어 있고 victim이 해당 S3

☑️*S3뿐만 아니라 Github Page 등 호스팅을 해주는 서비스 이용 시 자주 발생합니다.*   
☑️*S3의 잘못된 권한 설정으로 사용자가 버킷의 내용을 읽고, 쓰는 것에 대한 부분도 해당 취약점에 포함될 수 있으나, 여기선 제외하겠습니다.*
## Detect & Exploit 



## Security Measures

## References
