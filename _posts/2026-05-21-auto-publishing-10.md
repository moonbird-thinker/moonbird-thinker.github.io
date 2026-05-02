---
title: "[Auto-Publishing] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest"
date: 2026-05-21 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, SNS자동화, Threads, Twitter, Pinterest]
description: "블로그 발행 후 SNS 4곳에 자동으로 홍보 포스팅하는 파이프라인을 구현했습니다. Threads Graph API, Twitter v2 API, Instagram Graph API, Pinterest API 연동 방법을 설명합니다."
---

## 개요

블로그 글을 발행한 뒤 SNS에 홍보하지 않으면 초기 트래픽이 없어 구글 색인도 느려집니다.

이번 편에서는 블로그 발행 직후 **Twitter, Threads, Instagram, Pinterest에 자동으로 홍보 포스팅**하는 파이프라인을 설명합니다.

SNS 유입은 직접 수익보다 **색인 신호**로서 더 중요합니다. 사람들이 공유하고 클릭하면 구글이 이 페이지를 빠르게 색인합니다.

---

## SNS별 API 특성 비교

| 플랫폼 | API 방식 | 인증 | 무료 한도 |
|--------|---------|------|---------|
| Twitter | v2 OAuth2 | Bearer Token | 월 500 tweet |
| Threads | Graph API | Meta OAuth | 250 post/시간 |
| Instagram | Graph API | Meta OAuth | 25 post/시간 |
| Pinterest | v5 REST | OAuth2 | 제한 없음 |

Twitter가 가장 제한이 강합니다. 무료 플랜은 월 500개이므로 하루 16개가 한계입니다.

---

## 핵심 구현

### Twitter v2 API

```python
# publishers/sns/twitter.py
import requests

class TwitterPublisher:
    API_BASE = "https://api.twitter.com/2"
    
    def __init__(self, bearer_token: str):
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {bearer_token}",
            "Content-Type": "application/json",
        })
    
    def post(self, blog_url: str, title: str, tags: list[str]) -> str:
        """블로그 URL + 제목 + 해시태그로 트윗"""
        # Twitter 280자 제한
        hashtags = " ".join(f"#{t}" for t in tags[:3])
        text = f"{title}\n\n{blog_url}\n\n{hashtags}"
        text = text[:280]
        
        resp = self.session.post(
            f"{self.API_BASE}/tweets",
            json={"text": text},
        )
        resp.raise_for_status()
        tweet_id = resp.json()["data"]["id"]
        log(f"Twitter 게시 완료: {tweet_id}", "ok")
        return tweet_id
```

### Threads Graph API

Threads는 Meta의 Graph API를 사용합니다. Instagram과 연동되어 있어 같은 인증 토큰을 공유합니다.

```python
# publishers/sns/threads.py
class ThreadsPublisher:
    GRAPH_API = "https://graph.threads.net/v1.0"
    
    def __init__(self, user_id: str, access_token: str):
        self.user_id = user_id
        self.access_token = access_token
    
    def post(self, blog_url: str, title: str, image_url: str = None) -> str:
        """Threads 포스트 발행 (2단계: 생성 → 게시)"""
        
        # 1단계: 컨테이너 생성
        params = {
            "media_type": "TEXT",
            "text": f"{title}\n\n{blog_url}",
            "access_token": self.access_token,
        }
        
        if image_url:
            params["media_type"] = "IMAGE"
            params["image_url"] = image_url
        
        create_resp = requests.post(
            f"{self.GRAPH_API}/{self.user_id}/threads",
            params=params,
        )
        create_resp.raise_for_status()
        container_id = create_resp.json()["id"]
        
        # 2단계: 게시 (최소 30초 대기 후)
        time.sleep(30)
        
        publish_resp = requests.post(
            f"{self.GRAPH_API}/{self.user_id}/threads_publish",
            params={
                "creation_id": container_id,
                "access_token": self.access_token,
            },
        )
        publish_resp.raise_for_status()
        return publish_resp.json()["id"]
```

### Pinterest API

Pinterest는 핀(Pin)을 보드에 게시합니다. 비주얼 콘텐츠가 중심이므로 상품 이미지가 필수입니다.

