---
title: "[Auto-Publishing] WordPress REST API로 멀티 사이트 자동 발행 구현"
date: 2026-04-06 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, WordPress, RESTAPI, JWT인증]
description: "WordPress REST API와 JWT 인증을 활용해 여러 WordPress 사이트에 동시 자동 발행하는 파이프라인 구현을 설명합니다. 멀티 프로필 관리와 이미지 업로드까지 다룹니다."
---

## 개요

WordPress는 자동 발행에 가장 친화적인 플랫폼입니다. 공식 REST API가 잘 문서화되어 있고, JWT 인증 플러그인을 설치하면 완전한 자동화가 가능합니다.

이번 편에서는 **여러 WordPress 사이트에 동시 발행**하는 구조를 설명합니다. 하나의 콘텐츠를 복수의 사이트에 배포해 노출을 극대화하는 전략입니다.

---

## 왜 WordPress를 먼저 공략했나

자동화 관점에서 WordPress는 몇 가지 큰 장점이 있습니다:

1. **공식 REST API**: 문서가 명확하고, 인증 방식이 표준적
2. **플러그인 생태계**: JWT Authentication 플러그인으로 토큰 인증 즉시 추가
3. **멀티사이트**: 하나의 코드로 여러 블로그를 관리 가능
4. **무제한 발행**: 플랫폼 제한이 없음 (티스토리, 네이버는 발행 횟수 제한 존재)

---

## 핵심 구현

### WordPress 사이트 프로필 설정

```python
# publishers/wordpress.py
from dataclasses import dataclass
from typing import Optional
import requests

@dataclass
class WordPressProfile:
    """WordPress 사이트 설정"""
    name: str
    base_url: str        # https://myblog.com
    username: str
    app_password: str    # WordPress 앱 비밀번호 (JWT 대신 사용 가능)
    category_id: int     # 기본 카테고리 ID
    author_id: int = 1

# 멀티 프로필 설정
PROFILES = [
    WordPressProfile(
        name="블로그A",
        base_url="https://blog-a.com",
        username="admin",
        app_password="xxxx xxxx xxxx xxxx",
        category_id=5,
    ),
    WordPressProfile(
        name="블로그B", 
        base_url="https://blog-b.com",
        username="editor",
        app_password="yyyy yyyy yyyy yyyy",
        category_id=3,
    ),
]
```

### 기본 인증 (Application Password)

WordPress 5.6+에서는 별도 플러그인 없이 **앱 비밀번호**로 REST API 인증이 가능합니다.

```python
class WordPressPublisher:
    def __init__(self, profile: WordPressProfile):
        self.profile = profile
        self.session = requests.Session()
        self.session.auth = (profile.username, profile.app_password)
        self.session.headers.update({
            "Content-Type": "application/json",
        })
    
    def _api(self, endpoint: str) -> str:
        return f"{self.profile.base_url}/wp-json/wp/v2/{endpoint}"
```

### 이미지 업로드

상품 이미지를 먼저 WordPress 미디어 라이브러리에 업로드합니다.

