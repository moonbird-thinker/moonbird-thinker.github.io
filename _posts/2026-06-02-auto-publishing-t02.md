---
title: "[Auto-Publishing] 알리익스프레스 CAPTCHA 슬라이더 자동 감지 — nc_container·punish URL 패턴 분석"
date: 2026-06-02 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, 알리익스프레스, CAPTCHA, playwright-stealth, 슬라이더CAPTCHA]
description: "알리익스프레스 크롤링에서 자주 마주치는 슬라이더 CAPTCHA의 URL 패턴과 DOM 요소를 정확히 감지하고, headful 모드·playwright-stealth로 발생 빈도를 줄이는 전략을 설명합니다."
---

## 문제 상황

알리익스프레스 크롤러를 돌리다 보면 갑자기 페이지가 바뀝니다.

```
정상 URL: https://www.aliexpress.com/item/1005006...
CAPTCHA URL: https://www.aliexpress.com/anti-fraud?_____tmd_____=...
또는: https://baxia.aliexpress.com/punish?spm=...
```

화면에는 슬라이더가 나타나고 크롤러는 멈춥니다. 이것이 알리익스프레스의 **Baxia Bot Defense** 시스템입니다.

---

## 원인 분석

알리익스프레스는 Alibaba의 자체 봇 방어 시스템인 Baxia를 사용합니다. 감지 기준:

**행동 분석:**
- 마우스 이동 없이 즉시 클릭
- 스크롤 없이 페이지 하단 접근
- 일정한 속도의 연속 요청

**기술적 신호:**
- `navigator.webdriver === true`
- Canvas 지문 불일치
- WebGL 렌더러 이상 (headless GPU)
- Chrome 플러그인 목록 비어있음

**세션 신호:**
- 신규 세션 (기존 쿠키 없음)
- 비정상적인 Referrer 패턴

---

## CAPTCHA 감지 로직 상세

세 가지 방법으로 CAPTCHA를 감지합니다.

```python
# sources/aliexpress.py

def _is_captcha_page(self) -> bool:
    """CAPTCHA / 봇 차단 페이지 감지"""
    
    # 1. URL 패턴 감지 (가장 빠름)
    url = self._page.url
    captcha_url_patterns = [
        "_____tmd_____",   # 알리 봇 감지 파라미터
        "punish",          # Baxia punish 페이지
        "anti-fraud",      # 사기 방지 페이지
        "captcha",         # 일반 CAPTCHA
        "baxia",           # Baxia 도메인
    ]
    if any(p in url for p in captcha_url_patterns):
        log(f"URL 패턴으로 CAPTCHA 감지: {url[:80]}", "warn")
        return True
    
    # 2. 페이지 타이틀 감지
    title = self._page.title().lower()
    if any(p in title for p in ["captcha", "robot", "verify", "human"]):
        log(f"타이틀로 CAPTCHA 감지: {title}", "warn")
        return True
    
    # 3. DOM 요소 감지 (슬라이더 CAPTCHA 요소 확인)
    has_captcha_element = self._page.evaluate("""
        () => {
            // Baxia 슬라이더 CAPTCHA 요소들
            const selectors = [
                '[id^="nc_"]',          // nc_1_n1z 같은 슬라이더 ID
                '.nc-container',        // 슬라이더 컨테이너
                '#baxia-dialog',        // Baxia 다이얼로그
                '.baxia-dialog',        
                '#J_MIDDLEWARE_ERROR',   // 미들웨어 에러 페이지
                '.J-middleWare-error',
            ];
            
            return selectors.some(sel => {
                const el = document.querySelector(sel);
                return el && el.offsetParent !== null; // 실제로 보이는 요소만
            });
        }
    """)
    
    if has_captcha_element:
        log("DOM 요소로 CAPTCHA 감지 (슬라이더)", "warn")
        return True
    
    return False
```

---

## 시도한 해결책들

### 시도 1: 슬라이더 자동 풀기 ❌

Playwright로 슬라이더를 프로그래밍적으로 드래그하면 감지됩니다.

```python
# 이것은 작동하지 않습니다
slider = page.locator(".nc-container .btn_slide")
slider.drag_to(page.locator(".nc-container .track"))
# 결과: "인증 실패" — 움직임 패턴이 비자연스러움
```

**실패 이유**: 알리의 슬라이더는 마우스 가속도, 미세한 흔들림, 드래그 시간 등을 분석합니다.

### 시도 2: 2captcha 서비스 △

2captcha 같은 외부 CAPTCHA 해결 서비스가 있습니다. 하지만:
- 비용이 발생 ($0.001~0.003/개)
- 처리 시간 10~30초
- 알리의 행동 기반 감지는 우회 못함

### 시도 3: 발생 자체를 줄이기 ✅

CAPTCHA를 푸는 것보다 **CAPTCHA가 뜨지 않게 하는 것**이 훨씬 효과적입니다.

---

## 최종 전략: CAPTCHA 발생 억제

### playwright-stealth 적용

```python
from playwright_stealth import Stealth

context = browser.new_context(...)
Stealth().apply_stealth_sync(context)
```

