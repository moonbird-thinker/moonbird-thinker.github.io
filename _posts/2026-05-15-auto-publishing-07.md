---
title: "[Auto-Publishing] 티스토리 자동 발행 — Playwright Persistent Context로 Kakao SSO 유지"
date: 2026-05-15 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, 티스토리자동발행, 카카오SSO, Playwright]
description: "티스토리 자동 발행에서 가장 어려운 부분인 카카오 SSO 세션 유지를 Playwright Persistent Context로 해결한 방법을 설명합니다. CAPTCHA 추가 인증 감지와 텔레그램 알림까지 구현합니다."
---

## 개요

티스토리는 카카오 로그인(SSO)을 사용합니다. 이것이 자동화의 가장 큰 장벽입니다.

일반적인 ID/PW 로그인이 아니라 카카오 OAuth 흐름을 따르기 때문에, 자동화 스크립트가 로그인을 유지하기가 까다롭습니다.

이번 편에서는 **Playwright Persistent Context**로 카카오 세션을 수주간 유지하는 방법을 설명합니다.

---

## 왜 Persistent Context인가

Playwright에는 두 가지 컨텍스트 모드가 있습니다:

| 방식 | 설명 | 세션 지속 |
|------|------|---------|
| 일반 `new_context()` | 매번 새 프로필 생성 | 실행마다 초기화 |
| `launch_persistent_context()` | 디스크에 프로필 저장 | 영구 유지 |

티스토리 + 카카오 SSO에는 **반드시 Persistent Context**를 써야 합니다. 한 번 로그인하면 카카오 세션 쿠키가 저장되고, 이후 실행에서는 자동으로 로그인 상태가 복원됩니다.

---

## 핵심 구현

### Persistent Context 설정

```python
# publishers/tistory.py
from playwright.sync_api import sync_playwright
from pathlib import Path
import time

PROFILE_DIR = Path(".sessions/tistory_profile")
BLOG_NAME = "myblog"  # myblog.tistory.com

class TistoryPublisher:
    def __init__(self):
        self._pw = None
        self._browser = None
        self._context = None
        self._page = None
    
    def __enter__(self):
        self._pw = sync_playwright().start()
        
        PROFILE_DIR.mkdir(parents=True, exist_ok=True)
        
        # Persistent Context 실행 — headful 필수
        self._context = self._pw.chromium.launch_persistent_context(
            user_data_dir=str(PROFILE_DIR),
            headless=False,  # 티스토리는 headless 차단
            args=[
                "--disable-blink-features=AutomationControlled",
                "--no-sandbox",
            ],
            user_agent=(
                "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/124.0.0.0 Safari/537.36"
            ),
        )
        
        self._page = self._context.new_page()
        return self
    
    def __exit__(self, *args):
        try:
            self._context.close()
        except Exception:
            pass
        self._pw.stop()
```

### 로그인 상태 확인 + 자동 재로그인

```python
def _is_logged_in(self) -> bool:
    """티스토리 관리 페이지 접근 가능 여부로 로그인 판단"""
    try:
        resp = self._page.goto(
            f"https://{BLOG_NAME}.tistory.com/manage",
            wait_until="domcontentloaded",
            timeout=15000,
        )
        # /auth/login으로 리다이렉트되면 로그아웃 상태
        return "/auth/login" not in self._page.url
    except Exception:
        return False

def ensure_logged_in(self) -> bool:
    """로그인 확인, 필요 시 자동 재로그인"""
    if self._is_logged_in():
        return True
    
    log("티스토리 로그인 필요 — 카카오 SSO 시작", "info")
    return self._kakao_login()

def _kakao_login(self) -> bool:
    """카카오 간편로그인 자동 진행"""
    self._page.goto(
        "https://www.tistory.com/auth/kakao",
        wait_until="domcontentloaded",
        timeout=15000,
    )
    
    # 카카오 로그인 버튼 클릭
    kakao_btn = self._page.locator("a.kakao_account")
    if kakao_btn.count() > 0:
        kakao_btn.click()
    
    # 저장된 계정 선택 (카카오 간편로그인)
    on_kakao_now = True
    kakao_stuck_since = None
    intervention_notified = False
    
    deadline = time.time() + 60
    while time.time() < deadline:
        cur_url = self._page.url
        
        # 카카오 도메인에 25초 이상 머물면 추가 인증으로 판단
        if "kakao.com" in cur_url or "kakaocdn.net" in cur_url:
            if kakao_stuck_since is None:
                kakao_stuck_since = time.time()
            elif not intervention_notified and time.time() - kakao_stuck_since > 25:
                self._notify_login_stuck(cur_url)
                intervention_notified = True
        else:
            kakao_stuck_since = None
        
        # 티스토리 manage 페이지 도달 = 로그인 성공
        if "tistory.com/manage" in cur_url:
            log("티스토리 카카오 로그인 성공", "ok")
            return True
        
        time.sleep(1)
    
    # 타임아웃 후 manage 페이지 재확인 (false negative 방지)
    if self._is_logged_in():
        log("timeout 이후 /manage 도달 확인 — 성공으로 처리", "ok")
        return True
    
    log("카카오 로그인 타임아웃", "error")
    return False

def _notify_login_stuck(self, url: str):
    """추가 인증 필요 시 텔레그램으로 즉시 알림"""
    from common.notifier import notify_telegram
    notify_telegram(
        f"⚠️ 티스토리 카카오 로그인 추가 인증 필요\n"
        f"현재 URL: {url}\n"
        f"수동으로 인증을 완료해주세요."
    )
    log("추가 인증 알림 발송", "warn")
```

