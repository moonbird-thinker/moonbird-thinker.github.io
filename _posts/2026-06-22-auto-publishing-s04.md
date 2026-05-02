---
title: "[Auto-Publishing] 네이버 서치어드바이저 색인 API 자동화 — 일일 50개 한도 안에서 최대 효율 뽑기"
date: 2026-06-22 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, SEO, 네이버서치어드바이저, 색인API, 네이버색인, 검색노출최적화, 사이트맵제출]
description: "네이버 서치어드바이저 URL 수집 요청 API를 인증하고, 일일 50개 한도 안에서 우선순위 기반으로 최대 효율을 뽑는 자동화 구현을 설명합니다."
---

## 네이버 서치어드바이저 색인 API

네이버 서치어드바이저는 URL을 직접 제출해 네이버 검색 색인을 요청하는 API를 제공합니다.

**특징:**
- 일일 최대 50개 URL
- 인증 방식: API 키 + HMAC 서명
- 응답 코드로 결과 즉시 확인 가능
- 구글 Indexing API보다 한도가 훨씬 적음

50개 한도가 매우 빡빡합니다. 우선순위 관리가 핵심입니다.

---

## 설정 단계

### 1단계: 사이트 등록

1. [네이버 서치어드바이저](https://searchadvisor.naver.com) 접속
2. 웹마스터 도구 → 사이트 추가
3. `https://moonbird-thinker.github.io` 입력
4. 소유 확인: HTML 파일 업로드 또는 메타태그 삽입

### 2단계: API 키 발급

1. 서치어드바이저 → 계정 설정 → API 키
2. API 키와 시크릿 키 확인

### 3단계: 사이트맵 제출

1. 요청 → 사이트맵 제출
2. `https://moonbird-thinker.github.io/sitemap.xml` 입력

사이트맵 제출은 한 번만 해도 됩니다. 이후 sitemap.xml이 업데이트될 때마다 자동으로 처리됩니다.

---

## 인증 방식

네이버 서치어드바이저 API는 HMAC-SHA256 서명을 사용합니다.

```python
# common/indexing_naver.py
import hmac
import hashlib
import base64
import time
import requests
from pathlib import Path
import json

# 설정값 (환경변수로 관리)
NAVER_API_URL = "https://apis.naver.com/searchadvisor/crawl/siteSubmit"
NAVER_CUSTOMER_ID = "YOUR_CUSTOMER_ID"   # API 키
NAVER_ACCESS_LICENSE = "YOUR_ACCESS_LICENSE"
NAVER_SECRET_KEY = "YOUR_SECRET_KEY"
SITE_URL = "https://moonbird-thinker.github.io"
DAILY_LIMIT = 50
REQUEST_INTERVAL = 10  # 요청 사이 10초 대기

def _build_auth_header(timestamp: str) -> dict:
    """HMAC-SHA256 서명 헤더 생성"""
    message = f"{NAVER_ACCESS_LICENSE}\n{timestamp}"
    
    signature = base64.b64encode(
        hmac.new(
            NAVER_SECRET_KEY.encode("utf-8"),
            message.encode("utf-8"),
            hashlib.sha256
        ).digest()
    ).decode("utf-8")
    
    return {
        "X-Naver-Client-Id": NAVER_CUSTOMER_ID,
        "X-Naver-Access-License": NAVER_ACCESS_LICENSE,
        "X-Naver-Timestamp": timestamp,
        "X-Naver-Signature": signature,
        "Content-Type": "application/json",
    }
```

---

## 핵심 구현

```python
# common/indexing_naver.py

def index_naver(urls: list[str]) -> dict[str, str]:
    """
    URL 목록을 네이버에 색인 요청
    반환: {url: "ok" | "limit" | "error" | "duplicate"}
    """
    results = {}
    
    for idx, url in enumerate(urls):
        # 일일 한도 체크 (50개)
        if idx >= DAILY_LIMIT:
            log(f"[Naver 색인] 일일 한도({DAILY_LIMIT}개) 초과 — 나머지 skip", "warn")
            for remaining in urls[idx:]:
                results[remaining] = "limit"
            break
        
        timestamp = str(int(time.time() * 1000))
        headers = _build_auth_header(timestamp)
        
        body = {
            "siteUrl": SITE_URL,
            "urlList": [url],
        }
        
        try:
            resp = requests.post(
                NAVER_API_URL,
                headers=headers,
                json=body,
                timeout=15,
            )
            
            data = resp.json()
            message = data.get("message", "")
            
            if message == "SUCCESS":
                results[url] = "ok"
                log(f"[Naver 색인] 성공: {url}", "info")
            
            elif message == "FAIL_MAX_DOCUMENT_COUNT":
                log(f"[Naver 색인] 일일 한도 초과 (API 응답) — 즉시 중단", "warn")
                results[url] = "limit"
                for remaining in urls[idx+1:]:
                    results[remaining] = "limit"
                break
            
            elif message == "FAIL_DUPLICATE_DOCUMENT":
                log(f"[Naver 색인] 이미 제출된 URL: {url}", "info")
                results[url] = "duplicate"
            
            elif message == "FAIL_INVALID_URL":
                log(f"[Naver 색인] 유효하지 않은 URL: {url}", "warn")
                results[url] = "error"
            
            else:
                log(f"[Naver 색인] 알 수 없는 응답: {message}", "warn")
                results[url] = "error"
        
        except Exception as e:
            log(f"[Naver 색인] 요청 실패: {url} — {e}", "error")
            results[url] = "error"
        
        # 요청 사이 10초 대기 (Naver 권고 사항)
        if idx < len(urls) - 1:
            time.sleep(REQUEST_INTERVAL)
    
    ok_count = sum(1 for s in results.values() if s == "ok")
    log(f"[Naver 색인] 완료: {ok_count}/{len(urls)}개 성공", "ok")
    
    return results
```

---

## 우선순위 큐로 50개 최대 활용

50개 한도는 하루 발행량이 많을 때 금방 차버립니다. 중요한 URL을 먼저 처리해야 합니다.

```python
# common/indexing_queue.py — T04 포스트의 확장판

from dataclasses import dataclass, field
from enum import IntEnum
import heapq
from datetime import datetime, timedelta

class Priority(IntEnum):
    TODAY = 1       # 당일 발행
    RECENT = 2      # 최근 3일
    UPDATED = 3     # 내용 업데이트
    OLD = 4         # 오래된 재색인 요청

@dataclass(order=True)
class IndexTask:
    priority: int
    published_at: str = field(compare=False)
    url: str = field(compare=False)
    platform: str = field(compare=False, default="naver")

class NaverIndexQueue:
    """네이버 색인 전용 우선순위 큐 — 50개 한도 최적화"""
    
    QUEUE_FILE = Path(".data/naver_index_queue.json")
    
    def __init__(self):
        self._heap = []
        self._load()
    
    def enqueue(self, url: str, published_at: str | None = None):
        """새 URL 추가 — 나이에 따라 우선순위 자동 결정"""
        if published_at is None:
            published_at = datetime.now().isoformat()
        
        pub_dt = datetime.fromisoformat(published_at)
        age = datetime.now() - pub_dt
        
        if age < timedelta(days=1):
            priority = Priority.TODAY
        elif age < timedelta(days=3):
            priority = Priority.RECENT
        elif age < timedelta(days=14):
            priority = Priority.UPDATED
        else:
            priority = Priority.OLD
        
        task = IndexTask(priority=priority.value, published_at=published_at, url=url)
        heapq.heappush(self._heap, task)
        self._save()
    
    def pop_daily_batch(self) -> list[IndexTask]:
        """오늘 처리할 최대 50개 추출"""
        batch = []
        for _ in range(min(DAILY_LIMIT, len(self._heap))):
            batch.append(heapq.heappop(self._heap))
        self._save()
        return batch
    
    def requeue_failed(self, urls: list[str]):
        """실패/한도 초과 URL 재삽입 (우선순위 유지)"""
        for url in urls:
            self.enqueue(url)
    
    def __len__(self):
        return len(self._heap)
    
    def _save(self):
        self.QUEUE_FILE.parent.mkdir(exist_ok=True)
        data = [
            {"priority": t.priority, "url": t.url, "published_at": t.published_at}
            for t in self._heap
        ]
        self.QUEUE_FILE.write_text(json.dumps(data, ensure_ascii=False, indent=2))
    
    def _load(self):
        if not self.QUEUE_FILE.exists():
            return
        data = json.loads(self.QUEUE_FILE.read_text())
        self._heap = [
            IndexTask(d["priority"], d["published_at"], d["url"])
            for d in data
        ]
        heapq.heapify(self._heap)
```

---

## 매일 자정 자동 실행

```python
# scheduler.py

import schedule

def run_naver_indexing():
    """매일 자정 직후 네이버 색인 처리"""
    queue = NaverIndexQueue()
    
    if len(queue) == 0:
        log("[Naver 색인] 대기 중인 URL 없음", "info")
        return
    
    batch = queue.pop_daily_batch()
    urls = [task.url for task in batch]
    
    log(f"[Naver 색인] 오늘 처리: {len(urls)}개 (대기 중: {len(queue)}개)", "info")
    
    results = index_naver(urls)
    
    # 실패/한도 초과 URL 재큐잉
    failed = [url for url, status in results.items() if status in ("error", "limit")]
    if failed:
        queue.requeue_failed(failed)
        log(f"[Naver 색인] 재큐잉: {len(failed)}개", "warn")
    
    ok_count = sum(1 for s in results.values() if s == "ok")
    
    notify_telegram(
        f"📊 네이버 색인 일일 결과\n"
        f"성공: {ok_count}/{len(urls)}개\n"
        f"대기 중: {len(queue)}개\n"
    )

# 매일 00:05에 실행 (자정 직후)
schedule.every().day.at("00:05").do(run_naver_indexing)
```

---

## 응답 코드 전체 목록

| 코드 | 의미 | 대응 |
|------|------|------|
| `SUCCESS` | 색인 요청 성공 | 완료 |
| `FAIL_MAX_DOCUMENT_COUNT` | 일일 한도 초과 | 내일 재시도 |
| `FAIL_DUPLICATE_DOCUMENT` | 이미 제출된 URL | 무시 |
| `FAIL_INVALID_URL` | 유효하지 않은 URL | URL 확인 후 수정 |
| `FAIL_INVALID_SITE` | 등록되지 않은 사이트 | 서치어드바이저 사이트 등록 확인 |
| `FAIL_NOT_EXIST_DOCUMENT` | 존재하지 않는 URL | 페이지 실제 존재 여부 확인 |
| `FAIL_AUTH` | 인증 실패 | API 키·서명 확인 |

---

## 구글 vs 네이버 색인 비교

| 항목 | 구글 Indexing API | 네이버 서치어드바이저 |
|------|:---:|:---:|
| 일일 한도 | 200개 | 50개 |
| 인증 방식 | 서비스 계정 JWT | API 키 + HMAC-SHA256 |
| 요청 간격 | 0.5초 | 10초 (권고) |
| 결과 확인 | 즉시 응답 | 즉시 응답 |
| 실제 색인 소요 | 수시간 | 수일 |
| 한도 초과 오류 | HTTP 429 | `FAIL_MAX_DOCUMENT_COUNT` |

네이버는 한도가 적고 실제 색인까지 더 오래 걸립니다. 핵심 포스트 위주로 전략적으로 사용하세요.

---

## 배운 점 / 주의사항

**중복 제출은 카운트에서 제외됩니다.** `FAIL_DUPLICATE_DOCUMENT`는 50개 한도에서 차감되지 않습니다. 단, 이미 색인된 URL을 반복 제출해도 색인 순위에 도움이 되지 않습니다.

**오래된 포스트 재색인은 비효율적입니다.** 50개 한도에서 오래된 URL에 낭비하지 말고, 당일/최근 발행 포스트를 우선시하세요.

**사이트맵이 있으면 결국 색인됩니다.** API 한도를 초과해도 sitemap.xml이 제출되어 있으면 네이버 Yeti가 주기적으로 크롤링합니다. API는 속도를 높이는 도구이지 유일한 방법이 아닙니다.

---

## 관련 포스팅

- [[S02] 색인이란 무엇인가 — 구글·네이버가 내 글을 발견하는 원리](/posts/auto-publishing-s02/)
- [[S03] Google Search Console Indexing API 자동화](/posts/auto-publishing-s03/)
- [[T04] Rate Limit 429 대응 — Google·Naver 색인 API 일일 한도](/posts/auto-publishing-t04/)