playwright-stealth가 패치하는 항목들:
- `navigator.webdriver` → undefined
- `navigator.plugins` → 실제 플러그인 목록으로 채움
- `navigator.languages` → 한국어로 설정
- `window.chrome` → Chrome 객체 주입
- Canvas 지문 → 약간의 노이즈 추가
- WebGL 렌더러 → 정상 GPU로 위장

### 자연스러운 탐색 패턴

```python
def _warmup_session(self):
    """크롤링 전 자연스러운 브라우징으로 세션 워밍업"""
    
    # 1. 알리익스프레스 메인 방문
    self._page.goto("https://www.aliexpress.com", 
                    timeout=15000, wait_until="domcontentloaded")
    time.sleep(random.uniform(2, 4))
    
    # 2. 스크롤 (사람처럼)
    self._page.evaluate("window.scrollBy(0, 300)")
    time.sleep(random.uniform(0.5, 1.5))
    self._page.evaluate("window.scrollBy(0, 200)")
    time.sleep(random.uniform(1, 2))
    
    # 3. 카테고리 페이지 방문
    self._page.goto(
        "https://www.aliexpress.com/category/100003109",
        timeout=15000, wait_until="domcontentloaded"
    )
    time.sleep(random.uniform(2, 3))
```

### 랜덤 딜레이 적용

```python
def _navigate_naturally(self, url: str):
    """자연스러운 딜레이로 페이지 이동"""
    # 이동 전 잠깐 대기
    time.sleep(random.uniform(1, 3))
    
    self._page.goto(url, timeout=30000, wait_until="domcontentloaded")
    
    # 페이지 로드 후 스크롤
    time.sleep(random.uniform(0.5, 1.5))
    self._page.evaluate(f"window.scrollBy(0, {random.randint(100, 400)})")
```

### 수동 CAPTCHA 대기 (최후 수단)

```python
def _wait_for_captcha_solve(self, wait_sec: int = 180) -> bool:
    """CAPTCHA 감지 시 사용자에게 수동 해결 요청"""
    log(f"⚠️  CAPTCHA 감지 — {wait_sec}초 내에 슬라이더를 밀어주세요", "warn")
    
    # 텔레그램 + 카카오톡 동시 알림
    notify_telegram(
        f"🔴 알리익스프레스 CAPTCHA 발생!\n"
        f"현재 URL: {self._page.url}\n"
        f"수동으로 슬라이더를 밀어주세요.\n"
        f"제한 시간: {wait_sec}초"
    )
    
    # CAPTCHA 해결 감지 루프
    deadline = time.time() + wait_sec
    while time.time() < deadline:
        time.sleep(3)
        if not self._is_captcha_page():
            log("CAPTCHA 해결 확인 — 크롤링 재개", "ok")
            return True
    
    log("CAPTCHA 타임아웃 — 세션 초기화 필요", "error")
    return False
```

---

## 재발 방지 & 모니터링

**CAPTCHA 발생 빈도 추적:**

```python
captcha_count = 0

def fetch_with_captcha_tracking(page, url: str) -> bool:
    global captcha_count
    
    page.goto(url, timeout=30000, wait_until="domcontentloaded")
    
    if _is_captcha_page(page):
        captcha_count += 1
        if captcha_count % 5 == 0:
            notify_telegram(f"⚠️ CAPTCHA {captcha_count}회 발생 — 전략 점검 필요")
        return False
    
    return True
```

**일일 CAPTCHA 비율 리포트:**

CAPTCHA가 총 요청의 10% 이상이면 탐지 회피 전략을 재검토해야 합니다.

---

## 관련 포스팅

- **1부 [04] 알리익스프레스 기본 구현**: [[04] Playwright로 CAPTCHA 우회하고 제휴 링크 추출](/posts/auto-publishing-04/)
- **2부 [T07] AliExpress 5xx 오류 복구**: [[T07] warmup 세션 재구성과 storage_state 초기화](/posts/auto-publishing-t07/)
- **2부 [T08] 탐지 우회 종합**: [[T08] 자동화 탐지 우회 종합](/posts/auto-publishing-t08/)

---

## 시리즈 전체 목차

### 2부 — 트러블슈팅 다이어리 (8편)

- [[T01] 쿠팡 크롤링 403 Access Denied 완전 해부](/posts/auto-publishing-t01/)
- **[현재 글] [T02] 알리익스프레스 CAPTCHA 슬라이더 자동 감지**
- [[T03] 세션 만료와 자동 재로그인](/posts/auto-publishing-t03/)
- [[T04] Rate Limit 429 대응 — Google·Naver 색인 API](/posts/auto-publishing-t04/)
- [[T05] Kakao SSO 25초 이상 멈춤 — 추가 인증 감지](/posts/auto-publishing-t05/)
- [[T06] 로그인 폼 셀렉터 깨짐 대응](/posts/auto-publishing-t06/)
- [[T07] AliExpress 5xx 오류 복구](/posts/auto-publishing-t07/)
- [[T08] 자동화 탐지 우회 종합](/posts/auto-publishing-t08/)
