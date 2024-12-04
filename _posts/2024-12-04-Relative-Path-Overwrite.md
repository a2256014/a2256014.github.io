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
  - Path_Traversal
  - Protocol-relative
sidebar:
---

## Intro
Relative Path Overwrite(RPO)란 해석 그대로 "상대 경로 덮어쓰기"라는 공격 기법? 으로 상대 경로로 된 URL을 덮어씌워 의도치 않는 경로로 요청하게 하거나, Host를 바꾸는 등의 여러가지 파생된 취약점들의 근간이 되는 녀석입니다.
> 대표적으로 활용되는 취약점으로 Path Traversal, Protocol relative URL 가 있으며, 특수한 상황에서는 DOM Clobbering, [CSS Injection](./CSS-Injection.html) 등 다양한 취약점에서도 활용될 수 있습니다.

상대 경로란 URL에 `Host`가 포함되지 않은 경로로 `assets/js/config.js` 혹은 `./assets/js/config.js` 등의 경로입니다.
> Absolute URL : `https://victim.com/path`
> Absolute Path : `/path`
> 

## Detect & Exploit 
### Detect

### Exploit



## Security Measures

단순하게 상대경로를 사용하지 않는 것이지만, 추가적으로 `<script src>`, `<link href>` 등 리소스를 불러오는 태그의 주소를 사용자가 임의로 수정하지 못하도록 하는 방안도 있습니다.

## References
