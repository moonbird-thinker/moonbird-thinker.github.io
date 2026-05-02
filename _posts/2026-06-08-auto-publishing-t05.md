---
title: "[Auto-Publishing] Kakao SSO 25초 이상 멈춤 — 추가 인증 감지와 텔레그램 즉시 통지 자동화"
date: 2026-06-08 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, 카카오SSO, 추가인증, 텔레그램알림, 티스토리자동화]
description: "티스토리 자동 발행 중 카카오 SSO가 25초 이상 멈추는 경우를 추가 인증으로 판단하고, 텔레그램으로 즉시 알림을 보내 수동 개입을 유도하는 로직을 설명합니다."
---

## 문제 상황

티스토리 자동 로그인 중 가끔 아무 일도 일어나지 않는 구간이 생깁니다.

```
[INFO] 티스토리 카카오 로그인 시작
[INFO] Kakao 로그인 버튼 클릭
... (25초 후)
[WARN] 25초 경과 — 카카오 도메인에서 진행 없음
[WARN] 추가 인증 필요로 판단 — 텔레그램 알림 발송
```

카카오가 "의심스러운 로그인"으로 판단하면 **SMS 인증**, **카카오 앱 푸시 알림**, 또는 **추가 본인확인**을 요구합니다. 자동화 스크립트는 이것을 처리하지 못하고 멈춥니다.

---

## 원인 분석

카카오는 다음 상황에서 추가 인증을 요구합니다:

**환경 변화 감지:**
- 새로운 IP 주소
- 비정상적인 로그인 시간 (새벽 3시 등)
- 오랫동안 사용하지 않은 기기

**보안 정책:**
- 특정 기간마다 보안 재확인
- 계정 보호 강화 정책 활성화
- 다중 기기 로그인 감지

---

## 핵심 구현: 25초 타이머 감지

```python
# publishers/tistory.py

def _kakao_login(self) -> bool:
    """카카오 SSO 로그인 — 추가 인증 감지 포함"""
    
    # 카카오 로그인 시작
    self._page.goto(
        "https://www.tistory.com/auth/kakao",
        wait_until="domcontentloaded",
        timeout=15000,
    )
    
    kakao_stuck_since = None      # 카카오 도메인에서 멈춘 시작 시간
    intervention_notified = False  # 알림 발송 여부 (중복 방지)
    
    KAKAO_TIMEOUT = 60  # 전체 타임아웃
    STUCK_THRESHOLD = 25  # 이 시간 이상 멈추면 추가 인증으로 판단
    
    deadline = time.time() + KAKAO_TIMEOUT
    
    while time.time() < deadline:
        cur_url = self._page.url
        
        # 카카오 도메인 여부 확인
        is_on_kakao = any(
            domain in cur_url
            for domain in ["kakao.com", "kakaocdn.net", "kakaoenterprise.com"]
        )
        
        if is_on_kakao:
            # 카카오 도메인에 처음 들어온 시점 기록
            if kakao_stuck_since is None:
                kakao_stuck_since = time.time()
            
            # 25초 이상 카카오에 머물고 있으면 추가 인증으로 판단
            elapsed = time.time() - kakao_stuck_since
            
            if elapsed > STUCK_THRESHOLD and not intervention_notified:
                self._notify_login_stuck(cur_url, elapsed)
                intervention_notified = True
                
                # 추가 인증 화면 스크린샷 저장 (디버깅용)
                ts = int(time.time())
                screenshot_path = Path(f".debug/kakao_stuck_{ts}.png")
                screenshot_path.parent.mkdir(exist_ok=True)
                self._page.screenshot(path=str(screenshot_path))
                log(f"스크린샷 저장: {screenshot_path}", "info")
        
        else:
            # 카카오를 벗어난 경우 타이머 리셋
            kakao_stuck_since = None
        
        # 목표 달성 확인: 티스토리 manage 도달
        if "tistory.com/manage" in cur_url:
            log("티스토리 카카오 로그인 성공", "ok")
            return True
        
        time.sleep(1)
    
    # 타임아웃 후 최종 확인 (false negative 방지)
    if self._is_logged_in():
        log("timeout 이후 /manage 도달 확인 — 성공으로 처리", "ok")
        return True
    
    log(f"카카오 로그인 타임아웃 ({KAKAO_TIMEOUT}초)", "error")
    notify_telegram("❌ 티스토리 카카오 로그인 최종 실패 — 수동 확인 필요")
    return False

def _notify_login_stuck(self, url: str, elapsed: float):
    """추가 인증 감지 시 즉시 알림"""
    message = (
        f"⚠️ <b>티스토리 카카오 추가 인증 감지</b>\n\n"
        f"• 경과 시간: {elapsed:.0f}초\n"
        f"• 현재 URL: {url[:100]}\n\n"
        f"카카오 앱 또는 SMS를 확인하고\n"
        f"인증을 완료해주세요.\n\n"
        f"완료 후 자동으로 재개됩니다. ⏳"
    )
    
    notify_telegram(message)
    notify_kakao(f"티스토리 카카오 추가 인증 필요 — 앱/SMS 확인하세요")
    
    log("추가 인증 알림 발송 완료", "warn")
```

