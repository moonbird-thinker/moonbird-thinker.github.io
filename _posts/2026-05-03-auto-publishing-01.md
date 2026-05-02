---
title: "[Auto-Publishing] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계"
date: 2026-05-03 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, AI, 파이썬, 블로그자동화, 패시브인컴, 아키텍처]
description: "AI 기반 자동 발행 파이프라인의 전체 구조를 공개합니다. 키워드 수집부터 콘텐츠 생성, 멀티 플랫폼 발행까지 하나의 파이프라인으로 연결한 설계 원칙과 핵심 패턴을 설명합니다."
---

## 개요

이 시리즈는 **AI를 활용한 자동 발행 시스템**을 처음부터 끝까지 직접 구축하면서 겪은 경험을 기록한 것입니다.

단순히 "이런 코드를 짰다"가 아니라 **왜 이 구조를 선택했는지**, **어떤 실패를 겪었는지**, **어떻게 해결했는지**를 있는 그대로 적겠습니다.

이번 1편에서는 시스템 전체 아키텍처를 먼저 조감합니다. 이후 편에서 각 모듈을 하나씩 깊이 파고듭니다.

---

## 왜 이 시스템이 필요했나

블로그 수익화의 핵심은 **트래픽**입니다. 트래픽을 만들려면 콘텐츠가 많아야 하고, 콘텐츠를 많이 만들려면 시간이 필요합니다. 풀스택 개발자로 20년을 일하면서 가장 부족한 것은 늘 시간이었습니다.

그래서 목표는 단순했습니다: **내가 자는 동안에도 글이 올라가는 시스템**.

- 쿠팡·알리익스프레스 제휴 상품 소개글을 AI가 자동 생성
- WordPress·티스토리·네이버 블로그·GitHub Pages에 동시 발행
- Twitter·Threads·Instagram·Pinterest에 SNS 홍보까지 자동화

이걸 하나의 Python 파이프라인으로 묶는 것이 목표였습니다.

---

## 전체 아키텍처

시스템은 크게 4개 레이어로 구성됩니다.

```
[1] 소스 (Sources)
    ├── 쿠팡 상품 크롤링 (Chrome CDP)
    ├── 알리익스프레스 상품 크롤링 (Playwright)
    ├── 뉴스픽 API
    └── 정책브리핑 RSS

[2] 콘텐츠 생성 (AI)
    ├── Claude CLI
    └── Gemini API

[3] 발행 (Publishers)
    ├── WordPress REST API
    ├── 티스토리 (Playwright)
    ├── 네이버 블로그·카페 (CDP + RSA)
    ├── GitHub Pages (Jekyll + git)
    └── SNS (Twitter, Threads, Instagram, Pinterest)

[4] 인프라 (Common)
    ├── 세션 관리 (SessionManager)
    ├── 브라우저 프로필 (BrowserProfile)
    ├── 스케줄러 (Registry 패턴)
    └── 알림 (텔레그램 + 카카오톡)
```

---

## 핵심 설계 원칙: Registry 패턴

가장 중요한 설계 결정은 **파이프라인 자동 발견(auto-discovery)** 입니다.

새 파이프라인을 추가할 때마다 스케줄러에 수동으로 등록하는 것은 번거롭고 실수가 생깁니다. 대신 `pkgutil.iter_modules()`를 사용해 `pipelines/` 디렉토리 안의 파일을 자동으로 발견하고 등록합니다.

```python
# scheduler.py
import pkgutil
import importlib
import pipelines

def discover_pipelines():
    """pipelines/ 디렉토리 안의 모든 파이프라인을 자동 등록"""
    found = {}
    for importer, modname, ispkg in pkgutil.iter_modules(pipelines.__path__):
        module = importlib.import_module(f"pipelines.{modname}")
        if hasattr(module, "PIPELINE"):
            pipeline = module.PIPELINE
            found[pipeline.name] = pipeline
    return found
```

