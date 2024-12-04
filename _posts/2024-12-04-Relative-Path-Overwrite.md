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
리소스(`<script src>`, `<link href>` 등) 주소

### Exploit



## Security Measures

단순하게 상대경로를 사용하지 않는 것이지만, 추가적으로 `<script src>`, `<link href>` 등 리소스를 불러오는 태그의 주소를 사용자가 임의로 수정하지 못하도록 하는 방안도 있습니다.

## References
