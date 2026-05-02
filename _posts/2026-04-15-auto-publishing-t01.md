---
title: "[Auto-Publishing] 쿠팡 크롤링 403 Access Denied 완전 해부 — Cloudflare WAF가 막는 이유와 CDP 우회 전략"
date: 2026-04-15 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, 쿠팡크롤링, 403오류, CloudflareWAF, ChromeCDP, 봇탐지]
description: "쿠팡 크롤링 시 403 Access Denied가 발생하는 정확한 원인을 분석하고, Chrome CDP 모바일 에뮬레이션으로 우회하는 전략을 단계별로 설명합니다. 실패한 시도들도 포함합니다."
---

## 문제 상황

```python
import requests
resp = requests.get("https://www.coupang.com/np/search?q=무선이어폰")
print(resp.status_code)  # 403
```

쿠팡에 크롤러를 붙이면 거의 항상 403이 반환됩니다. 심지어 User-Agent를 바꿔도, 헤더를 다 흉내내도 마찬가지입니다.

처음에는 단순히 "IP가 차단됐나" 싶었지만, 실제 원인은 훨씬 복잡합니다.

---

## 원인 분석: Cloudflare Bot Management

쿠팡은 Cloudflare Bot Management를 사용합니다. 단순한 IP 기반 차단이 아닌, **여러 신호를 종합해 봇 여부를 판단**합니다.

### Cloudflare가 감지하는 신호들

**1. TLS 지문 (JA3/JA4)**

`requests`, `urllib`, `httpx` 등 Python HTTP 라이브러리는 실제 Chrome과 다른 TLS 핸드셰이크 패턴을 만듭니다. Cloudflare는 이것만으로 Python 클라이언트를 탐지합니다.

```
requests TLS: JA3=769,47-53-5-10-49161-49162-...
Chrome TLS:   JA3=769,4865-4866-4867-49195-49196-...
```

→ 헤더만 바꿔도 TLS 지문이 다르면 바로 탐지됩니다.

**2. navigator.webdriver**

일반 Playwright/Selenium으로 실행하면 `navigator.webdriver === true`가 됩니다.

```javascript
// 봇: true
// 사람: undefined
console.log(navigator.webdriver);
```

**3. Chrome 버전 불일치**

User-Agent에 `Chrome/124.0.0.0`라고 써있지만, 실제 브라우저 기능이 Chrome 124와 다르면 감지됩니다.

**4. 행동 패턴**

검색 → 즉시 상품 클릭 → 즉시 다음 검색 같은 패턴은 사람이 하기 어렵습니다.

---

## 시도한 해결책들 (실패 포함)

### 시도 1: requests + 헤더 위장 ❌

```python
headers = {
    "User-Agent": "Mozilla/5.0 Chrome/124.0.0.0...",
    "Accept-Language": "ko-KR,ko;q=0.9",
    "Accept-Encoding": "gzip, deflate, br",
    # ... 수십 개의 헤더
}
resp = requests.get(url, headers=headers)
# 결과: 여전히 403
```

**실패 이유**: TLS 지문이 Python requests 그대로여서 즉시 탐지.

### 시도 2: httpx + HTTP/2 ❌

```python
import httpx
with httpx.Client(http2=True) as client:
    resp = client.get(url, headers=headers)
# 결과: 403
```

**실패 이유**: httpx의 TLS 지문도 Chrome과 다릅니다.

### 시도 3: curl-cffi (TLS 위장) △

```python
from curl_cffi import requests as cffi_requests
resp = cffi_requests.get(url, impersonate="chrome124")
# 결과: 간혹 성공, 하지만 불안정
```

TLS 지문을 Chrome처럼 위장하지만, navigator.webdriver 등 다른 신호들은 커버 못합니다.

### 시도 4: Playwright headless ❌

```python
with sync_playwright() as pw:
    browser = pw.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://www.coupang.com/np/search?q=무선이어폰")
# 결과: 403 또는 CAPTCHA 페이지
```

**실패 이유**: headless Chrome은 여러 지문이 다릅니다. screen 크기, WebGL 렌더러, 폰트 목록 등.

### 시도 5: Playwright headful △

headless를 headful로 바꾸면 성공률이 올라가지만, 자동화 감지 플래그(`navigator.webdriver`)가 남아있어서 불안정합니다.

### 시도 6: Chrome CDP + 모바일 UA ✅

**최종 해결책입니다.**

