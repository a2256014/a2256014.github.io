---
layout: post
title: S3 Bucket Takeover
subtitle: Subdomain Takeover 취약점 사례
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
  - AWS
  - S3
  - CNAME
  - Subdomain_Takeover
sidebar:
---


## Intro
S3 Bucket Takeover 취약점은 Subdomain Takeover 취약점이 Cloud 서비스가 많은 현재 상황에 맞게 적용되어 나온 취약점으로 이름만 다를 뿐 같은 취약점이라고 봐도 무방합니다.   

주로 서비스의 Subdomain에 S3 Bucket 주소가 매핑되어 있거나, 서비스 내에서 S3 Bucket 주소를 사용하고 있을 때 확인해볼 만한 취약점입니다.    

resource.victim.com 도메인(`CNAME`)이 AWS에서 제공한 S3 엔드포인트 주소(`https://[bucket name].s3.[aws-region].amazonaws.com`)와 매핑되어 있고 victim이 해당 S3를 사용하지 않게 되어 삭제했을 때, 해당 취약점이 발생합니다.

☑️*S3뿐만 아니라 Github Page 등 호스팅을 해주는 서비스 이용 시 자주 발생합니다.*   
☑️*S3의 잘못된 권한 설정으로 사용자가 버킷의 내용을 읽고, 쓰는 것에 대한 부분도 해당 취약점에 포함될 수 있으나, 여기선 제외하겠습니다.*
## Detect & Exploit 
### Detect
탐지 방법은 아주 간단합니다. S3와 매핑된 특정 Subdomain 혹은 서비스 내 사용 중인 S3 Bucket 주소에 접근해서 응답값을 확인하면 됩니다.

![](/assets/images/posts/2024-12-03-S3-Bucket-Takeover/22008c437f300c90f30c783bebcccb46_MD5.jpeg)
**⚠️ 기본적으로 Bucket이 없다면 위와 같은 모습이지만, 취약점인 이유는 서비스에서 사용 중이기 때문입니다.**   

위와 같이 `The specified bucket does not exist` 이라면 해당 Bucket을 제어할 수 있게됩니다.
### Exploit
1. 취약한 S3 Bucket 명으로 S3 Bucket Public 생성 - ex) g3rm-bucket
2. Takeover 증적을 위한 html 파일 업로드
		![](/assets/images/posts/2024-12-03-S3-Bucket-Takeover/01be6ed49b61da66fa3a56d1545894d9_MD5.jpeg)
3. 해당 Bucket을 누구나 읽을 수 있도록 정책 설정
		![](/assets/images/posts/2024-12-03-S3-Bucket-Takeover/6f41f8c47959bdd5b49e8092132cacb9_MD5.jpeg)
4. 접근 시 공격자가 업로드한 파일이 노출됨.
		![](/assets/images/posts/2024-12-03-S3-Bucket-Takeover/e27de20cfed50b0ff2eee576b6486227_MD5.jpeg)
   
☑️만약 주소가 `https://[bucket name].s3-website.[aws-region].amazonaws.com`이라면 정적 웹 사이트 호스팅 설정을 해주시면 됩니다.  
## Security Measures
보통 해당 취약점은 테스트 용으로 사용했다가 S3 Bucket만 지우고 매핑된 CNAME이나 사용된 서비스에서는 삭제하지 않아서 발생합니다.   

S3 Bucket 삭제 시 매핑된 CNAME 및 서비스 내 사용중인 해당 Bucket 주소를 삭제해주면 됩니다.
## References
[Amazon S3 Bucket Takeover](https://satyasai1460.medium.com/amazon-s3-bucket-takeover-648ed9561ee7)