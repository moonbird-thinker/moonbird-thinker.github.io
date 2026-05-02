---
title: "[Auto-Publishing] 자동화 탐지 우회 종합 — headful 모드·stealth 패치·랜덤 딜레이 실전 조합"
date: 2026-06-14 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, 봇탐지우회, PlaywrightStealth, headful, AutomationControlled, 랜덤딜레이]
description: "자동화 스크립트가 봇으로 탐지되는 원인을 분석하고, headful 모드·playwright-stealth·랜덤 딜레이·User-Agent 모사를 조합해 탐지를 우회하는 실전 전략을 설명합니다."
---

## 왜 자동화가 탐지되는가

크롤러가 차단되는 이유는 크게 세 가지입니다.

**1. 브라우저 지문(fingerprint) 노출**

Playwright로 띄운 Chromium은 기본적으로 `navigator.webdriver = true`를 노출합니다. Cloudflare, Baxia, Akamai 같은 봇 방어 솔루션이 이 속성을 우선 확인합니다.

**2. 행동 패턴의 비인간성**

- 요청 간격이 일정(0.0초 또는 정확히 N초)
- 마우스가 완벽한 직선으로 이동
- 스크롤이 순간적으로 완료
- 모든 요청이 같은 뷰포트에서 발생

**3. HTTP 헤더 이상**

- `User-Agent`가 headless Chromium을 노출
- `Accept-Language`, `Accept-Encoding` 부재
- `sec-ch-ua` 헤더가 자동화 브라우저와 일치

---

## 1단계: playwright-stealth 패치

`playwright-stealth`는 Playwright가 생성하는 자동화 흔적을 지웁니다.

```python
# common/browser_factory.py
from playwright.sync_api import sync_playwright
from playwright_stealth import stealth_sync

def create_stealthy_page(headless: bool = True):
    """stealth 패치가 적용된 페이지 생성"""
    pw = sync_playwright().start()
    
    browser = pw.chromium.launch(
        headless=headless,
        args=[
            "--disable-blink-features=AutomationControlled",
            "--disable-dev-shm-usage",
            "--no-sandbox",
            "--disable-gpu",
            # User-Agent 재정의 (아래에서 설명)
        ],
    )
    
    context = browser.new_context(
        user_agent=(
            "Mozilla/5.0 (Linux; Android 13; SM-G991B) "
            "AppleWebKit/537.36 (KHTML, like Gecko) "
            "Chrome/120.0.0.0 Mobile Safari/537.36"
        ),
        viewport={"width": 390, "height": 844},
        locale="ko-KR",
        timezone_id="Asia/Seoul",
    )
    
    page = context.new_page()
    
    # stealth 패치 적용 (navigator.webdriver 등 제거)
    stealth_sync(page)
    
    return pw, browser, context, page
```

### stealth가 패치하는 주요 항목

```python
# playwright-stealth 내부 동작 (참고용)
patches = [
    "navigator.webdriver",          # 가장 중요 — False로 설정
    "navigator.plugins",            # 플러그인 목록 주입
    "navigator.languages",          # ['ko-KR', 'ko', 'en-US', 'en']
    "WebGLRenderingContext",        # GPU 렌더러 위장
    "chrome.runtime",               # Chrome extension API 노출
    "permissions.query",            # 권한 쿼리 응답 정상화
    "Notification.permission",      # 알림 권한 기본값
    "PluginArray.prototype.item",   # 플러그인 배열 노출
]
```

---

## 2단계: headful 모드 전략적 사용

headless 브라우저는 특정 Canvas API 결과, WebGL 렌더러 문자열, 화면 해상도가 실제 기기와 다릅니다. 고급 봇 탐지 시스템은 이를 감지합니다.

```python
# common/browser_factory.py

def should_use_headful(platform: str) -> bool:
    """플랫폼별 headful 필요 여부"""
    ALWAYS_HEADFUL = {
        "aliexpress",  # Baxia 고급 지문 분석
        "kakao",       # 추가 인증 팝업 처리
        "coupang",     # Cloudflare Bot Manager
    }
    return platform in ALWAYS_HEADFUL

def create_browser(platform: str):
    headless = not should_use_headful(platform)
    
    if not headless:
        log(f"[{platform}] headful 모드로 실행 (봇 탐지 우회)", "info")
    
    return create_stealthy_page(headless=headless)
```

### headful이 필요한 상황