```python
CHROME_PATH = "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
MOBILE_UA = (
    "Mozilla/5.0 (Linux; Android 13; SM-G991B) "
    "AppleWebKit/537.36 Chrome/120.0.0.0 Mobile Safari/537.36"
)

cmd = [
    CHROME_PATH,
    f"--remote-debugging-port={CDP_PORT}",
    f"--user-data-dir={str(USER_DATA_DIR)}",
    f"--user-agent={MOBILE_UA}",
    "--window-size=390,844",
    "--disable-blink-features=AutomationControlled",
]
proc = subprocess.Popen(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
time.sleep(2)

# CDP로 연결
browser = playwright.chromium.connect_over_cdp(f"http://localhost:{CDP_PORT}")
```

**성공 이유:**
1. **실제 Chrome 실행**: TLS 지문이 진짜 Chrome과 동일
2. **모바일 UA + 뷰포트**: 모바일 쿠팡을 타겟 (PC보다 보안이 약함)
3. **AutomationControlled 비활성화**: `navigator.webdriver`가 undefined
4. **기존 Chrome 프로필 재사용**: 이미 로그인된 쿠키, 실제 사용자처럼 보임

---

## 최종 해결책 코드

```python
# sources/coupang.py (핵심 부분)
import subprocess, time, random
from playwright.sync_api import sync_playwright

def launch_and_connect():
    proc = subprocess.Popen([
        CHROME_PATH,
        f"--remote-debugging-port={CDP_PORT}",
        f"--user-data-dir={USER_DATA_DIR}",
        f"--user-agent={MOBILE_UA}",
        "--window-size=390,844",
        "--disable-blink-features=AutomationControlled",
        "--no-sandbox",
    ], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    
    time.sleep(2)  # Chrome 기동 대기
    
    pw = sync_playwright().start()
    browser = pw.chromium.connect_over_cdp(f"http://localhost:{CDP_PORT}")
    context = browser.contexts[0]
    page = context.new_page()
    page.set_viewport_size({"width": 390, "height": 844})
    
    return proc, pw, page

def cleanup(proc, pw):
    try:
        pw.stop()
    except Exception:
        pass
    try:
        proc.terminate()
        proc.wait(timeout=5)
    except Exception:
        pass
```

---

## 재발 방지 & 모니터링

**자동 감지 & 알림 설정:**

```python
def fetch_with_validation(page, url: str) -> bool:
    page.goto(url, timeout=30000, wait_until="domcontentloaded")
    
    # 403 또는 차단 페이지 감지
    if page.url != url:  # 리다이렉트 발생
        if "error" in page.url or "blocked" in page.url:
            notify_telegram(f"⚠️ 쿠팡 차단 감지: {page.url}")
            return False
    
    # 페이지 콘텐츠 확인
    if "Access Denied" in page.content() or "403" in page.title():
        notify_telegram("⚠️ 쿠팡 Access Denied 발생")
        return False
    
    return True
```

**Chrome 프로세스 모니터링:**

```python
import psutil

def check_chrome_health(port: int) -> bool:
    """CDP 포트가 열려있는지 확인"""
    for proc in psutil.process_iter(['name', 'cmdline']):
        cmdline = " ".join(proc.info['cmdline'] or [])
        if f"--remote-debugging-port={port}" in cmdline:
            return True
    return False
```

---

## 관련 포스팅

- **1부 [03] 쿠팡 크롤링 기본 구현**: [[03] Chrome CDP로 WAF 우회하고 파트너스 링크 자동 생성](/posts/auto-publishing-03/)
- **2부 [T08] 봇 탐지 우회 종합**: [[T08] 자동화 탐지 우회 종합 — headful·stealth·랜덤 딜레이](/posts/auto-publishing-t08/)

---

## 시리즈 전체 목차

### 2부 — 트러블슈팅 다이어리 (8편)

- **[현재 글] [T01] 쿠팡 크롤링 403 Access Denied 완전 해부**
- [[T02] 알리익스프레스 CAPTCHA 슬라이더 자동 감지](/posts/auto-publishing-t02/)
- [[T03] 세션 만료와 자동 재로그인](/posts/auto-publishing-t03/)
- [[T04] Rate Limit 429 대응 — Google·Naver 색인 API](/posts/auto-publishing-t04/)
- [[T05] Kakao SSO 25초 이상 멈춤 — 추가 인증 감지](/posts/auto-publishing-t05/)
- [[T06] 로그인 폼 셀렉터 깨짐 대응](/posts/auto-publishing-t06/)
- [[T07] AliExpress 5xx 오류 복구](/posts/auto-publishing-t07/)
- [[T08] 자동화 탐지 우회 종합](/posts/auto-publishing-t08/)

### 1부 — 핵심 아키텍처 / 3부 — SEO 심화

- [[01~14] 1부 전체 아키텍처 시리즈](/posts/auto-publishing-01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
