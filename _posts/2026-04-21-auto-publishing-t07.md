---
title: "[Auto-Publishing] AliExpress 5xx 오류 복구 — warmup 세션 재구성과 storage_state 초기화 전략"
date: 2026-04-21 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, 알리익스프레스, 5xx오류, ERR_HTTP_RESPONSE_CODE_FAILURE, 세션초기화]
description: "알리익스프레스 크롤링 중 발생하는 5xx 서버 오류와 ERR_HTTP_RESPONSE_CODE_FAILURE를 warmup 세션 재구성과 storage_state 초기화로 복구하는 2단계 전략을 설명합니다."
---

## 문제 상황

알리익스프레스 크롤러가 갑자기 이런 오류를 냅니다.

```
playwright._impl._errors.Error: net::ERR_HTTP_RESPONSE_CODE_FAILURE at https://www.aliexpress.com/item/1005006...
```

HTTP 5xx 오류(500, 503)가 발생했을 때 Playwright가 던지는 예외입니다. 알리익스프레스가 서버 오류를 반환한 것이지만, 실제로는 **봇으로 판단해서 차단**하는 경우가 대부분입니다.

---

## 원인 분석

`ERR_HTTP_RESPONSE_CODE_FAILURE`가 발생하는 실제 이유:

**1. 봇 감지 후 5xx로 응답**

Cloudflare나 알리의 Baxia가 봇을 감지했을 때 페이지를 반환하는 대신 HTTP 503을 반환합니다.

**2. 세션 손상**

오래된 storage_state 파일이 알리 서버와 불일치할 때 서버가 요청을 거부합니다.

**3. 실제 서버 오류**

알리 서버가 일시적으로 과부하 상태일 때도 5xx가 납니다.

세 가지 경우를 구분해서 처리해야 합니다.

---

## 2단계 복구 전략

```python
# sources/aliexpress.py

def _goto_with_retry(self, url: str, retries: int = 2) -> bool:
    """페이지 이동 — ERR_HTTP_RESPONSE_CODE_FAILURE 시 2단계 복구"""
    
    for attempt in range(retries + 1):
        try:
            self._page.goto(
                url,
                timeout=30000,
                wait_until="domcontentloaded",
            )
            
            # CAPTCHA 체크
            if self._is_captcha_page():
                if not self._wait_for_captcha_solve():
                    return False
            
            return True
            
        except Exception as e:
            msg = str(e)
            log(f"goto 실패 [{attempt+1}/{retries+1}]: {msg[:120]}", "warn")
            
            # ERR_HTTP_RESPONSE_CODE_FAILURE가 아닌 다른 오류면 즉시 포기
            if "ERR_HTTP_RESPONSE_CODE_FAILURE" not in msg:
                log(f"복구 불가능한 오류 유형 — 포기", "error")
                return False
            
            # 최대 재시도 횟수 초과
            if attempt >= retries:
                log("재시도 횟수 초과 — 포기", "error")
                return False
            
            # 1차 복구: warmup으로 세션 활성화
            if attempt == 0:
                log("1차 복구: warmup 세션 재구성 시도", "info")
                self._warmup_session()
                time.sleep(3)
            
            # 2차 복구: storage_state 초기화 + 재로그인
            elif not self._session_reset_done:
                log("2차 복구: storage_state 초기화 + 재로그인 시도", "warn")
                if not self._reset_session():
                    log("세션 초기화 실패 — 포기", "error")
                    return False
                time.sleep(3)
    
    return False
```

---

## 1차 복구: warmup 세션 재구성

5xx가 발생하면 먼저 알리익스프레스를 "자연스럽게" 탐색해서 세션을 활성화합니다.

```python
def _warmup_session(self):
    """자연스러운 탐색 패턴으로 세션 워밍업"""
    
    log("warmup 시작: 메인 → 카테고리 탐색", "info")
    
    warmup_steps = [
        # 1. 메인 방문
        ("https://www.aliexpress.com", 2.0, 4.0),
        # 2. 인기 카테고리 방문
        ("https://www.aliexpress.com/category/100003109", 2.0, 3.0),
        # 3. 검색 페이지 (크롤링 목표와 관련된 카테고리)
        ("https://www.aliexpress.com/popular.htm", 1.5, 2.5),
    ]
    
    for url, min_wait, max_wait in warmup_steps:
        try:
            self._page.goto(url, timeout=15000, wait_until="domcontentloaded")
            
            # 스크롤 (사람처럼)
            self._page.evaluate(f"window.scrollBy(0, {random.randint(200, 400)})")
            
            time.sleep(random.uniform(min_wait, max_wait))
            
            # CAPTCHA 발생 시 즉시 중단
            if self._is_captcha_page():
                log("warmup 중 CAPTCHA 발생", "warn")
                break
                
        except Exception as e:
            log(f"warmup 단계 실패: {e}", "warn")
            break
    
    log("warmup 완료", "info")
```

