---
title: "[Auto-Publishing] 로그인 폼 셀렉터 깨짐 대응 — CSS 다중 폴백과 동적 DOM 변화 처리 패턴"
date: 2026-04-20 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, Playwright, CSS셀렉터, 동적DOM, 로그인자동화]
description: "네이버·카카오 등 플랫폼이 로그인 폼 UI를 변경할 때 Playwright 셀렉터가 깨지는 문제를 다중 폴백 패턴과 동적 DOM 감지로 안정적으로 처리하는 방법을 설명합니다."
---

## 문제 상황

어제까지 잘 되던 자동화가 오늘 갑자기 실패합니다.

```
[ERROR] locator("#id").fill() - Element not found
[ERROR] Timeout 30000ms exceeded waiting for locator("input.login_id")
```

플랫폼이 로그인 페이지를 리뉴얼했습니다. 클래스명이 바뀌거나, 입력 필드가 추가되거나, 버튼 구조가 변경됩니다.

---

## 원인 분석

로그인 폼 변경이 자동화를 깨는 주요 패턴들:

**패턴 1: 클래스명 변경**
```html
<!-- 이전 -->
<input class="login_id" type="text">

<!-- 변경 후 -->
<input class="NM_INPUT_ID" type="text">
```

**패턴 2: 구조 변경**
```html
<!-- 이전: 단일 페이지 -->
<form class="login_form">
  <input type="text" name="id">
  <input type="password" name="pw">
</form>

<!-- 변경 후: 2단계 (ID → 다음 버튼 → PW) -->
<input type="text" placeholder="아이디">
<!-- 다음 버튼 클릭 후 -->
<input type="password" placeholder="비밀번호">
```

**패턴 3: 동적 로딩**

React/Vue로 렌더링되는 페이지는 DOM이 자바스크립트로 늦게 생성됩니다.

---

## 다중 셀렉터 폴백 패턴

한 셀렉터가 실패하면 다음을 시도하는 방식으로 안정성을 높입니다.

```python
# common/playwright_utils.py
from typing import Optional

def find_element(page, selectors: list[str], timeout: int = 5000) -> Optional[object]:
    """여러 셀렉터 중 하나라도 매칭되면 반환"""
    for sel in selectors:
        try:
            loc = page.locator(sel)
            loc.wait_for(state="visible", timeout=timeout)
            if loc.count() > 0:
                return loc.first
        except Exception:
            continue
    return None

def fill_input(page, selectors: list[str], value: str) -> bool:
    """여러 셀렉터로 입력 필드 탐색 후 입력"""
    element = find_element(page, selectors)
    if element:
        element.fill(value)
        return True
    return False
```

### 네이버 로그인 폼 다중 셀렉터

```python
# common/auth.py
NAVER_ID_SELECTORS = [
    "#id",                   # 표준 ID
    "input[name='id']",      # name 속성
    "input[placeholder*='아이디']",  # placeholder 포함
    ".input_id",             # 클래스명 패턴 1
    ".NM_INPUT_ID",          # 클래스명 패턴 2
    "#input_id",             # ID 패턴
]

NAVER_PW_SELECTORS = [
    "#pw",
    "input[name='pw']",
    "input[type='password']",
    "input[placeholder*='비밀번호']",
    ".input_pw",
    ".NM_INPUT_PW",
]

NAVER_LOGIN_BTN_SELECTORS = [
    "#log\\.login",          # ID (이스케이프 필요)
    "button.btn_login",
    "button[type='submit']",
    "input[type='submit']",
    "button:has-text('로그인')",
    "button:has-text('로그인하기')",
]

def naver_login(page, username: str, password: str) -> bool:
    """다중 셀렉터로 안정적인 네이버 로그인"""
    
    # ID 입력
    if not fill_input(page, NAVER_ID_SELECTORS, username):
        log("네이버 ID 입력 필드를 찾지 못했습니다", "error")
        _save_debug_screenshot(page, "naver_login_id_fail")
        return False
    
    time.sleep(random.uniform(0.5, 1.0))
    
    # PW 입력
    if not fill_input(page, NAVER_PW_SELECTORS, password):
        log("네이버 PW 입력 필드를 찾지 못했습니다", "error")
        _save_debug_screenshot(page, "naver_login_pw_fail")
        return False
    
    time.sleep(random.uniform(0.3, 0.7))
    
    # 로그인 버튼 클릭
    btn = find_element(page, NAVER_LOGIN_BTN_SELECTORS)
    if not btn:
        log("네이버 로그인 버튼을 찾지 못했습니다", "error")
        _save_debug_screenshot(page, "naver_login_btn_fail")
        return False
    
    btn.click()
    return True
```

### 카카오 간편로그인 다중 셀렉터