```python
# 세션 생성 시에만 headful, 이후 세션 재사용은 headless
def initialize_session(platform: str) -> dict:
    """최초 로그인은 headful — 이후는 storage_state 재사용"""
    
    pw, browser, ctx, page = create_browser(platform)
    
    # headful로 로그인 (사람처럼 보임)
    login_success = perform_login(page, platform)
    
    if login_success:
        # 세션 저장
        state = ctx.storage_state()
        save_session(platform, state)
        log(f"[{platform}] 세션 저장 완료 — 이후 headless 재사용 가능", "ok")
    
    browser.close()
    pw.stop()
    
    return state
```

---

## 3단계: 랜덤 딜레이 — 인간 행동 모사

일정한 간격은 즉시 봇으로 탐지됩니다. 인간의 실제 클릭/스크롤 패턴을 모사합니다.

```python
# common/human_behavior.py
import random
import time
import math

def human_delay(min_sec: float = 0.5, max_sec: float = 2.0):
    """인간적 딜레이 — 정규분포 기반"""
    # 완전한 균등분포 대신 자연스러운 분포
    mean = (min_sec + max_sec) / 2
    std = (max_sec - min_sec) / 6
    
    delay = max(min_sec, min(max_sec, random.gauss(mean, std)))
    time.sleep(delay)

def human_scroll(page, target_y: int = None):
    """자연스러운 스크롤 — 단계적으로 내려감"""
    if target_y is None:
        target_y = random.randint(300, 800)
    
    current_y = 0
    
    while current_y < target_y:
        # 스크롤 양을 랜덤하게
        scroll_amount = random.randint(80, 200)
        current_y = min(current_y + scroll_amount, target_y)
        
        page.evaluate(f"window.scrollTo(0, {current_y})")
        
        # 스크롤 사이 짧은 멈춤
        time.sleep(random.uniform(0.05, 0.15))
    
    # 스크롤 완료 후 잠깐 대기 (읽는 척)
    time.sleep(random.uniform(0.5, 1.5))

def human_type(element, text: str):
    """키보드 입력 — 자연스러운 타이핑 속도"""
    for char in text:
        element.type(char, delay=random.randint(50, 150))
        
        # 가끔 더 긴 멈춤 (생각하는 척)
        if random.random() < 0.05:
            time.sleep(random.uniform(0.3, 0.8))

def human_mouse_move(page, target_x: int, target_y: int):
    """마우스를 곡선으로 이동 (직선은 봇처럼 보임)"""
    
    # 현재 위치 가져오기 (없으면 중앙으로 가정)
    start_x, start_y = 195, 422
    
    steps = random.randint(15, 30)
    
    for i in range(steps + 1):
        t = i / steps
        
        # 베지어 곡선 중간점 (약간의 굴곡)
        control_x = start_x + (target_x - start_x) * 0.5 + random.randint(-30, 30)
        control_y = start_y + (target_y - start_y) * 0.5 + random.randint(-30, 30)
        
        # 2차 베지어 곡선
        x = (1-t)**2 * start_x + 2*(1-t)*t * control_x + t**2 * target_x
        y = (1-t)**2 * start_y + 2*(1-t)*t * control_y + t**2 * target_y
        
        page.mouse.move(x, y)
        time.sleep(random.uniform(0.01, 0.03))
```

---

## 4단계: User-Agent와 HTTP 헤더 위장

```python
# common/ua_pool.py

# 실제 사용 중인 모바일 UA 풀
MOBILE_USER_AGENTS = [
    # Galaxy S23 시리즈
    "Mozilla/5.0 (Linux; Android 13; SM-G991B) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36",
    "Mozilla/5.0 (Linux; Android 14; SM-S918B) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Mobile Safari/537.36",
    # Galaxy A 시리즈
    "Mozilla/5.0 (Linux; Android 13; SM-A546B) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Mobile Safari/537.36",
    # Pixel 시리즈
    "Mozilla/5.0 (Linux; Android 14; Pixel 8) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36",
]

DESKTOP_USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36",
]

def get_random_ua(mobile: bool = True) -> str:
    pool = MOBILE_USER_AGENTS if mobile else DESKTOP_USER_AGENTS
    return random.choice(pool)

def build_context_options(mobile: bool = True) -> dict:
    """플랫폼별 컨텍스트 옵션 생성"""
    ua = get_random_ua(mobile)
    
    if mobile:
        return {
            "user_agent": ua,
            "viewport": {"width": random.choice([390, 412, 375, 360]), "height": 844},
            "device_scale_factor": 3,
            "is_mobile": True,
            "has_touch": True,
            "locale": "ko-KR",
            "timezone_id": "Asia/Seoul",
            "extra_http_headers": {
                "Accept-Language": "ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7",
                "Accept-Encoding": "gzip, deflate, br",
            },
        }
    else:
        return {
            "user_agent": ua,
            "viewport": {"width": random.choice([1366, 1440, 1920]), "height": random.choice([768, 900, 1080])},
            "locale": "ko-KR",
            "timezone_id": "Asia/Seoul",
        }
```