이 패턴 덕분에 `pipelines/coupang_to_wordpress.py` 파일을 추가하기만 하면 스케줄러가 자동으로 인식합니다. 12편에서 자세히 다룹니다.

---

## 세션 관리 구조

각 플랫폼은 인증 방식이 다릅니다. 이것을 통일된 인터페이스로 추상화했습니다.

```python
# common/session.py
class SessionManager:
    """requests.Session 기반 쿠키 영속화"""
    
    SESSIONS_DIR = Path(".sessions")
    
    def __init__(self, name: str):
        self.name = name
        self._session = requests.Session()
        self._session.headers.update({
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                         "AppleWebKit/537.36 Chrome/120.0.0.0 Safari/537.36"
        })
        self._load()
    
    def _load(self):
        path = self.SESSIONS_DIR / f"{self.name}.pkl"
        if path.exists():
            with open(path, "rb") as f:
                self._session.cookies.update(pickle.load(f))
    
    def save(self):
        self.SESSIONS_DIR.mkdir(exist_ok=True)
        path = self.SESSIONS_DIR / f"{self.name}.pkl"
        with open(path, "wb") as f:
            pickle.dump(self._session.cookies, f)
```

브라우저가 필요한 플랫폼(티스토리, 알리익스프레스)은 `BrowserProfile`로 Playwright Persistent Context를 관리합니다. 세션이 살아있는 한 재로그인이 불필요합니다.

---

## 파이프라인 기본 구조

모든 파이프라인은 동일한 인터페이스를 따릅니다.

```python
# pipelines/coupang_to_wordpress.py
from dataclasses import dataclass
from typing import Callable

@dataclass
class Pipeline:
    name: str
    description: str
    run: Callable[[], None]
    cron: str  # "0 9 * * *" 형태

def _run():
    products = CoupangSource().fetch(keyword="무선 이어폰", limit=5)
    for product in products:
        content = GeminiWriter().write(product)
        WordPressPublisher().publish(content)

PIPELINE = Pipeline(
    name="coupang_to_wordpress",
    description="쿠팡 상품 → AI 글쓰기 → WordPress 발행",
    run=_run,
    cron="0 9 * * *",  # 매일 오전 9시
)
```

---

## 실패 사례: 모놀리식으로 시작했다가 전부 다시 짰다

처음에는 하나의 거대한 스크립트로 시작했습니다. 1,500줄짜리 파일 하나에 쿠팡 크롤링, 글쓰기, WordPress 발행이 전부 들어있었습니다.

문제는 바로 나왔습니다:
- 쿠팡 파트를 고치면 WordPress 파트가 깨짐
- 새 플랫폼을 추가할 때마다 기존 코드를 건드려야 함
- 에러가 어느 단계에서 났는지 추적이 어려움

**3주 후 전부 다시 짰습니다.** sources / publishers / common 레이어로 분리하고, 각 파이프라인은 독립 파일로 만들었습니다. 이후 새 플랫폼 추가가 30분 이내로 가능해졌습니다.

---

## 배운 점 / 주의사항

**관심사 분리가 핵심입니다.** 소스(크롤링)와 발행(publishing)을 섞으면 유지보수가 불가능해집니다.

**세션 파일은 `.gitignore`에 반드시 추가하세요.** `.sessions/` 디렉토리에는 로그인 쿠키가 들어있습니다. 실수로 커밋하면 계정이 탈취될 수 있습니다.

**플랫폼 하나씩 검증하세요.** 처음부터 6개 플랫폼을 동시에 붙이려다 어느 플랫폼에서 문제가 생기는지 파악이 안 됩니다.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. **[현재 글] [Auto-Publishing] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계**
2. [[02] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기](/posts/auto-publishing-02/)
3. [[03] 쿠팡 크롤링 — Access Denied 뚫기: Chrome CDP로 WAF 우회](/posts/auto-publishing-03/)
4. [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
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
