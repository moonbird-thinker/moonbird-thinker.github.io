---
title: "[Auto-Publishing] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright"
date: 2026-04-14 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, 웹크롤링인증, RSA암호화, HMAC, JWT, Playwright]
description: "자동 발행 시스템에서 사용한 모든 인증 전략을 플랫폼별로 총정리합니다. CDP, RSA, HMAC-SHA256, JWT Bearer, Playwright Persistent Context 각각의 선택 기준과 트레이드오프를 설명합니다."
---

## 개요

1부 시리즈를 마무리하면서, 지금까지 사용한 모든 인증 전략을 한 곳에 정리합니다.

플랫폼마다 인증 방식이 다르고, 각 방식은 고유한 장단점과 실패 패턴이 있습니다. 이 글은 새 플랫폼을 추가할 때 어떤 전략을 선택할지 결정하는 **의사결정 가이드**로 활용하세요.

---

## 인증 전략 비교표

| 플랫폼 | 방식 | 난이도 | 세션 지속 | 실패 위험 |
|--------|------|--------|---------|---------|
| 쿠팡 크롤링 | Chrome CDP | 중 | 반영구 | WAF 감지 시 차단 |
| 쿠팡 파트너스 | HMAC-SHA256 | 하 | 만료 없음 | API 한도 초과 |
| 알리익스프레스 | Playwright + storage_state | 중 | 3~7일 | CAPTCHA |
| 티스토리 | Playwright Persistent Context | 상 | 2~3주 | Kakao SSO 추가 인증 |
| 네이버 | CDP 쿠키 → RSA 폴백 | 상 | 수주 | 추가 인증, API 변경 |
| WordPress | App Password (Basic Auth) | 하 | 만료 없음 | 거의 없음 |
| Twitter | OAuth2 Bearer | 하 | 만료 없음 | 월 500개 한도 |
| Threads/Instagram | Meta Graph API OAuth | 중 | 60~90일 | 토큰 만료 |
| Kakao 알림 | OAuth2 + Refresh Token | 중 | 2개월 | Refresh 만료 |
| Google Indexing | Service Account JWT | 하 | 만료 없음 | 일일 200개 한도 |

---

## 전략별 상세 설명

### 1. Chrome CDP (Chrome DevTools Protocol)

**사용 플랫폼**: 쿠팡 크롤링, 네이버 쿠키 추출

**원리**: 로컬 Chrome을 `--remote-debugging-port`로 실행하고, Playwright로 CDP WebSocket에 연결합니다. 실제 Chrome과 동일한 지문을 가지므로 WAF 탐지를 피할 수 있습니다.

```python
# 핵심 코드 패턴
cmd = [CHROME_PATH, f"--remote-debugging-port={PORT}",
       "--disable-blink-features=AutomationControlled"]
proc = subprocess.Popen(cmd)
browser = playwright.chromium.connect_over_cdp(f"http://localhost:{PORT}")
```

**선택 기준**: WAF가 있는 사이트, 로컬 Chrome 프로필의 쿠키를 재사용해야 할 때

**주의사항**: headless 모드는 대부분의 WAF에서 탐지됩니다. 서버 배포 시 Xvfb 필수.

---

### 2. HMAC-SHA256 (쿠팡 파트너스)

**사용 플랫폼**: 쿠팡 파트너스 API

**원리**: 요청 시간 + API 경로를 Secret Key로 서명. 서버가 서명을 검증해 요청 위조를 방지합니다.

```python
import hmac, hashlib, time

def sign(path: str, secret: str) -> str:
    ts = str(int(time.time() * 1000))
    msg = f"{ts}{path}"
    sig = hmac.new(secret.encode(), msg.encode(), hashlib.sha256).hexdigest()
    return f"CEA ... signed-date={ts}, signature={sig}"
```

**선택 기준**: 공식 API가 있고 서명 기반 인증을 사용하는 경우

---

### 3. RSA 암호화 로그인 (네이버)

**사용 플랫폼**: 네이버 로그인 폴백

**원리**: 서버에서 공개키를 발급받고, ID/PW를 공개키로 RSA 암호화해 전송합니다.

```python
import rsa, base64

pub_key = rsa.PublicKey(n, e)
encrypted = rsa.encrypt(message.encode(), pub_key)
encrypted_b64 = base64.b64encode(encrypted).decode()
```

**선택 기준**: ID/PW 로그인이 필요하지만 평문 전송 대신 암호화가 필요한 경우

**주의사항**: 새 IP/기기에서는 추가 인증이 요구될 수 있습니다.

---

### 4. Playwright Persistent Context

**사용 플랫폼**: 티스토리 (Kakao SSO), 알리익스프레스

**원리**: 브라우저 프로필을 디스크에 저장해 세션을 영속화합니다. 한 번 로그인하면 이후 자동으로 로그인 상태가 복원됩니다.

```python
context = playwright.chromium.launch_persistent_context(
    user_data_dir=".sessions/profile",
    headless=False,
)
```

