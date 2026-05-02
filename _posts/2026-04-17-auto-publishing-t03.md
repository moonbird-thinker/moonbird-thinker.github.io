---
title: "[Auto-Publishing] 세션 만료와 자동 재로그인 — Persistent Profile vs requests.Session 실전 비교"
date: 2026-04-17 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, 세션만료, Playwright, requests세션, 자동재로그인, 쿠키영속화]
description: "자동 발행 시스템에서 가장 흔한 장애 원인인 세션 만료를 플랫폼별로 감지하고 자동으로 재로그인하는 전략을 비교합니다. Playwright Persistent Context와 requests.Session의 선택 기준을 설명합니다."
---

## 문제 상황

새벽 3시에 파이프라인이 돌아야 하는데, 로그를 보니 모든 발행이 실패했습니다.

```
[ERROR] 티스토리: /auth/login으로 리다이렉트 감지 — 세션 만료
[ERROR] 알리익스프레스: storage_state 파싱 실패 — 만료된 세션
[ERROR] 뉴스픽: API 응답이 HTML (로그인 페이지)
```

세션이 다 만료된 것입니다. 자동화 시스템에서 가장 많이 만나는 장애입니다.

---

## 세션 만료 유형별 분석

### 유형 1: HTTP 세션 (requests.Session + pickle)

가장 단순한 방식입니다. 로그인 후 쿠키를 pickle 파일로 저장합니다.

```python
# 세션 저장
with open(".sessions/naver.pkl", "wb") as f:
    pickle.dump(session.cookies, f)

# 세션 복원
with open(".sessions/naver.pkl", "rb") as f:
    session.cookies.update(pickle.load(f))
```

**만료 패턴:**
- 쿠키 `expires` 날짜가 지남
- 서버에서 세션 ID를 무효화
- IP 변경 시 세션 무효화 (네이버 등)

**만료 감지 방법:**

```python
def is_session_valid(session: requests.Session, check_url: str) -> bool:
    try:
        resp = session.get(check_url, timeout=5, allow_redirects=True)
        
        # 로그인 페이지로 리다이렉트 됐는지 확인
        if "login" in resp.url or "signin" in resp.url:
            return False
        
        # 응답 코드 확인
        if resp.status_code in (401, 403):
            return False
        
        # 로그인 마커 확인 (플랫폼별)
        login_markers = ["로그아웃", "내 계정", "MY"]
        return any(m in resp.text for m in login_markers)
    except Exception:
        return False
```

### 유형 2: Playwright storage_state (알리익스프레스)

`storage_state.json`에 쿠키 + localStorage를 저장합니다.

**만료 감지 방법:**

```python
def is_aliexpress_session_valid(page) -> bool:
    """알리 내부 API 응답으로 세션 유효성 확인"""
    try:
        resp = page.request.get(
            "https://www.aliexpress.com/af/icbu-auth.json",
            timeout=5000,
        )
        body = resp.text()
        
        # HTML이 반환되면 만료 (JSON이어야 정상)
        if not body.strip().startswith("{"):
            if "login" in body.lower() or "sign in" in body.lower():
                log("알리 세션 만료 — JSON 대신 HTML 로그인 페이지", "warn")
                return False
        
        return True
    except Exception:
        return False
```

### 유형 3: Playwright Persistent Context (티스토리)

프로필 디렉토리에 세션이 저장됩니다.

**만료 감지 방법:**

```python
def is_tistory_session_valid(page) -> bool:
    """manage 접근 여부로 티스토리 로그인 상태 확인"""
    try:
        page.goto(
            "https://myblog.tistory.com/manage",
            wait_until="domcontentloaded",
            timeout=10000,
        )
        # 로그인 페이지로 리다이렉트 됐으면 만료
        return "/auth/login" not in page.url
    except Exception:
        return False
```

---

## 플랫폼별 세션 지속 시간

| 플랫폼 | 방식 | 평균 지속 | 갱신 방법 |
|--------|------|---------|---------|
| 네이버 | pickle 쿠키 | 1~2주 | CDP 쿠키 재추출 또는 RSA 재로그인 |
| 알리익스프레스 | storage_state | 3~7일 | 재로그인 + storage_state 재생성 |
| 티스토리 | persistent context | 2~3주 | Kakao SSO 재인증 |
| 뉴스픽 | pickle 쿠키 | 1~4주 | Playwright 재로그인 |
| WordPress | App Password | 만료 없음 | 변경 없음 |