```python
# sources/newspick.py
def _click_kakao_account(page, email: str) -> bool:
    """저장된 카카오 계정 클릭 — 다중 셀렉터 + X버튼 제외"""
    
    selectors = [
        # 가장 신뢰할 수 있는 카카오 클래스
        ".kc_item_select .kc_btn_simple",
        "button.kc_btn_simple",
        
        # 이메일 텍스트로 찾기 (X 버튼 제외)
        f'a:has-text("{email}"):not(.btn_delete)',
        f'button:has-text("{email}"):not(.btn_delete)',
        
        # 첫 번째 계정 선택 (이메일 무관)
        ".kc_list_wrap li:first-child a:not(.btn_delete)",
        ".account-list > li:first-child button:not(.delete)",
    ]
    
    for sel in selectors:
        loc = page.locator(sel)
        if loc.count() > 0:
            try:
                # 네비게이션 발생 여부 검증 (false positive 방지)
                with page.expect_navigation(timeout=8000):
                    loc.first.click(timeout=2000)
                return True
            except Exception:
                continue
    
    return False
```

---

## 동적 DOM 처리

### wait_for_selector 전략

```python
def wait_for_login_form(page, timeout: int = 15000) -> bool:
    """로그인 폼이 완전히 렌더링될 때까지 대기"""
    
    # 여러 가능한 폼 마커 중 하나가 나타나면 진행
    form_markers = [
        "#id",
        "input[name='id']",
        "input[type='email']",
        ".login-form",
        "#loginForm",
    ]
    
    for marker in form_markers:
        try:
            page.wait_for_selector(marker, state="visible", timeout=timeout // len(form_markers))
            return True
        except Exception:
            continue
    
    return False
```

### 2단계 로그인 폼 처리

ID 입력 → 다음 버튼 → PW 입력 구조:

```python
def handle_two_step_login(page, username: str, password: str) -> bool:
    """2단계 로그인 폼 자동 감지 및 처리"""
    
    # ID 입력
    id_field = find_element(page, NAVER_ID_SELECTORS)
    if not id_field:
        return False
    id_field.fill(username)
    
    # "다음" 버튼이 있으면 2단계 방식
    next_btn_selectors = [
        "button:has-text('다음')",
        "button[data-action='next']",
        ".btn_next",
    ]
    next_btn = find_element(page, next_btn_selectors, timeout=2000)
    
    if next_btn:
        log("2단계 로그인 폼 감지 — '다음' 클릭", "info")
        next_btn.click()
        
        # PW 필드 대기
        page.wait_for_selector("input[type='password']", timeout=10000)
    
    # PW 입력
    pw_field = find_element(page, NAVER_PW_SELECTORS)
    if not pw_field:
        return False
    pw_field.fill(password)
    
    # 로그인 버튼
    btn = find_element(page, NAVER_LOGIN_BTN_SELECTORS)
    if btn:
        btn.click()
        return True
    
    return False
```

---

## 실패 시 디버그 자동화

셀렉터 실패 시 스크린샷을 저장해 나중에 원인 파악이 가능합니다.

```python
def _save_debug_screenshot(page, name: str):
    """실패 시 스크린샷 + HTML 저장"""
    debug_dir = Path(".debug")
    debug_dir.mkdir(exist_ok=True)
    
    ts = int(time.time())
    
    # 스크린샷
    page.screenshot(path=str(debug_dir / f"{name}_{ts}.png"), full_page=True)
    
    # HTML 소스 (셀렉터 분석용)
    (debug_dir / f"{name}_{ts}.html").write_text(
        page.content(), encoding="utf-8"
    )
    
    log(f"디버그 파일 저장: .debug/{name}_{ts}.*", "info")
    
    # 텔레그램으로 스크린샷 전송
    notify_telegram_photo(
        str(debug_dir / f"{name}_{ts}.png"),
        caption=f"🔍 셀렉터 실패 디버그: {name}"
    )
```

---

## 배운 점 / 주의사항

**셀렉터는 최소 3개 이상 준비하세요.** `id`, `name 속성`, `placeholder 텍스트` 조합이 가장 안정적입니다. 클래스명은 가장 자주 바뀝니다.

**`page.content()`를 주기적으로 저장하세요.** 셀렉터가 깨졌을 때 HTML을 분석해서 새 셀렉터를 찾아야 합니다. 디버그 스크린샷만으론 부족합니다.

---

## 관련 포스팅

- [[08] 네이버 RSA 암호화 로그인으로 블로그·카페 자동 발행](/posts/auto-publishing-08/)
- [[T03] 세션 만료와 자동 재로그인](/posts/auto-publishing-t03/)
- [[T08] 자동화 탐지 우회 종합](/posts/auto-publishing-t08/)