---

## 2차 복구: storage_state 초기화

warmup으로 해결이 안 되면 세션 파일을 폐기하고 재로그인합니다.

```python
def _reset_session(self, wipe_storage: bool = True) -> bool:
    """세션 완전 초기화"""
    
    log("세션 초기화 시작", "warn")
    
    # 1. 현재 브라우저 종료
    self._close_browser()
    
    # 2. storage_state 파일 삭제
    if wipe_storage:
        for path in (STORAGE_PATH, COOKIE_PATH):
            if path.exists():
                path.unlink()
                log(f"세션 파일 삭제: {path.name}", "info")
    
    # 3. 재로그인 시도
    success = self._relogin()
    
    if success:
        self._session_reset_done = True  # 이번 실행에서 1회만 초기화
        log("세션 초기화 완료", "ok")
        notify_telegram("ℹ️ 알리익스프레스 세션 초기화 및 재로그인 완료")
    else:
        notify_telegram("❌ 알리익스프레스 재로그인 실패 — 수동 개입 필요")
    
    return success

def _relogin(self) -> bool:
    """브라우저 재시작 + 알리익스프레스 로그인"""
    from common.aliexpress_login import AliExpressLogin
    
    try:
        login_handler = AliExpressLogin()
        
        # 로그인은 항상 headful (CAPTCHA 대응)
        success = login_handler.login(headless=False)
        
        if success:
            # 새 storage_state 저장
            login_handler.context.storage_state(path=str(STORAGE_PATH))
            log("알리 재로그인 + storage_state 저장 완료", "ok")
            
            # 새 브라우저 컨텍스트로 연결
            self._init_browser()
        
        login_handler.close()
        return success
        
    except Exception as e:
        log(f"재로그인 예외: {e}", "error")
        return False
```

---

## 5xx vs 봇 차단 구분

동일한 `ERR_HTTP_RESPONSE_CODE_FAILURE`지만 실제 서버 오류와 봇 차단은 다릅니다.

```python
def _diagnose_5xx(self, url: str) -> str:
    """5xx 오류 원인 진단"""
    
    # requests로 동일한 URL 직접 접속 시도 (User-Agent 최소화)
    try:
        raw_resp = requests.get(
            url,
            headers={"User-Agent": "curl/7.64.1"},
            timeout=10,
        )
        
        if raw_resp.status_code == 200:
            # requests로는 성공 → 봇 차단 (브라우저 지문 문제)
            return "bot_detection"
        elif raw_resp.status_code in (500, 502, 503, 504):
            # requests도 실패 → 실제 서버 오류
            return "server_error"
    except Exception:
        pass
    
    return "unknown"

# 진단 결과에 따른 처리
diagnosis = self._diagnose_5xx(url)

if diagnosis == "server_error":
    # 실제 서버 오류 — 30분 후 재시도
    log("알리 서버 오류 — 30분 후 재시도 예약", "warn")
    schedule_retry(url, delay_minutes=30)
    
elif diagnosis == "bot_detection":
    # 봇 차단 — 세션 초기화
    log("봇 차단 감지 — 세션 초기화", "warn")
    self._reset_session()
```

---

## 재발 방지

**오류율 모니터링:**

```python
error_count = 0
total_count = 0

def track_5xx_rate():
    rate = error_count / max(total_count, 1)
    if rate > 0.3:  # 30% 이상이면 경고
        notify_telegram(
            f"⚠️ 알리 5xx 오류율 {rate:.1%}\n"
            f"세션 전략 재검토 필요"
        )
```

---

## 관련 포스팅

- [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
- [[T02] 알리익스프레스 CAPTCHA 슬라이더 자동 감지](/posts/auto-publishing-t02/)
- [[T03] 세션 만료와 자동 재로그인](/posts/auto-publishing-t03/)
