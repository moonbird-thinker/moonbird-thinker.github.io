---
title: "[Auto-Publishing] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현"
date: 2026-04-13 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, 텔레그램봇, 카카오알림, OAuth자동갱신]
description: "자동 발행 시스템의 모니터링 핵심인 텔레그램 봇 알림과 카카오톡 알림을 병행하는 방법을 설명합니다. Kakao OAuth 토큰 자동 갱신과 카카오 캘린더 연동까지 다룹니다."
---

## 개요

자동화 시스템에서 가장 무서운 것은 **조용히 실패하는 것**입니다. 시스템이 작동을 멈췄는데 며칠 후에야 알게 되는 상황을 방지해야 합니다.

이번 편에서는 두 가지 알림 채널을 병행합니다:
- **텔레그램**: 개발자용 기술 알림 (상세 로그)
- **카카오톡**: 간단한 성공/실패 요약

두 채널을 병행하는 이유는 **텔레그램을 사용하지 않는 시간**에도 카카오톡으로 빠르게 인지하기 위해서입니다.

---

## 핵심 구현

### 텔레그램 봇 알림

```python
# common/notifier.py
import requests

TELEGRAM_BOT_TOKEN = "YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID = "YOUR_CHAT_ID"

def notify_telegram(message: str, parse_mode: str = "HTML") -> bool:
    """텔레그램 메시지 발송"""
    try:
        resp = requests.post(
            f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage",
            json={
                "chat_id": TELEGRAM_CHAT_ID,
                "text": message,
                "parse_mode": parse_mode,
            },
            timeout=10,
        )
        return resp.status_code == 200
    except Exception as e:
        # 알림 실패는 무시 (알림이 시스템을 죽이면 안 됨)
        print(f"텔레그램 알림 실패: {e}")
        return False

def notify_pipeline_result(
    pipeline_name: str,
    success: bool,
    published_count: int = 0,
    error: str = None,
):
    """파이프라인 실행 결과 알림"""
    if success:
        msg = (
            f"✅ <b>{pipeline_name}</b> 완료\n"
            f"📝 발행: {published_count}편\n"
            f"🕐 {datetime.now().strftime('%Y-%m-%d %H:%M')}"
        )
    else:
        msg = (
            f"❌ <b>{pipeline_name}</b> 실패\n"
            f"⚠️ 오류: {error or '알 수 없음'}\n"
            f"🕐 {datetime.now().strftime('%Y-%m-%d %H:%M')}"
        )
    
    notify_telegram(msg)
```

### 카카오톡 알림 메시지

카카오 API는 내 계정에 나에게 메시지를 보낼 수 있습니다.

```python
# common/kakao_notify.py
import requests
import json

KAKAO_API = "https://kapi.kakao.com/v2/api/talk/memo/default/send"

def notify_kakao(message: str) -> bool:
    """카카오톡 나에게 메시지 전송"""
    access_token = get_access_token()  # OAuth 토큰
    
    template = {
        "object_type": "text",
        "text": message,
        "link": {
            "web_url": "https://myblog.github.io",
            "mobile_web_url": "https://myblog.github.io",
        },
    }
    
    resp = requests.post(
        KAKAO_API,
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/x-www-form-urlencoded",
        },
        data={"template_object": json.dumps(template, ensure_ascii=False)},
        timeout=10,
    )
    
    return resp.status_code == 200
```

### Kakao OAuth 토큰 자동 갱신

카카오 Access Token은 약 6시간마다 만료됩니다. Refresh Token으로 자동 갱신합니다.