---

## 5단계: 요청 패턴 분산

```python
# common/rate_limiter.py
import time
import random
from collections import deque

class AdaptiveRateLimiter:
    """봇 탐지를 피하는 적응형 속도 제한기"""
    
    def __init__(self, base_interval: float = 3.0, jitter: float = 2.0):
        self.base_interval = base_interval
        self.jitter = jitter
        self._last_request = 0.0
        self._consecutive_fast = 0
        self._error_count = 0
    
    def wait(self):
        """다음 요청 전 대기"""
        
        # 연속 오류 발생 시 간격 자동 증가
        if self._error_count > 0:
            extra = self._error_count * 5.0
            interval = self.base_interval + extra + random.uniform(0, self.jitter)
        else:
            interval = self.base_interval + random.uniform(-self.jitter/2, self.jitter)
        
        interval = max(1.0, interval)
        
        elapsed = time.time() - self._last_request
        if elapsed < interval:
            time.sleep(interval - elapsed)
        
        self._last_request = time.time()
    
    def on_success(self):
        self._error_count = max(0, self._error_count - 1)
    
    def on_error(self):
        self._error_count += 1
        log(f"연속 오류 {self._error_count}회 — 요청 간격 증가", "warn")

# 플랫폼별 기본 설정
RATE_LIMITERS = {
    "aliexpress": AdaptiveRateLimiter(base_interval=5.0, jitter=3.0),
    "coupang": AdaptiveRateLimiter(base_interval=8.0, jitter=4.0),
    "naver": AdaptiveRateLimiter(base_interval=2.0, jitter=1.5),
}
```

---

## 종합 적용 예시

```python
# sources/aliexpress.py (실제 적용 패턴)

class AliExpressSource:
    
    def __init__(self):
        # stealth + headful + 랜덤 UA 조합
        self._pw, self._browser, self._ctx, self._page = create_browser("aliexpress")
        self._limiter = RATE_LIMITERS["aliexpress"]
    
    def fetch_product(self, product_id: str) -> dict:
        url = f"https://www.aliexpress.com/item/{product_id}.html"
        
        # 1. 요청 전 대기
        self._limiter.wait()
        
        # 2. 자연스러운 스크롤 (탐색 패턴 모사)
        human_scroll(self._page, random.randint(200, 500))
        human_delay(0.5, 1.5)
        
        # 3. 페이지 이동
        try:
            self._page.goto(url, timeout=30000, wait_until="domcontentloaded")
            
            # 4. 도착 후 스크롤 (실제 독자처럼)
            human_scroll(self._page, random.randint(300, 700))
            human_delay(1.0, 3.0)
            
            self._limiter.on_success()
            return self._extract_product()
            
        except Exception as e:
            self._limiter.on_error()
            raise
```

---

## 탐지 우회 효과 비교

| 전략 | headless only | + stealth | + headful | + 랜덤딜레이 | 종합 |
|------|:---:|:---:|:---:|:---:|:---:|
| navigator.webdriver | ❌ 노출 | ✅ 패치 | ✅ 패치 | ✅ 패치 | ✅ |
| Canvas 지문 | ❌ 이상 | ⚠️ 부분 | ✅ 실제 | ✅ 실제 | ✅ |
| 요청 패턴 | ❌ 일정 | ❌ 일정 | ❌ 일정 | ✅ 랜덤 | ✅ |
| TLS 지문 | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| IP 평판 | ❌ | ❌ | ❌ | ❌ | ❌ |

TLS 지문과 IP 평판은 이 방법으로 해결이 안 됩니다. 필요 시 Residential Proxy를 추가로 고려하세요.

---

## 배운 점 / 주의사항

**stealth 패치는 매 컨텍스트 생성마다 적용해야 합니다.** 페이지가 새로 열릴 때마다 다시 패치가 필요하지는 않지만, 컨텍스트가 새로 만들어지면 재패치가 필요합니다.

**headful 모드는 서버 환경에서 Xvfb가 필요합니다.** Linux 서버에서 headful을 사용하려면 가상 디스플레이가 있어야 합니다.

```bash
# Linux 서버에서 headful 실행
Xvfb :99 -screen 0 1920x1080x24 &
export DISPLAY=:99
python main.py
```

---

## 관련 포스팅

- [[T01] 쿠팡 크롤링 403 Access Denied 완전 해부](/posts/auto-publishing-t01/)
- [[T02] 알리익스프레스 CAPTCHA 슬라이더 자동 감지](/posts/auto-publishing-t02/)
- [[03] 쿠팡 크롤링 — Access Denied 뚫기](/posts/auto-publishing-03/)
- [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
