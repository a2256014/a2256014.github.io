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

상대 경로란 URL에 `Host`가 포함되지 않은 경로로 `assets/js/config.js` 로 구성된 경로입니다.
> RPO에서 말하는 상대 / 절대 경로는 통상적으로 말하는 경로와 다를 수 있습니다.
> Absolute URL : `https://victim.com/path`
> Absolute Path : `/path`
> Relative Path : `path`

## Detect & Exploit 
### Detect
어떤 공격을 하느냐에 따라 탐지 방법이 여러개 있을 수 있습니다.
1. 리소스(`<script src>`, `<link href>` 등) 주소에 관여할 수 있는 지 파악하면 됩니다.
2. URL 라우터가 `victim.com/rpo.html`와  `victim.com/rpo.html/`, `victim.com/rpo.html/blah` 에 대해서 동일하게 `rpo.html` 를 내려주는 지 파악하면 됩니다.
> 2번 탐지의 경우 다른 취약점(DOM Clobbering, [CSS Injection](./CSS-Injection.html), Text Injection)에 대해 bypass 및 연계 용도로 사용됩니다. 

### Exploit



## Security Measures

단순하게 상대경로를 사용하지 않는 것이지만, 추가적으로 `<script src>`, `<link href>` 등 리소스를 불러오는 태그의 주소를 사용자가 임의로 수정하지 못하도록 하는 방안도 있습니다.

## References
[Large-Scale Analysis of Style Injection by Relative Path Overwrite](https://dl.acm.org/doi/fullHtml/10.1145/3178876.3186090)   
[Relative Path Overwrite](https://support.detectify.com/support/solutions/articles/48001048955-relative-path-overwrite)   