---

## 추가 인증 유형별 대응

### SMS 인증 (자동 처리 불가)

카카오가 등록된 전화번호로 6자리 코드를 보냅니다.

```
화면: "인증번호를 입력해주세요"
필요한 입력: SMS 수신 후 6자리 코드
```

→ **현실적 대응**: 텔레그램 알림 → 사람이 핸드폰 확인 → 화면에 직접 입력 → 자동화 재개

### 카카오 앱 푸시 (자동 처리 가능성 있음)

```python
# 이론적으로는 카카오 앱 API로 승인 가능하지만
# 카카오 공식 API에는 이 기능이 없음
# → 사람이 직접 앱에서 승인해야 함
```

### 비밀번호 재확인 (자동 처리 가능)

가끔 비밀번호 재입력을 요구합니다. 이것은 자동화가 가능합니다.

```python
def _handle_password_reconfirm(self) -> bool:
    """비밀번호 재확인 팝업 처리"""
    pw_input = self._page.locator("input[type='password']")
    
    if pw_input.count() > 0 and pw_input.is_visible():
        log("비밀번호 재확인 화면 감지 — 자동 입력", "info")
        pw_input.fill(KAKAO_PASSWORD)
        self._page.locator("button[type='submit']").click()
        return True
    
    return False
```

---

## 발생 빈도 줄이기

**전략 1: 같은 IP 유지**

카카오는 IP 변경 시 추가 인증을 요구하는 경향이 있습니다. 가능하면 서버 IP를 고정하세요.

**전략 2: 로그인 빈도 줄이기**

Persistent Context로 세션을 최대한 오래 유지하면 로그인 횟수가 줄고, 추가 인증 빈도도 줄어듭니다.

**전략 3: 자연스러운 로그인 시간**

새벽 3시에 로그인하는 것은 의심스럽습니다. 파이프라인 시작 시간을 낮 시간대로 조정하거나, 세션 체크는 낮에 하고 발행만 예약 시간으로 처리합니다.

---

## 모니터링: 추가 인증 발생 이력 추적

```python
# common/event_log.py
def log_kakao_intervention(url: str, elapsed: float, resolved: bool):
    """추가 인증 이벤트 기록"""
    EVENTS_FILE = Path(".data/kakao_interventions.jsonl")
    
    event = {
        "timestamp": datetime.now().isoformat(),
        "url": url,
        "elapsed_sec": elapsed,
        "resolved": resolved,
    }
    
    with open(EVENTS_FILE, "a") as f:
        f.write(json.dumps(event, ensure_ascii=False) + "\n")
    
    # 월 3회 이상이면 카카오 보안 설정 검토 권고
    recent_events = _count_recent_events(EVENTS_FILE, days=30)
    if recent_events >= 3:
        notify_telegram(
            f"📊 이번 달 카카오 추가 인증 {recent_events}회 발생\n"
            f"카카오 계정 보안 설정 검토를 권장합니다."
        )
```

---

## 관련 포스팅

- [[07] 티스토리 자동 발행 — Playwright Persistent Context로 Kakao SSO 유지](/posts/auto-publishing-07/)
- [[T03] 세션 만료와 자동 재로그인](/posts/auto-publishing-t03/)
- [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
