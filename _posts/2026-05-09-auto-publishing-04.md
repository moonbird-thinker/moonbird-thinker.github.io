---
title: "[Auto-Publishing] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회하고 제휴 링크 추출"
date: 2026-05-09 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, 알리익스프레스, Playwright, CAPTCHA우회]
description: "알리익스프레스 크롤링에서 가장 큰 난관인 슬라이더 CAPTCHA와 세션 관리를 Playwright와 playwright-stealth로 해결한 실전 구현을 공개합니다."
---

## 개요

알리익스프레스는 저렴한 가격과 폭넓은 상품군 덕분에 제휴 마케팅에서 매력적인 플랫폼입니다. 하지만 크롤링 관점에서는 쿠팡보다 훨씬 까다롭습니다.

두 가지 주요 장벽이 있습니다:
1. **슬라이더 CAPTCHA** — 자동화 감지 시 슬라이더를 밀어야만 계속 진행 가능
2. **세션 만료** — 로그인 상태가 자주 초기화됨

이번 편에서는 이 두 문제를 어떻게 처리했는지 설명합니다.

---

## 왜 Playwright를 선택했나

쿠팡은 Chrome CDP(로컬 Chrome 직접 실행)를 사용했지만, 알리익스프레스는 Playwright를 사용합니다.

그 이유는 **세션 관리의 차이**입니다:
- 쿠팡: 로컬 Chrome 프로필로 쿠키를 영속화
- 알리익스프레스: `storage_state.json`으로 로그인 상태 저장/복원이 필요

Playwright의 `storage_state` 기능이 여기서 빛을 발합니다.

---

## 핵심 구현

### playwright-stealth 설치 및 적용

일반 Playwright는 `navigator.webdriver` 속성이 `true`로 설정됩니다. 알리익스프레스는 이것을 감지해서 CAPTCHA를 보여줍니다.

`playwright-stealth`를 사용하면 이 속성을 제거할 수 있습니다.

```bash
pip install playwright-stealth
```

```python
# sources/aliexpress.py
from playwright.sync_api import sync_playwright, BrowserContext
from pathlib import Path

STORAGE_PATH = Path(".sessions/aliexpress_storage.json")
COOKIE_PATH = Path(".sessions/aliexpress_cookies.json")

class AliExpressSource:
    def __init__(self):
        self._page = None
        self._context = None
        self._session_reset_done = False
    
    def launch(self, headless: bool = False) -> BrowserContext:
        pw = sync_playwright().start()
        
        launch_args = [
            "--disable-blink-features=AutomationControlled",
            "--lang=ko-KR",
        ]
        
        browser = pw.chromium.launch(
            headless=headless,
            args=launch_args,
        )
        
        # 저장된 세션이 있으면 불러오기
        storage = str(STORAGE_PATH) if STORAGE_PATH.exists() else None
        
        context = browser.new_context(
            storage_state=storage,
            user_agent=(
                "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/124.0.0.0 Safari/537.36"
            ),
            locale="ko-KR",
            viewport={"width": 1280, "height": 900},
        )
        
        # playwright-stealth 적용
        try:
            from playwright_stealth import Stealth
            Stealth().apply_stealth_sync(context)
        except ImportError:
            log("playwright-stealth 미설치 — CAPTCHA 위험 증가", "warn")
        
        return context
```

### CAPTCHA 감지 로직

CAPTCHA가 나타나는 패턴은 일정합니다. 세 가지 방식으로 감지합니다.

```python
def _is_captcha_page(self) -> bool:
    """CAPTCHA / punish 페이지 감지"""
    url = self._page.url
    title = self._page.title().lower()
    
    # URL 패턴 감지
    if "captcha" in url or "punish" in url or "_____tmd_____" in url:
        return True
    
    # 제목 패턴 감지
    if "captcha" in title or "robot" in title:
        return True
    
    # DOM 요소 감지 (슬라이더 CAPTCHA)
    has_captcha = self._page.evaluate("""
        () => !!document.querySelector(
            '[id^="nc_"], .nc-container, #baxia-dialog, .baxia-dialog'
        )
    """)
    
    return bool(has_captcha)

def _wait_for_captcha_solve(self, wait_sec: int = 180) -> bool:
    """사용자가 수동으로 슬라이더를 밀 때까지 대기"""
    log(f"⚠️  CAPTCHA 감지 — {wait_sec}초 내에 슬라이더를 밀어주세요", "warn")
    
    # 텔레그램으로 즉시 알림
    notify_telegram("알리익스프레스 CAPTCHA 발생 — 수동 해결 필요")
    
    deadline = time.time() + wait_sec
    while time.time() < deadline:
        time.sleep(3)
        if not self._is_captcha_page():
            log("CAPTCHA 해결 확인", "ok")
            return True
    
    log("CAPTCHA 타임아웃", "error")
    return False
```