```python
def upload_image(self, image_url: str, filename: str) -> Optional[int]:
    """이미지 URL → WordPress 미디어 라이브러리 업로드 → media_id 반환"""
    try:
        # 원본 이미지 다운로드
        img_resp = requests.get(image_url, timeout=15)
        img_resp.raise_for_status()
        
        # 파일명에서 확장자 추출
        ext = image_url.split(".")[-1].split("?")[0]
        if ext not in ("jpg", "jpeg", "png", "webp"):
            ext = "jpg"
        
        # WordPress 미디어 API로 업로드
        upload_resp = self.session.post(
            self._api("media"),
            headers={
                "Content-Disposition": f'attachment; filename="{filename}.{ext}"',
                "Content-Type": f"image/{ext}",
            },
            data=img_resp.content,
            timeout=30,
        )
        upload_resp.raise_for_status()
        return upload_resp.json()["id"]
        
    except Exception as e:
        log(f"이미지 업로드 실패: {e}", "warn")
        return None

def publish(self, post: BlogPost, product: Product) -> str:
    """WordPress에 포스트 발행 → 발행된 URL 반환"""
    
    # 1. 이미지 업로드
    media_id = self.upload_image(product.image, product.name[:50])
    
    # 2. 포스트 데이터 구성
    payload = {
        "title": post.title,
        "content": post.content,
        "excerpt": post.description,
        "status": "publish",
        "categories": [self.profile.category_id],
        "tags": self._get_or_create_tags(post.tags),
        "author": self.profile.author_id,
    }
    
    if media_id:
        payload["featured_media"] = media_id
    
    # 3. 발행
    resp = self.session.post(
        self._api("posts"),
        json=payload,
        timeout=30,
    )
    resp.raise_for_status()
    
    published_url = resp.json()["link"]
    log(f"WordPress 발행 완료: {published_url}", "ok")
    return published_url

def _get_or_create_tags(self, tag_names: list[str]) -> list[int]:
    """태그명으로 ID 조회, 없으면 생성"""
    tag_ids = []
    for name in tag_names:
        # 태그 검색
        resp = self.session.get(
            self._api("tags"),
            params={"search": name, "per_page": 1},
        )
        tags = resp.json()
        
        if tags:
            tag_ids.append(tags[0]["id"])
        else:
            # 없으면 생성
            create_resp = self.session.post(
                self._api("tags"),
                json={"name": name},
            )
            tag_ids.append(create_resp.json()["id"])
    
    return tag_ids
```

### 멀티사이트 동시 발행

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def publish_to_all(post: BlogPost, product: Product) -> dict[str, str]:
    """모든 WordPress 프로필에 동시 발행"""
    results = {}
    
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = {
            executor.submit(
                WordPressPublisher(profile).publish, post, product
            ): profile.name
            for profile in PROFILES
        }
        
        for future in as_completed(futures):
            name = futures[future]
            try:
                url = future.result()
                results[name] = url
            except Exception as e:
                log(f"{name} 발행 실패: {e}", "error")
                results[name] = "failed"
    
    return results
```

---

## 실패 사례 & 해결책

**실패 1: 앱 비밀번호 공백 처리**

WordPress 앱 비밀번호는 `xxxx xxxx xxxx xxxx` 형태로 공백이 포함됩니다. 환경변수로 읽을 때 공백이 제거되면 인증 실패가 납니다.

→ **해결**: `.env` 파일에서 따옴표로 감싸서 저장. `APP_PASSWORD="xxxx xxxx xxxx xxxx"`.

**실패 2: 이미지 중복 업로드**

같은 상품을 여러 사이트에 발행할 때마다 이미지를 새로 업로드하면 스토리지가 낭비됩니다.

→ **해결**: 상품 URL → 해시값으로 로컬 캐시. 이미 업로드된 이미지는 재사용.

---

## 배운 점 / 주의사항

**카테고리와 태그는 사전에 만들어두세요.** API로 자동 생성도 되지만, 너무 많은 태그가 생기면 WordPress 관리 화면이 지저분해집니다.

**발행 상태를 `draft`로 먼저 테스트하세요.** `status: "draft"`로 발행하면 실제 공개가 안 됩니다. 초기 테스트에 유용합니다.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2. [[02] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기](/posts/auto-publishing-02/)
3. [[03] 쿠팡 크롤링 — Access Denied 뚫기: Chrome CDP로 WAF 우회](/posts/auto-publishing-03/)
4. [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
5. [[05] Claude CLI와 Gemini API로 상품 소개글 자동 생성하기](/posts/auto-publishing-05/)
6. **[현재 글] [Auto-Publishing] WordPress REST API로 멀티 사이트 자동 발행 구현**
7. [[07] 티스토리 자동 발행 — Playwright Persistent Context로 Kakao SSO 유지](/posts/auto-publishing-07/)
8. [[08] 네이버 RSA 암호화 로그인으로 블로그·카페 자동 발행](/posts/auto-publishing-08/)
9. [[09] GitHub Pages 자동 발행 — Jekyll Markdown 자동 커밋·푸시](/posts/auto-publishing-09/)
10. [[10] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest](/posts/auto-publishing-10/)
11. [[11] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행](/posts/auto-publishing-11/)
12. [[12] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기](/posts/auto-publishing-12/)
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T01~T08] 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
