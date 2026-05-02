---
title: "[Auto-Publishing] 쿠팡 크롤링 — Access Denied 뚫기: Chrome CDP로 WAF 우회하고 파트너스 링크 자동 생성"
date: 2026-05-07 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, 쿠팡크롤링, ChromeCDP, WAF우회]
description: "쿠팡 크롤링 시 가장 큰 장벽인 Cloudflare WAF의 403 Access Denied를 Chrome DevTools Protocol로 우회하는 실전 구현을 공개합니다. 모바일 에뮬레이션과 로컬 Chrome 프로필 활용이 핵심입니다."
---

## 개요

쿠팡은 크롤러 차단이 매우 강합니다. 일반 `requests`나 `urllib`으로 접근하면 대부분 **403 Access Denied**가 반환됩니다. Cloudflare WAF가 버티고 있기 때문입니다.

이번 편에서는 이 문제를 **Chrome DevTools Protocol(CDP)** 로 해결한 방법을 설명합니다. 핵심은 실제 Chrome을 로컬에서 실행하고, 평소 쿠팡 앱처럼 보이는 모바일 User-Agent를 사용하는 것입니다.

트러블슈팅 상세는 [T01편](/posts/auto-publishing-t01/)에서 별도로 다룹니다.

---

## 왜 이 기술이 필요했나

쿠팡 파트너스 수익의 전제 조건은 **쿠팡 상품 링크가 글에 포함**되는 것입니다. 그러려면 상품 정보(이름, 가격, 이미지, URL)를 수집해야 합니다.

쿠팡은 공개 API를 제공하지 않습니다. 파트너스 API는 존재하지만 HMAC 서명 방식의 인증이 필요하고, 제공하는 데이터가 제한적입니다.

결국 직접 상품 페이지를 크롤링하는 것이 가장 현실적인 방법입니다.

---

## 핵심 구현

### CDP 기반 Chrome 실행

CDP는 Chrome을 원격에서 제어할 수 있게 해주는 프로토콜입니다. Playwright나 Selenium과 다르게, 완전히 **일반 Chrome과 동일한 지문**을 가집니다.

```python
# sources/coupang.py
import subprocess
import time
import requests
from pathlib import Path

CHROME_PATH = "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
CDP_PORT = 9222
USER_DATA_DIR = Path(".sessions/coupang_chrome")

# 모바일 UA: 실제 안드로이드 갤럭시 S21
MOBILE_UA = (
    "Mozilla/5.0 (Linux; Android 13; SM-G991B) "
    "AppleWebKit/537.36 (KHTML, like Gecko) "
    "Chrome/120.0.0.0 Mobile Safari/537.36"
)

def launch_chrome() -> subprocess.Popen:
    """로컬 Chrome을 CDP 모드로 실행"""
    USER_DATA_DIR.mkdir(parents=True, exist_ok=True)
    
    cmd = [
        CHROME_PATH,
        f"--remote-debugging-port={CDP_PORT}",
        f"--user-data-dir={str(USER_DATA_DIR)}",
        f"--user-agent={MOBILE_UA}",
        "--window-size=390,844",        # iPhone 14 Pro 크기
        "--disable-blink-features=AutomationControlled",  # 봇 탐지 비활성화
        "--no-sandbox",
        "--disable-extensions",
        "--disable-default-apps",
    ]
    proc = subprocess.Popen(
        cmd,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )
    time.sleep(2)  # Chrome 기동 대기
    return proc
```

**왜 `--disable-blink-features=AutomationControlled`인가?**

이 플래그가 없으면 `navigator.webdriver === true`가 됩니다. Cloudflare는 이 속성을 확인해서 자동화 탐지를 합니다.

### Playwright로 CDP에 연결

```python
from playwright.sync_api import sync_playwright

def connect_to_chrome():
    """실행 중인 Chrome에 CDP로 연결"""
    with sync_playwright() as pw:
        # CDP WebSocket 엔드포인트에 연결
        browser = pw.chromium.connect_over_cdp(
            f"http://localhost:{CDP_PORT}"
        )
        context = browser.contexts[0]
        page = context.new_page()
        
        # 모바일 뷰포트 설정
        page.set_viewport_size({"width": 390, "height": 844})
        
        return browser, context, page
```

### 상품 검색 및 정보 추출

```python
class CoupangSource:
    def __init__(self):
        self._proc = None
        self._browser = None
    
    def fetch(self, keyword: str, limit: int = 10) -> list:
        self._proc = launch_chrome()
        self._browser, context, page = connect_to_chrome()
        
        try:
            return self._search_products(page, keyword, limit)
        finally:
            self._cleanup()
    
    def _search_products(self, page, keyword: str, limit: int) -> list:
        # 쿠팡 모바일 검색 URL
        search_url = f"https://m.coupang.com/nm/search?q={keyword}"
        page.goto(search_url, wait_until="domcontentloaded", timeout=30000)
        
        # 상품 카드 로딩 대기
        page.wait_for_selector(".search-product-wrap", timeout=10000)
        
        products = []
        items = page.query_selector_all(".search-product-wrap")[:limit]
        
        for item in items:
            try:
                name = item.query_selector(".name").inner_text()
                price = item.query_selector(".price-value").inner_text()
                link = item.query_selector("a").get_attribute("href")
                img = item.query_selector("img").get_attribute("src")
                
                products.append({
                    "name": name.strip(),
                    "price": price.strip(),
                    "url": f"https://m.coupang.com{link}",
                    "image": img,
                    "affiliate_url": self._make_affiliate_url(link),
                })
                
                # 자연스러운 탐색 패턴 (봇 탐지 회피)
                time.sleep(random.uniform(1.5, 2.5))
                
            except Exception as e:
                log(f"상품 파싱 실패: {e}", "warn")
                continue
        
        return products
```