```python
# common/kakao_auth.py
import json
from pathlib import Path

TOKEN_FILE = Path(".sessions/kakao_token.json")
KAKAO_TOKEN_URL = "https://kauth.kakao.com/oauth/token"

def get_access_token() -> str:
    """유효한 access_token 반환 (만료 시 자동 갱신)"""
    token_data = _load_token()
    
    # 만료 여부 확인 (만료 10분 전에 미리 갱신)
    expires_at = token_data.get("expires_at", 0)
    if time.time() + 600 > expires_at:
        token_data = refresh_access_token()
    
    return token_data["access_token"]

def refresh_access_token() -> dict:
    """refresh_token으로 새 access_token 발급"""
    token_data = _load_token()
    refresh_token = token_data.get("refresh_token")
    
    if not refresh_token:
        raise RuntimeError("카카오 refresh_token이 없습니다. 재인증이 필요합니다.")
    
    resp = requests.post(
        KAKAO_TOKEN_URL,
        data={
            "grant_type": "refresh_token",
            "client_id": KAKAO_CLIENT_ID,
            "refresh_token": refresh_token,
        },
        timeout=10,
    )
    
    if resp.status_code != 200:
        raise RuntimeError(f"토큰 갱신 실패: {resp.text}")
    
    new_tokens = resp.json()
    
    # 기존 토큰 파일 업데이트
    token_data["access_token"] = new_tokens["access_token"]
    token_data["expires_at"] = time.time() + new_tokens["expires_in"]
    
    # refresh_token도 새로 발급됐으면 업데이트
    if "refresh_token" in new_tokens:
        token_data["refresh_token"] = new_tokens["refresh_token"]
        token_data["refresh_token_expires_at"] = (
            time.time() + new_tokens.get("refresh_token_expires_in", 5184000)
        )
    
    _save_token(token_data)
    log("카카오 access_token 자동 갱신 완료", "ok")
    return token_data

def _request_with_auth_retry(method: str, url: str, **kwargs) -> requests.Response:
    """401 발생 시 토큰 갱신 후 1회 재시도"""
    token = get_access_token()
    headers = kwargs.pop("headers", {})
    headers["Authorization"] = f"Bearer {token}"
    
    resp = requests.request(method, url, headers=headers, **kwargs)
    
    if resp.status_code == 401:
        log("카카오 401 — access_token 갱신 후 재시도", "warn")
        new_token = refresh_access_token()["access_token"]
        headers["Authorization"] = f"Bearer {new_token}"
        resp = requests.request(method, url, headers=headers, **kwargs)
    
    return resp
```

### 카카오 캘린더 연동 (발행 일정 자동 기록)

```python
# common/kakao_calendar.py
CALENDAR_API = "https://kapi.kakao.com/v2/api/calendar"

def add_publish_event(title: str, url: str, published_at: datetime):
    """발행 완료 이벤트를 카카오 캘린더에 자동 기록"""
    event = {
        "title": f"📝 자동발행 — {title[:30]}",
        "time": {
            "start_at": published_at.strftime("%Y%m%dT%H%M%S"),
            "end_at": published_at.strftime("%Y%m%dT%H%M%S"),
            "time_zone": "Asia/Seoul",
            "all_day": False,
        },
        "description": f"발행 URL: {url}",
        "color": "GREEN",
    }
    
    _request_with_auth_retry(
        "POST",
        f"{CALENDAR_API}/create/event",
        json={"calendar_id": "primary", "event": event},
        timeout=10,
    )
```

---

## 실패 사례 & 해결책

**실패 1: 텔레그램 메시지 폭탄**

파이프라인 10개가 동시에 실패하면 10개의 알림이 한꺼번에 옵니다. 알림 피로(alert fatigue)가 생겨 알림을 무시하게 됩니다.

→ **해결**: 같은 파이프라인의 연속 실패는 첫 번째만 알림. 이후 10번 실패마다 요약 알림.

**실패 2: Kakao refresh_token도 만료됨**

Access Token은 자동 갱신되지만, Refresh Token도 2개월마다 만료됩니다.

→ **해결**: Refresh Token 만료 7일 전에 텔레그램 알림. 수동으로 카카오 재인증 필요를 미리 알림.

---

## 배운 점 / 주의사항

**알림은 행동 가능한 정보만 담으세요.** "파이프라인 실패"만으로는 뭘 해야 할지 모릅니다. "티스토리 Kakao SSO 만료 — .sessions/tistory_profile 삭제 후 재로그인 필요"처럼 구체적으로 쓰세요.

**토큰 파일은 반드시 암호화하거나 접근 권한을 제한하세요.** 카카오 토큰이 노출되면 계정이 탈취될 수 있습니다.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1~12편 — [[01] 전체 아키텍처](/posts/auto-publishing-01/) 부터 [[12] Registry 패턴](/posts/auto-publishing-12/) 까지

13. **[현재 글] [Auto-Publishing] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현**
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T01~T08] 전체 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
