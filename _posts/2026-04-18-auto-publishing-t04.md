---
title: "[Auto-Publishing] Rate Limit 429 대응 — Google·Naver 색인 API 일일 한도 초과 처리 전략"
date: 2026-04-18 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 트러블슈팅, 429오류, RateLimit, GoogleIndexingAPI, 네이버서치어드바이저]
description: "Google Indexing API 일일 200개, 네이버 서치어드바이저 일일 50개 한도를 관리하며 색인 효율을 극대화하는 전략을 설명합니다. 429 응답 처리와 URL 우선순위 큐 설계까지 다룹니다."
---

## 문제 상황

자동 발행이 잘 돌아가다가 어느 날부터 색인 요청이 전부 실패하기 시작했습니다.

```
[WARN] [Google 색인] 일일 한도(200개) 초과 — 중단
[WARN] [Naver 색인] FAIL_MAX_DOCUMENT_COUNT — 중단
```

Google Indexing API는 하루 200개, 네이버 서치어드바이저는 하루 50개가 한도입니다. 자동 발행이 늘어나면서 이 한도를 초과하기 시작했습니다.

---

## 원인 분석

### Google Indexing API 한도

Google Indexing API는 서비스 계정당 하루 200개 URL을 처리합니다.

```python
# common/indexing_google.py 실제 응답 처리
for idx, url in enumerate(urls):
    resp = indexing_service.urlNotifications().publish(
        body={"url": url, "type": "URL_UPDATED"}
    ).execute()
    
    status = str(resp.get("urlNotificationMetadata", {}).get("latestUpdate", {}).get("type", ""))
    
    if "429" in str(resp) or idx >= 200:
        log(f"[Google 색인] 일일 한도(200개) 초과 — 중단", "warn")
        # 나머지 URL은 "limit" 상태로 마킹
        for remaining_url in list(urls)[idx:]:
            results[remaining_url] = "limit"
        break
```

### 네이버 서치어드바이저 한도

네이버는 하루 50개 제한이 있고, `FAIL_MAX_DOCUMENT_COUNT` 오류 코드로 알려줍니다.

```python
# common/indexing_naver.py
DAILY_LIMIT = 50
REQUEST_INTERVAL = 10  # 요청 사이 10초 대기

for idx, url in enumerate(urls):
    if idx >= DAILY_LIMIT:
        log(f"[Naver 색인] 일일 한도({DAILY_LIMIT}개) 초과 — 중단", "warn")
        break
    
    resp = session.post(
        NAVER_API_URL,
        json={"siteUrl": SITE_URL, "urlList": [url]},
        timeout=15,
    )
    
    data = resp.json()
    message = data.get("message", "")
    
    if message == "FAIL_MAX_DOCUMENT_COUNT":
        log("[Naver 색인] 일일 한도 초과 — 즉시 중단", "warn")
        for remaining in urls[idx:]:
            results[remaining] = "limit"
        break
    
    time.sleep(REQUEST_INTERVAL)
```

---

## URL 우선순위 큐 설계

하루 한도가 정해져 있다면, **중요한 URL을 먼저 처리**해야 합니다.

```python
# common/indexing_queue.py
from dataclasses import dataclass, field
from enum import IntEnum
import heapq
import json
from pathlib import Path

class Priority(IntEnum):
    URGENT = 1      # 당일 발행 포스트
    HIGH = 2        # 최근 3일 포스트
    MEDIUM = 3      # 업데이트된 포스트
    LOW = 4         # 재색인 요청

@dataclass(order=True)
class IndexingTask:
    priority: int
    url: str = field(compare=False)
    published_at: str = field(compare=False)

class IndexingQueue:
    """영속적 우선순위 큐 — 재시작 후에도 유지"""
    
    QUEUE_FILE = Path(".data/indexing_queue.json")
    
    def __init__(self):
        self._heap = []
        self._load()
    
    def add(self, url: str, priority: Priority, published_at: str):
        task = IndexingTask(priority=priority.value, url=url, published_at=published_at)
        heapq.heappush(self._heap, task)
        self._save()
    
    def pop_batch(self, n: int) -> list[IndexingTask]:
        """상위 n개 URL 추출"""
        batch = []
        for _ in range(min(n, len(self._heap))):
            batch.append(heapq.heappop(self._heap))
        self._save()
        return batch
    
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
            IndexingTask(d["priority"], d["url"], d["published_at"])
            for d in data
        ]
        heapq.heapify(self._heap)
```