```python
# publishers/sns/pinterest.py
class PinterestPublisher:
    API_BASE = "https://api.pinterest.com/v5"
    
    def __init__(self, access_token: str, board_id: str):
        self.access_token = access_token
        self.board_id = board_id
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json",
        })
    
    def post(self, blog_url: str, title: str, image_url: str, description: str) -> str:
        """상품 이미지 + 블로그 링크로 핀 생성"""
        payload = {
            "board_id": self.board_id,
            "title": title[:100],
            "description": description[:500],
            "link": blog_url,
            "media_source": {
                "source_type": "image_url",
                "url": image_url,
            },
        }
        
        resp = self.session.post(
            f"{self.API_BASE}/pins",
            json=payload,
        )
        resp.raise_for_status()
        pin_id = resp.json()["id"]
        log(f"Pinterest 핀 생성 완료: {pin_id}", "ok")
        return pin_id
```

### SNS 동시 발행 오케스트레이터

```python
# publishers/sns/__init__.py
from concurrent.futures import ThreadPoolExecutor, as_completed

class SNSPublisher:
    def __init__(self):
        self.twitter = TwitterPublisher(TWITTER_BEARER_TOKEN)
        self.threads = ThreadsPublisher(THREADS_USER_ID, THREADS_TOKEN)
        self.pinterest = PinterestPublisher(PINTEREST_TOKEN, PINTEREST_BOARD_ID)
    
    def publish_all(self, blog_url: str, post: BlogPost, product: Product) -> dict:
        """4개 SNS 동시 발행"""
        results = {}
        
        tasks = {
            "twitter": lambda: self.twitter.post(blog_url, post.title, post.tags),
            "threads": lambda: self.threads.post(blog_url, post.title, product.image),
            "pinterest": lambda: self.pinterest.post(
                blog_url, post.title, product.image, post.description
            ),
        }
        
        with ThreadPoolExecutor(max_workers=3) as executor:
            futures = {
                executor.submit(fn): name
                for name, fn in tasks.items()
            }
            for future in as_completed(futures):
                name = futures[future]
                try:
                    results[name] = future.result()
                except Exception as e:
                    log(f"{name} SNS 발행 실패: {e}", "error")
                    results[name] = "failed"
        
        return results
```

---

## 실패 사례 & 해결책

**실패 1: Twitter API v1.1 → v2 전환**

기존에 v1.1을 사용하던 코드가 갑자기 동작을 안 했습니다. 2023년 이후 무료 플랜은 v2만 지원합니다.

→ **해결**: `tweepy` 라이브러리 → 직접 requests로 v2 API 호출로 전환.

**실패 2: Threads 컨테이너 생성 후 즉시 게시 시 오류**

컨테이너를 만들고 1초 후 게시하면 "컨테이너가 준비되지 않았습니다" 오류가 납니다.

→ **해결**: 30초 대기 후 게시. Meta 문서에 명시된 권장 대기 시간입니다.

---

## 배운 점 / 주의사항

**SNS는 직접 수익보다 색인 신호용으로 생각하세요.** 블로그 글을 SNS에 공유하면 소셜 신호(social signal)로 구글이 빠르게 크롤링합니다.

**API 토큰 만료를 모니터링하세요.** Instagram/Threads 토큰은 60~90일 후 만료됩니다. 만료 2주 전에 갱신 알림을 받도록 설정하세요.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2. [[02] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기](/posts/auto-publishing-02/)
3. [[03] 쿠팡 크롤링 — Access Denied 뚫기: Chrome CDP로 WAF 우회](/posts/auto-publishing-03/)
4. [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
5. [[05] Claude CLI와 Gemini API로 상품 소개글 자동 생성하기](/posts/auto-publishing-05/)
6. [[06] WordPress REST API로 멀티 사이트 자동 발행 구현](/posts/auto-publishing-06/)
7. [[07] 티스토리 자동 발행 — Playwright Persistent Context로 Kakao SSO 유지](/posts/auto-publishing-07/)
8. [[08] 네이버 RSA 암호화 로그인으로 블로그·카페 자동 발행](/posts/auto-publishing-08/)
9. [[09] GitHub Pages 자동 발행 — Jekyll Markdown 자동 커밋·푸시](/posts/auto-publishing-09/)
10. **[현재 글] [Auto-Publishing] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest**
11. [[11] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행](/posts/auto-publishing-11/)
12. [[12] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기](/posts/auto-publishing-12/)
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T01~T08] 전체 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