### 포스트 발행

티스토리 글쓰기는 에디터를 직접 제어합니다.

```python
def publish(self, post: BlogPost, product: Product) -> str:
    """티스토리 글 발행"""
    if not self.ensure_logged_in():
        raise RuntimeError("티스토리 로그인 실패")
    
    # 새 글 작성 페이지
    self._page.goto(
        f"https://{BLOG_NAME}.tistory.com/manage/post/",
        wait_until="domcontentloaded",
        timeout=15000,
    )
    
    # 제목 입력
    title_input = self._page.locator("#title")
    title_input.fill(post.title)
    
    # 기본 모드 → HTML 모드 전환 (코드 삽입이 안정적)
    self._page.locator("button.btn_mode_switch").click()
    self._page.wait_for_selector("textarea#content-html", timeout=5000)
    
    # HTML 콘텐츠 삽입
    self._page.fill("textarea#content-html", post.content)
    
    # 카테고리 선택
    self._select_category("제품리뷰")
    
    # 태그 입력
    tag_input = self._page.locator("#tag")
    for tag in post.tags[:5]:
        tag_input.fill(tag)
        tag_input.press("Enter")
    
    # 공개 발행
    self._page.locator("button.btn_publish").click()
    
    # 발행 완료 URL 추출
    self._page.wait_for_url("**/manage/post/**", timeout=10000)
    return self._page.url

def _select_category(self, category_name: str):
    """카테고리 선택 (없으면 무시)"""
    try:
        cat_select = self._page.locator("select#category")
        cat_select.select_option(label=category_name)
    except Exception:
        pass
```

---

## 실패 사례 & 해결책

**실패 1: headless 모드에서 카카오 로그인 실패**

headless Chrome에서 카카오 로그인을 진행하면 "자동화 환경에서의 접근은 차단됩니다" 메시지가 나옵니다.

→ **해결**: 티스토리는 예외 없이 `headless=False`. 서버 배포 시 Xvfb 가상 디스플레이 필수.

**실패 2: 세션이 2~3주 후 만료**

Persistent Context 프로필이 저장되어 있어도 카카오 세션은 약 2~3주 후 만료됩니다.

→ **해결**: 매주 세션 유효성 검사 스케줄. 만료 감지 시 텔레그램 알림 → 수동 재로그인.

**실패 3: 카카오 추가 인증(2FA) 팝업**

카카오가 비정상적인 로그인으로 판단하면 SMS 인증 또는 푸시 알림을 요구합니다.

→ **해결**: 25초 타임아웃으로 감지 → 텔레그램 알림 → 사람이 직접 처리. 상세 내용은 [T05편](/posts/auto-publishing-t05/) 참고.

---

## 배운 점 / 주의사항

**동일한 IP에서 너무 자주 로그인하면 카카오가 추가 인증을 요구합니다.** 세션을 최대한 오래 유지하는 것이 핵심입니다. 불필요한 재로그인을 피하세요.

**프로필 디렉토리(`PROFILE_DIR`)는 `.gitignore`에 반드시 추가하세요.** 카카오 세션 정보가 들어있습니다.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2. [[02] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기](/posts/auto-publishing-02/)
3. [[03] 쿠팡 크롤링 — Access Denied 뚫기: Chrome CDP로 WAF 우회](/posts/auto-publishing-03/)
4. [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
5. [[05] Claude CLI와 Gemini API로 상품 소개글 자동 생성하기](/posts/auto-publishing-05/)
6. [[06] WordPress REST API로 멀티 사이트 자동 발행 구현](/posts/auto-publishing-06/)
7. **[현재 글] [Auto-Publishing] 티스토리 자동 발행 — Playwright Persistent Context로 Kakao SSO 유지**
8. [[08] 네이버 RSA 암호화 로그인으로 블로그·카페 자동 발행](/posts/auto-publishing-08/)
9. [[09] GitHub Pages 자동 발행 — Jekyll Markdown 자동 커밋·푸시](/posts/auto-publishing-09/)
10. [[10] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest](/posts/auto-publishing-10/)
11. [[11] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행](/posts/auto-publishing-11/)
12. [[12] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기](/posts/auto-publishing-12/)
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T05] Kakao SSO 25초 이상 멈춤 — 추가 인증 감지와 텔레그램 통지](/posts/auto-publishing-t05/)
- [[T01~T08] 전체 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