### 우선순위 기반 색인 실행

```python
# common/indexing_manager.py
from datetime import datetime, timedelta

def run_daily_indexing():
    """매일 자정 직후 실행 — 한도 내에서 최대 효율"""
    queue = IndexingQueue()
    
    GOOGLE_DAILY_LIMIT = 200
    NAVER_DAILY_LIMIT = 50
    
    # 구글 색인: 상위 200개
    google_batch = queue.pop_batch(GOOGLE_DAILY_LIMIT)
    google_urls = [t.url for t in google_batch]
    google_results = index_google(google_urls)
    
    # 네이버 색인: 상위 50개
    naver_batch = queue.pop_batch(NAVER_DAILY_LIMIT)
    naver_urls = [t.url for t in naver_batch]
    naver_results = index_naver(naver_urls)
    
    # 한도 초과로 실패한 URL 재큐잉
    for url, status in {**google_results, **naver_results}.items():
        if status == "limit":
            queue.add(url, Priority.HIGH, datetime.now().isoformat())
    
    # 결과 요약 알림
    success = sum(1 for s in google_results.values() if s == "ok")
    notify_telegram(
        f"📊 오늘의 색인 처리 결과\n"
        f"구글: {success}/{len(google_urls)}개 성공\n"
        f"네이버: {sum(1 for s in naver_results.values() if s == 'ok')}/{len(naver_urls)}개 성공\n"
        f"대기 중: {len(queue)}개"
    )
```

---

## 서비스 계정 복수 운영

Google은 서비스 계정당 200개 한도입니다. 서비스 계정을 여러 개 만들면 한도를 늘릴 수 있습니다.

```python
# common/indexing_google.py
GOOGLE_SA_KEYS = [
    Path(".keys/sa_primary.json"),
    Path(".keys/sa_backup.json"),
]

def index_google_with_fallback(urls: list[str]) -> dict:
    """여러 서비스 계정으로 순차 시도"""
    results = {}
    remaining = list(urls)
    
    for sa_path in GOOGLE_SA_KEYS:
        if not remaining:
            break
        
        batch_results = _index_google_batch(remaining, sa_path)
        results.update(batch_results)
        
        # 한도 초과 URL만 다음 SA로
        remaining = [
            url for url, status in batch_results.items()
            if status == "limit"
        ]
    
    return results
```

---

## 재발 방지: 한도 사용량 모니터링

```python
# 일일 사용량 추적
USAGE_FILE = Path(".data/daily_usage.json")

def track_usage(platform: str, count: int):
    today = datetime.now().strftime("%Y-%m-%d")
    usage = json.loads(USAGE_FILE.read_text()) if USAGE_FILE.exists() else {}
    
    if today not in usage:
        usage[today] = {}
    usage[today][platform] = usage[today].get(platform, 0) + count
    
    USAGE_FILE.write_text(json.dumps(usage))
    
    # 한도 80% 도달 시 경고
    limits = {"google": 200, "naver": 50}
    if platform in limits:
        used = usage[today][platform]
        limit = limits[platform]
        if used >= limit * 0.8:
            notify_telegram(f"⚠️ {platform} 색인 한도 {used}/{limit}개 사용 중")
```

---

## 관련 포스팅

- [[S02] 색인이란 무엇인가 — 구글·네이버가 내 글을 발견하는 원리](/posts/auto-publishing-s02/)
- [[S03] Google Search Console Indexing API 자동화](/posts/auto-publishing-s03/)
- [[S04] 네이버 서치어드바이저 색인 API 자동화](/posts/auto-publishing-s04/)