### 파트너스 링크 생성 (HMAC-SHA256)

쿠팡 파트너스 API는 HMAC-SHA256 서명이 필요합니다.

```python
import hmac
import hashlib
import time

PARTNERS_ACCESS_KEY = "YOUR_ACCESS_KEY"
PARTNERS_SECRET_KEY = "YOUR_SECRET_KEY"

def make_affiliate_url(product_url: str) -> str:
    """쿠팡 파트너스 제휴 링크 생성"""
    timestamp = str(int(time.time() * 1000))
    path = "/v2/providers/affiliate_open_api/apis/openapi/v1/deeplink"
    
    # HMAC-SHA256 서명 생성
    message = f"{timestamp}{path}"
    signature = hmac.new(
        PARTNERS_SECRET_KEY.encode("utf-8"),
        message.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()
    
    resp = requests.post(
        f"https://api-gateway.coupang.com{path}",
        json={"coupangUrls": [product_url]},
        headers={
            "Authorization": f"CEA algorithm=HmacSHA256, access-key={PARTNERS_ACCESS_KEY}, "
                           f"signed-date={timestamp}, signature={signature}",
            "Content-Type": "application/json",
        },
        timeout=10,
    )
    
    data = resp.json()
    return data["data"][0]["shortenUrl"]
```

---

## 실패 사례 & 해결책

**실패 1: headless Chrome은 즉시 차단됨**

`--headless` 플래그를 사용하면 Cloudflare에서 즉시 차단됩니다. headless Chrome은 지문이 완전히 다릅니다.

→ **해결**: headless를 완전히 제거. 백그라운드에서 실행되더라도 headful 모드 사용. 서버에서는 Xvfb(가상 디스플레이)로 실행.

**실패 2: 로컬 Chrome 프로필 없으면 쿠키가 없음**

Chrome을 처음 실행하면 쿠팡 로그인이 안 된 상태입니다. 비로그인 상태에서 특정 상품 페이지는 접근이 막힙니다.

→ **해결**: `USER_DATA_DIR`을 고정 경로로 지정. 한 번만 수동으로 로그인하면 이후에는 쿠키가 유지됩니다.

**실패 3: ChromeDriver 버전 불일치**

Playwright가 내부적으로 Chromium을 관리하기 때문에 로컬 Chrome과 버전이 다를 수 있습니다.

→ **해결**: CDP 방식은 Playwright의 Chromium이 아닌 **로컬 Chrome을 직접 실행**합니다. 버전 불일치 문제가 없습니다.

---

## 배운 점 / 주의사항

**쿠팡 상품 페이지 구조는 수시로 바뀝니다.** CSS 셀렉터가 깨질 수 있습니다. 중요한 셀렉터는 여러 개의 폴백을 만들어 두는 것을 권장합니다.

**파트너스 API 호출 제한**을 확인하세요. 일일 호출 한도를 초과하면 링크 생성이 안 됩니다.

**Chrome 프로세스 정리**를 반드시 해야 합니다. `proc.terminate()`를 빠뜨리면 Chrome 프로세스가 계속 쌓입니다. 트러블슈팅은 [T01편](/posts/auto-publishing-t01/)을 참고하세요.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2. [[02] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기](/posts/auto-publishing-02/)
3. **[현재 글] [Auto-Publishing] 쿠팡 크롤링 — Access Denied 뚫기**
4. [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
5. [[05] Claude CLI와 Gemini API로 상품 소개글 자동 생성하기](/posts/auto-publishing-05/)
6. [[06] WordPress REST API로 멀티 사이트 자동 발행 구현](/posts/auto-publishing-06/)
7. [[07] 티스토리 자동 발행 — Playwright Persistent Context로 Kakao SSO 유지](/posts/auto-publishing-07/)
8. [[08] 네이버 RSA 암호화 로그인으로 블로그·카페 자동 발행](/posts/auto-publishing-08/)
9. [[09] GitHub Pages 자동 발행 — Jekyll Markdown 자동 커밋·푸시](/posts/auto-publishing-09/)
10. [[10] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest](/posts/auto-publishing-10/)
11. [[11] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행](/posts/auto-publishing-11/)
12. [[12] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기](/posts/auto-publishing-12/)
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 (8편)

- [[T01] 쿠팡 크롤링 403 Access Denied 완전 해부](/posts/auto-publishing-t01/)
- [[T02] 알리익스프레스 CAPTCHA 슬라이더 자동 감지](/posts/auto-publishing-t02/)
- [[T03] 세션 만료와 자동 재로그인](/posts/auto-publishing-t03/)
- [[T04] Rate Limit 429 대응 — Google·Naver 색인 API](/posts/auto-publishing-t04/)
- [[T05] Kakao SSO 25초 이상 멈춤 — 추가 인증 감지](/posts/auto-publishing-t05/)
- [[T06] 로그인 폼 셀렉터 깨짐 대응](/posts/auto-publishing-t06/)
- [[T07] AliExpress 5xx 오류 복구](/posts/auto-publishing-t07/)
- [[T08] 자동화 탐지 우회 종합](/posts/auto-publishing-t08/)

### 3부 — SEO 심화: 백링크와 색인 (4편)

- [[S01] 백링크란 무엇인가 — 내부 링크 구조가 SEO에 미치는 영향](/posts/auto-publishing-s01/)
- [[S02] 색인이란 무엇인가 — 구글·네이버가 내 글을 발견하는 원리](/posts/auto-publishing-s02/)
- [[S03] Google Search Console Indexing API 자동화](/posts/auto-publishing-s03/)
- [[S04] 네이버 서치어드바이저 색인 API 자동화](/posts/auto-publishing-s04/)
