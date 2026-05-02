---
title: "[Auto-Publishing] Google Search Console Indexing API 자동화 — 발행 즉시 색인 요청 파이프라인 구축"
date: 2026-04-25 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, SEO, GoogleIndexingAPI, SearchConsole, 서비스계정, JWT인증, 즉시색인]
description: "Google Indexing API를 서비스 계정으로 인증하고, 포스트 발행 즉시 색인 요청을 보내는 파이프라인을 구축하는 방법을 단계별로 설명합니다."
---

## Google Indexing API란

Google Indexing API는 URL을 구글에 직접 제출해 크롤링을 요청하는 API입니다.

보통 구글이 새 포스트를 발견하는 데 수일에서 수주가 걸립니다. Indexing API를 사용하면 발행 후 수시간 내에 색인이 완료됩니다.

**제한사항:**
- 서비스 계정당 하루 200개 URL
- 400개까지는 서비스 계정 2개로 처리 가능

---

## 설정 단계

### 1단계: Google Cloud 프로젝트 생성

1. [Google Cloud Console](https://console.cloud.google.com) 접속
2. 새 프로젝트 생성
3. `APIs & Services` → `Enable APIs` → `Web Search Indexing API` 활성화

### 2단계: 서비스 계정 생성

```bash
# gcloud CLI 사용 시
gcloud iam service-accounts create indexing-bot \
    --display-name="Indexing Bot"

# JSON 키 다운로드
gcloud iam service-accounts keys create .keys/sa_primary.json \
    --iam-account=indexing-bot@PROJECT_ID.iam.gserviceaccount.com
```

또는 Cloud Console에서:
1. `IAM & Admin` → `Service Accounts` → `Create Service Account`
2. 생성 후 `Keys` → `Add Key` → `JSON` 다운로드

### 3단계: Search Console에 서비스 계정 소유자 추가

이 단계가 빠지면 403 오류가 발생합니다.

1. [Google Search Console](https://search.google.com/search-console) 접속
2. 해당 속성(사이트) 선택
3. `설정` → `사용자 및 권한` → `사용자 추가`
4. 서비스 계정 이메일 추가 (형식: `이름@PROJECT_ID.iam.gserviceaccount.com`)
5. 권한: `소유자`

---

## 구현 코드

```python
# common/indexing_google.py

from google.oauth2 import service_account
from googleapiclient.discovery import build
from pathlib import Path
import json
import time

SCOPES = ["https://www.googleapis.com/auth/indexing"]
SA_KEY_PATH = Path(".keys/sa_primary.json")

def _build_service(sa_path: Path):
    """서비스 계정으로 Indexing API 서비스 객체 생성"""
    creds = service_account.Credentials.from_service_account_file(
        str(sa_path),
        scopes=SCOPES,
    )
    return build("indexing", "v3", credentials=creds)

def index_google(urls: list[str], sa_path: Path = SA_KEY_PATH) -> dict[str, str]:
    """
    URL 목록을 Google에 색인 요청
    반환: {url: "ok" | "limit" | "error" | "exists"}
    """
    results = {}
    
    try:
        service = _build_service(sa_path)
    except Exception as e:
        log(f"[Google 색인] 서비스 계정 인증 실패: {e}", "error")
        return {url: "error" for url in urls}
    
    for idx, url in enumerate(urls):
        # 일일 한도 체크 (200개)
        if idx >= 200:
            log(f"[Google 색인] 일일 한도(200개) 초과 — 나머지 {len(urls)-idx}개 skip", "warn")
            for remaining_url in urls[idx:]:
                results[remaining_url] = "limit"
            break
        
        try:
            resp = service.urlNotifications().publish(
                body={"url": url, "type": "URL_UPDATED"}
            ).execute()
            
            results[url] = "ok"
            log(f"[Google 색인] 요청 완료: {url}", "info")
            
        except Exception as e:
            error_str = str(e)
            
            if "429" in error_str or "Quota exceeded" in error_str:
                log(f"[Google 색인] 한도 초과 (429): {url}", "warn")
                results[url] = "limit"
                # 나머지도 limit 처리
                for remaining in urls[idx+1:]:
                    results[remaining] = "limit"
                break
            
            elif "403" in error_str:
                log(f"[Google 색인] 권한 오류 — Search Console 소유자 설정 확인: {url}", "error")
                results[url] = "error"
            
            else:
                log(f"[Google 색인] 오류: {url} — {error_str[:100]}", "warn")
                results[url] = "error"
        
        # 요청 간 짧은 딜레이 (API 안정성)
        time.sleep(0.5)
    
    ok_count = sum(1 for s in results.values() if s == "ok")
    log(f"[Google 색인] 완료: {ok_count}/{len(urls)}개 성공", "ok")
    
    return results
```

---

## 다중 서비스 계정으로 한도 확장

```python
# common/indexing_google.py

GOOGLE_SA_KEYS = [
    Path(".keys/sa_primary.json"),
    Path(".keys/sa_backup.json"),    # 두 번째 서비스 계정
]

def index_google_with_fallback(urls: list[str]) -> dict[str, str]:
    """여러 서비스 계정을 순차적으로 사용해 한도 분산"""
    
    all_results = {}
    remaining = list(urls)
    
    for sa_path in GOOGLE_SA_KEYS:
        if not remaining:
            break
        
        if not sa_path.exists():
            log(f"서비스 계정 키 없음: {sa_path}", "warn")
            continue
        
        batch_results = index_google(remaining, sa_path=sa_path)
        all_results.update(batch_results)
        
        # 한도 초과된 URL만 다음 계정으로 넘김
        remaining = [
            url for url, status in batch_results.items()
            if status == "limit"
        ]
        
        if remaining:
            log(f"SA 전환: {len(remaining)}개 다음 계정으로", "info")
    
    return all_results
```

---

## 파이프라인 통합

```python
# pipeline.py

def publish_and_index(post: dict):
    """포스트 발행 + 즉시 색인 요청"""
    
    # 1. GitHub Pages에 발행
    page_url = write_jekyll_post(post)
    
    if not page_url:
        log("발행 실패 — 색인 요청 스킵", "error")
        return
    
    # 2. git push 완료 대기 (GitHub Actions 빌드 시간)
    log("GitHub Actions 빌드 대기 중...", "info")
    time.sleep(120)  # 평균 2분
    
    # 3. 페이지 접근 가능 확인
    if not wait_for_page_live(page_url, max_wait=300):
        log(f"페이지 접근 불가 — 색인 요청 스킵: {page_url}", "warn")
        return
    
    # 4. 구글 색인 요청
    google_results = index_google_with_fallback([page_url])
    
    # 5. 네이버 색인 요청 (다음 포스트 참고)
    naver_results = index_naver([page_url])
    
    # 6. 결과 알림
    notify_telegram(
        f"✅ 발행 완료: {post['title']}\n"
        f"구글: {google_results.get(page_url, 'unknown')}\n"
        f"네이버: {naver_results.get(page_url, 'unknown')}"
    )

def wait_for_page_live(url: str, max_wait: int = 300) -> bool:
    """페이지가 실제로 접근 가능해질 때까지 대기"""
    import requests
    
    deadline = time.time() + max_wait
    
    while time.time() < deadline:
        try:
            resp = requests.head(url, timeout=10)
            if resp.status_code == 200:
                log(f"페이지 활성화 확인: {url}", "ok")
                return True
        except Exception:
            pass
        
        time.sleep(15)
    
    return False
```

---

## 색인 요청 결과 추적

```python
# common/indexing_tracker.py
from pathlib import Path
import json
from datetime import datetime

TRACKING_FILE = Path(".data/indexing_history.jsonl")

def record_indexing(url: str, platform: str, status: str):
    """색인 요청 이력 기록"""
    TRACKING_FILE.parent.mkdir(exist_ok=True)
    
    entry = {
        "timestamp": datetime.now().isoformat(),
        "url": url,
        "platform": platform,
        "status": status,
    }
    
    with open(TRACKING_FILE, "a") as f:
        f.write(json.dumps(entry, ensure_ascii=False) + "\n")

def get_daily_stats() -> dict:
    """오늘 색인 요청 통계"""
    today = datetime.now().strftime("%Y-%m-%d")
    
    stats = {"google": {"ok": 0, "error": 0, "limit": 0},
             "naver": {"ok": 0, "error": 0, "limit": 0}}
    
    if not TRACKING_FILE.exists():
        return stats
    
    for line in TRACKING_FILE.read_text().splitlines():
        entry = json.loads(line)
        if entry["timestamp"].startswith(today):
            platform = entry["platform"]
            status = entry["status"]
            if platform in stats and status in stats[platform]:
                stats[platform][status] += 1
    
    return stats
```

---

## 자주 발생하는 오류

**403 Forbidden**

```
서비스 계정을 Search Console에 소유자로 추가했는지 확인하세요.
도메인 속성 vs URL 속성을 구분하세요.
도메인 속성은 DNS TXT 레코드 소유 확인이 필요합니다.
```

**404 Not Found**

```
URL이 실제로 존재하는지 확인하세요.
GitHub Actions 빌드가 완료되기 전에 요청했을 가능성이 있습니다.
```

**429 Too Many Requests**

```
하루 200개 한도를 초과했습니다.
우선순위 큐([T04] 참고)를 사용해 중요한 URL을 먼저 처리하세요.
```

---

## 관련 포스팅

- [[T04] Rate Limit 429 대응 — Google·Naver 색인 API 일일 한도 초과 처리](/posts/auto-publishing-t04/)
- [[S02] 색인이란 무엇인가 — 구글·네이버가 내 글을 발견하는 원리](/posts/auto-publishing-s02/)
- [[S04] 네이버 서치어드바이저 색인 API 자동화](/posts/auto-publishing-s04/)