---

## 자동 재로그인 전략

### 공통 패턴: 감지 → 갱신 → 재시도

```python
# common/session_guard.py
from functools import wraps

def with_session_renewal(check_fn, login_fn):
    """세션 만료 시 자동 재로그인 데코레이터"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # 실행 전 세션 확인
            if not check_fn():
                log("세션 만료 감지 — 재로그인 시도", "warn")
                if not login_fn():
                    raise RuntimeError("재로그인 실패")
                log("재로그인 성공", "ok")
            
            # 실행
            try:
                return func(*args, **kwargs)
            except SessionExpiredError:
                # 실행 중 만료된 경우도 처리
                log("실행 중 세션 만료 — 재로그인 후 1회 재시도", "warn")
                if login_fn():
                    return func(*args, **kwargs)
                raise
        
        return wrapper
    return decorator
```

### 알리익스프레스 자동 재로그인

```python
def _reset_session(self, wipe_storage: bool = True) -> bool:
    """storage_state 폐기 + 브라우저 재시작 + 재로그인"""
    
    log("알리 세션 초기화 시작", "warn")
    
    # 1. 브라우저 종료
    self.close()
    
    # 2. 손상된 세션 파일 삭제
    if wipe_storage:
        for path in (STORAGE_PATH, COOKIE_PATH):
            if path.exists():
                path.unlink()
                log(f"세션 파일 삭제: {path.name}", "info")
    
    # 3. 재로그인
    if not self._relogin():
        notify_telegram("❌ 알리익스프레스 재로그인 실패 — 수동 개입 필요")
        return False
    
    self._session_reset_done = True
    log("알리 세션 초기화 완료", "ok")
    return True

def _relogin(self) -> bool:
    """브라우저 재시작 + 알리 로그인"""
    from common.aliexpress_login import AliExpressLogin
    
    try:
        login = AliExpressLogin()
        success = login.login(headless=False)  # 로그인은 headful
        
        if success:
            # 새 storage_state 저장
            login.context.storage_state(path=str(STORAGE_PATH))
            log("알리 재로그인 + storage_state 저장 완료", "ok")
        
        return success
    except Exception as e:
        log(f"알리 재로그인 실패: {e}", "error")
        return False
```

---

## 재발 방지: 선제적 세션 갱신

만료 직전에 미리 갱신하면 실행 중 실패를 막을 수 있습니다.

```python
# scheduler.py에 추가
def session_health_check():
    """매일 새벽 2시 세션 유효성 전수 검사"""
    results = {}
    
    # 각 플랫폼 세션 확인
    platforms = {
        "aliexpress": check_aliexpress_session,
        "tistory": check_tistory_session,
        "naver": check_naver_session,
        "newspick": check_newspick_session,
    }
    
    failed = []
    for name, check_fn in platforms.items():
        if not check_fn():
            failed.append(name)
    
    if failed:
        notify_telegram(
            f"⚠️ 세션 만료 감지\n"
            f"플랫폼: {', '.join(failed)}\n"
            f"자동 갱신을 시도합니다..."
        )
        # 갱신 시도
        for name in failed:
            renew_session(name)

# 매일 새벽 2시에 실행
schedule.every().day.at("02:00").do(session_health_check)
```

---

## 배운 점

**세션 만료는 "언제" 일어나는지가 중요합니다.** 자동화 시스템이 돌기 30분 전에 세션 체크를 넣으면 실행 중 실패를 크게 줄일 수 있습니다.

**재로그인이 실패하면 사람이 개입해야 합니다.** 완전 자동화가 목표지만, 카카오 추가 인증 같은 경우는 사람이 처리해야 합니다. 텔레그램 알림으로 빠르게 대응하는 것이 현실적입니다.

---

## 관련 포스팅

- [[T01] 쿠팡 크롤링 403 Access Denied 완전 해부](/posts/auto-publishing-t01/)
- [[T05] Kakao SSO 25초 이상 멈춤 — 추가 인증 감지](/posts/auto-publishing-t05/)
- [[T07] AliExpress 5xx 오류 복구 — warmup 세션 재구성](/posts/auto-publishing-t07/)