### 재시도 로직 (warmup + 세션 초기화)

알리익스프레스는 봇 감지 시 5xx 오류를 반환합니다. 이때 세션을 "워밍업"시키면 복구됩니다.

```python
def _goto_with_retry(self, url: str, retries: int = 2) -> bool:
    """페이지 이동 — 실패 시 warmup 후 재시도"""
    for attempt in range(retries + 1):
        try:
            self._page.goto(url, timeout=30000, wait_until="domcontentloaded")
            
            # CAPTCHA 체크
            if self._is_captcha_page():
                if not self._wait_for_captcha_solve():
                    return False
            
            return True
            
        except Exception as e:
            msg = str(e)
            log(f"goto 실패 [{attempt+1}/{retries+1}]: {msg[:120]}", "warn")
            
            if "ERR_HTTP_RESPONSE_CODE_FAILURE" not in msg or attempt >= retries:
                return False
            
            # 1차 복구: 자연스러운 네비게이션으로 세션 워밍업
            if attempt == 0:
                self._warmup_session()
                time.sleep(3)
            
            # 2차 복구: 세션 파일 삭제 후 재로그인
            elif not self._session_reset_done:
                if not self._reset_session():
                    return False
                time.sleep(3)
    
    return False

def _warmup_session(self):
    """홈 → 카테고리 순서로 자연스럽게 탐색해 세션 활성화"""
    try:
        self._page.goto("https://www.aliexpress.com", 
                       timeout=15000, wait_until="domcontentloaded")
        time.sleep(random.uniform(2, 3))
        self._page.goto("https://www.aliexpress.com/category/100003109",
                       timeout=15000, wait_until="domcontentloaded")
        time.sleep(random.uniform(2, 3))
    except Exception:
        pass

def _reset_session(self, wipe_storage: bool = True) -> bool:
    """storage_state 폐기 + 브라우저 재시작 + 재로그인"""
    self.close()
    
    if wipe_storage:
        for p in (STORAGE_PATH, COOKIE_PATH):
            if p.exists():
                p.unlink()
    
    if not self._relogin():
        return False
    
    self._session_reset_done = True
    return True
```

### 세션 만료 감지

알리익스프레스는 세션 만료 시 JSON이 아닌 HTML 로그인 페이지를 반환합니다.

```python
def _check_session_valid(self) -> bool:
    """세션 유효성 확인 — API 응답이 JSON이어야 정상"""
    try:
        resp = self._page.request.get(
            "https://www.aliexpress.com/af/icbu-auth.json",
            timeout=5000,
        )
        body = resp.text()
        
        # HTML이 반환되면 세션 만료
        if not body.strip().startswith("{"):
            if "login" in body.lower():
                log("세션 만료 감지 — storage 삭제 후 재로그인 필요", "warn")
                STORAGE_PATH.unlink(missing_ok=True)
                return False
        
        return True
    except Exception:
        return False
```

---

## 실패 사례 & 해결책

**실패 1: headless 모드에서 CAPTCHA가 항상 출현**

headless 모드는 알리익스프레스가 즉시 CAPTCHA를 발동합니다. 심지어 stealth 패치를 해도 차단됩니다.

→ **해결**: `headless=False`로 고정. 서버 배포 시 `Xvfb :99 &` 명령으로 가상 디스플레이를 먼저 실행하고 `DISPLAY=:99`를 설정합니다.

**실패 2: storage_state 로그인 정보가 3일마다 만료**

알리익스프레스 세션은 길지 않습니다. 3~7일 사이에 만료됩니다.

→ **해결**: 매일 새벽 세션 유효성 검사를 스케줄러에 추가. 만료 감지 시 텔레그램 알림 후 재로그인 파이프라인 자동 실행.

---

## 배운 점 / 주의사항

**랜덤 딜레이는 필수입니다.** 연속 요청 사이에 `time.sleep(random.uniform(2, 4))`를 넣지 않으면 빠르게 차단됩니다. 사람처럼 보이게 해야 합니다.

**stealth 라이브러리는 만능이 아닙니다.** playwright-stealth는 `navigator.webdriver` 등 기본 지문을 숨기지만, 행동 패턴(스크롤 속도, 마우스 이동)은 커버하지 못합니다.

자세한 트러블슈팅은 [T02편 - 알리익스프레스 CAPTCHA 슬라이더 자동 감지](/posts/auto-publishing-t02/)와 [T07편 - AliExpress 5xx 오류 복구](/posts/auto-publishing-t07/)를 참고하세요.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2. [[02] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기](/posts/auto-publishing-02/)
3. [[03] 쿠팡 크롤링 — Access Denied 뚫기: Chrome CDP로 WAF 우회](/posts/auto-publishing-03/)
4. **[현재 글] [Auto-Publishing] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회**
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