**선택 기준**: SSO(카카오, 구글 등)처럼 복잡한 로그인 흐름이 있는 플랫폼

**주의사항**: headful 모드가 필수인 경우가 많습니다.

---

### 5. storage_state (알리익스프레스)

**사용 플랫폼**: 알리익스프레스

**원리**: Playwright의 `storage_state.json`으로 쿠키 + localStorage를 파일로 저장/복원합니다.

```python
# 저장
context.storage_state(path="storage_state.json")

# 복원
context = browser.new_context(storage_state="storage_state.json")
```

**선택 기준**: Playwright 컨텍스트를 매번 새로 만들지만 로그인 상태를 유지해야 할 때

---

### 6. App Password + Basic Auth (WordPress)

**사용 플랫폼**: WordPress REST API

**원리**: WordPress 5.6+의 앱 비밀번호를 Basic Authentication으로 전송합니다.

```python
session.auth = (username, app_password)
```

**선택 기준**: 공식 REST API가 있고 자체 계정을 관리하는 서비스

**가장 간단하고 안정적인 방식**입니다. 가능하면 이 방식을 선택하세요.

---

### 7. OAuth2 Bearer (Twitter, Meta)

**사용 플랫폼**: Twitter, Threads, Instagram, Pinterest

**원리**: OAuth2 흐름으로 발급받은 Bearer Token을 Authorization 헤더에 포함합니다.

```python
session.headers.update({"Authorization": f"Bearer {bearer_token}"})
```

**선택 기준**: SNS 공식 API가 있는 경우

**주의사항**: 토큰 만료 일정을 반드시 추적하세요.

---

### 8. Service Account JWT (Google)

**사용 플랫폼**: Google Indexing API, Google Search Console

**원리**: 서비스 계정 키(JSON 파일)로 JWT를 생성해 Google API를 인증합니다.

```python
from google.oauth2 import service_account
import googleapiclient.discovery

creds = service_account.Credentials.from_service_account_file(
    "service_account.json",
    scopes=["https://www.googleapis.com/auth/indexing"],
)
```

**선택 기준**: Google 생태계 API (완전 자동화, 토큰 갱신 불필요)

---

## 인증 방식 선택 가이드

```
새 플랫폼 추가 시:

공식 API가 있는가?
├── Yes → OAuth2 / JWT / API Key 사용 (가장 안정적)
└── No → 브라우저 자동화 필요
    │
    WAF/봇 탐지가 강한가?
    ├── Yes → Chrome CDP (실제 Chrome 지문)
    └── No → Playwright
        │
        세션이 복잡한가? (SSO, OAuth 흐름)
        ├── Yes → Persistent Context
        └── No → storage_state 또는 requests.Session
```

---

## 배운 점 / 주의사항

**인증 전략은 단순할수록 좋습니다.** WordPress처럼 App Password가 있으면 브라우저 자동화는 불필요합니다. 복잡한 CDP/Playwright는 공식 API가 없을 때만 쓰세요.

**세션 파일은 모두 `.gitignore`에 추가하세요.** `.sessions/` 전체 디렉토리를 제외합니다.

---

## 1부 시리즈 완료

1부 14편에서 다룬 내용을 마쳤습니다. 2부에서는 실제 구축 과정에서 만난 **구체적인 오류와 해결책**을, 3부에서는 **백링크와 색인 SEO 심화**를 다룹니다.

### 2부 — 트러블슈팅 다이어리

- [[T01] 쿠팡 크롤링 403 Access Denied 완전 해부](/posts/auto-publishing-t01/)
- [[T02] 알리익스프레스 CAPTCHA 슬라이더 자동 감지](/posts/auto-publishing-t02/)
- [[T03] 세션 만료와 자동 재로그인](/posts/auto-publishing-t03/)
- [[T04] Rate Limit 429 대응 — Google·Naver 색인 API](/posts/auto-publishing-t04/)
- [[T05] Kakao SSO 25초 이상 멈춤 — 추가 인증 감지](/posts/auto-publishing-t05/)
- [[T06] 로그인 폼 셀렉터 깨짐 대응](/posts/auto-publishing-t06/)
- [[T07] AliExpress 5xx 오류 복구](/posts/auto-publishing-t07/)
- [[T08] 자동화 탐지 우회 종합](/posts/auto-publishing-t08/)

### 3부 — SEO 심화: 백링크와 색인

- [[S01] 백링크란 무엇인가 — 내부 링크 구조가 SEO에 미치는 영향](/posts/auto-publishing-s01/)
- [[S02] 색인이란 무엇인가 — 구글·네이버가 내 글을 발견하는 원리](/posts/auto-publishing-s02/)
- [[S03] Google Search Console Indexing API 자동화](/posts/auto-publishing-s03/)
- [[S04] 네이버 서치어드바이저 색인 API 자동화](/posts/auto-publishing-s04/)
