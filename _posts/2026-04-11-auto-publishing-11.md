---
title: "[Auto-Publishing] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행"
date: 2026-04-11 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, 뉴스픽API, 정책브리핑, RSS자동화]
description: "상품 소개글 외에도 정보성 콘텐츠를 자동화하는 방법을 공개합니다. 뉴스픽 파트너스 API와 정책브리핑 RSS를 활용해 최신 뉴스·정책 정보를 자동 수집하고 블로그로 발행합니다."
---

## 개요

자동 발행 시스템이 상품 소개글만 올리면 블로그가 '광고판'처럼 보입니다. 구글도 콘텐츠 다양성을 평가하고, 독자도 정보성 글이 섞여야 방문할 이유가 생깁니다.

이번 편에서는 **정보성 콘텐츠를 자동 수집·발행**하는 두 가지 소스를 설명합니다:

1. **뉴스픽(Newspick)**: 큐레이션된 뉴스 콘텐츠
2. **정책브리핑**: 정부 공식 정책 정보 (공공데이터 무료 활용)

---

## 왜 정보성 콘텐츠가 필요한가

상품 소개글은 **트랜잭션 키워드**(구매 의도)를 타겟하고, 정보성 글은 **정보성 키워드**(알고 싶은 것)를 타겟합니다.

두 유형이 함께 있어야:
- 신규 독자가 정보 검색으로 유입되고
- 내부 링크를 따라 상품 소개글로 이동하고
- 구매로 전환되는 흐름이 만들어집니다.

---

## 핵심 구현

### 뉴스픽 파트너스 API

뉴스픽은 파트너스 계정을 신청하면 콘텐츠 API를 제공합니다.

```python
# sources/newspick.py
import requests
from common.session import SessionManager

CONTENT_URL = "https://www.newspick.com/api/1/partner/contents"
LOGIN_URL = "https://www.newspick.com/auth"

class NewspickSource:
    def __init__(self, email: str, password: str):
        self.session_mgr = SessionManager("newspick")
        self.email = email
        self.password = password
    
    def _check_session(self) -> bool:
        """세션 유효성 확인 — partners API 응답이 JSON이면 유효"""
        try:
            resp = self.session_mgr.post(
                f"{CONTENT_URL}?channelNo=1&pageSize=1",
                timeout=10,
            )
            if resp.status_code == 200:
                ct = resp.headers.get("content-type", "")
                if "json" in ct:
                    data = resp.json()
                    return "recomList" in data or "contentList" in data
        except Exception:
            pass
        return False
    
    def _login(self) -> bool:
        """Playwright로 뉴스픽 로그인 → 세션 쿠키 저장"""
        from playwright.sync_api import sync_playwright
        
        with sync_playwright() as pw:
            browser = pw.chromium.launch(headless=True)
            page = browser.new_page()
            
            page.goto(LOGIN_URL, wait_until="domcontentloaded")
            page.fill("input[type='email']", self.email)
            page.fill("input[type='password']", self.password)
            page.click("button[type='submit']")
            
            # 카카오 간편로그인 처리
            kakao_selectors = [
                ".kc_item_select .kc_btn_simple",
                "button.kc_btn_simple",
                f'a:has-text("{self.email}"):not(.btn_delete)',
            ]
            
            for sel in kakao_selectors:
                loc = page.locator(sel)
                if loc.count() > 0:
                    loc.first.click(timeout=2000)
                    break
            
            # 로그인 완료 대기
            page.wait_for_url("**/newspick.com/**", timeout=15000)
            
            # 쿠키를 SessionManager에 저장
            cookies = page.context.cookies()
            for c in cookies:
                self.session_mgr._session.cookies.set(
                    c["name"], c["value"], domain=c["domain"]
                )
            self.session_mgr.save()
            
            browser.close()
        return True
    
    def fetch(self, channel: int = 1, limit: int = 10) -> list:
        """최신 뉴스 목록 가져오기"""
        if not self._check_session():
            log("뉴스픽 세션 만료 — 재로그인", "warn")
            self._login()
        
        resp = self.session_mgr.post(
            CONTENT_URL,
            json={
                "channelNo": channel,
                "pageSize": limit,
                "inputSwitch": False,
            },
            timeout=15,
        )
        resp.raise_for_status()
        
        items = resp.json().get("contentList", [])
        return [
            {
                "title": item["title"],
                "summary": item.get("summary", ""),
                "url": item["originUrl"],
                "published_at": item["publishedAt"],
                "category": item.get("categoryName", ""),
            }
            for item in items
        ]
```

### 정책브리핑 RSS 파싱

정책브리핑(korea.kr)은 공공데이터 RSS를 무료로 제공합니다.

```python
# sources/policy_briefing.py
import feedparser
from datetime import datetime

RSS_FEEDS = {
    "경제": "https://www.korea.kr/rss/policy.do?cat=economy",
    "복지": "https://www.korea.kr/rss/policy.do?cat=welfare",
    "일자리": "https://www.korea.kr/rss/policy.do?cat=job",
    "주택": "https://www.korea.kr/rss/policy.do?cat=housing",
}

class PolicyBriefingSource:
    def fetch(self, category: str = "경제", limit: int = 5) -> list:
        """정책브리핑 RSS 파싱"""
        rss_url = RSS_FEEDS.get(category, RSS_FEEDS["경제"])
        feed = feedparser.parse(rss_url)
        
        articles = []
        for entry in feed.entries[:limit]:
            articles.append({
                "title": entry.title,
                "summary": entry.get("summary", ""),
                "url": entry.link,
                "published_at": entry.get("published", ""),
                "category": category,
                "source": "정책브리핑",
            })
        
        return articles
```

### AI로 정보성 글 변환

원본 뉴스/정책 정보를 AI가 블로그 글 형태로 재작성합니다.

```python
# pipelines/newspick_to_blog.py
def run():
    # 뉴스픽에서 최신 5개 수집
    news_items = NewspickSource(EMAIL, PASSWORD).fetch(limit=5)
    
    # 정책브리핑에서 2개 추가
    policy_items = PolicyBriefingSource().fetch(category="경제", limit=2)
    
    all_items = news_items + policy_items
    
    for item in all_items:
        # AI로 정보성 블로그 글 변환
        prompt = f"""
다음 뉴스/정책 정보를 독자에게 유용한 블로그 글로 재작성해주세요.

제목: {item['title']}
요약: {item['summary']}
출처: {item.get('source', '뉴스픽')}

요구사항:
- 800~1,000자 블로그 글 (한국어)
- 독자 관점에서 "왜 이 정보가 나에게 중요한가" 설명
- 원본 출처 링크 포함
- 저작권을 침해하지 않도록 원문을 그대로 옮기지 말 것
"""
        post = GeminiWriter().write_from_prompt(prompt)
        
        # WordPress 발행
        for profile in WP_PROFILES:
            WordPressPublisher(profile).publish(post)
        
        time.sleep(random.uniform(10, 20))  # 발행 간격
```

---

## 실패 사례 & 해결책

**실패 1: 정책브리핑 RSS가 가끔 내려감**

`feedparser.parse()`는 접속 실패 시 예외가 아닌 빈 feed를 반환합니다. 이것을 성공으로 착각해 빈 글이 발행됐습니다.

→ **해결**: `len(feed.entries) == 0` 체크 후 실패로 처리. 이전 RSS 캐시 데이터를 폴백으로 사용.

**실패 2: AI가 뉴스 내용을 과도하게 요약**

뉴스 원문의 핵심이 날아가고 너무 얕은 글이 됐습니다.

→ **해결**: 요약 지시를 제거하고 "독자 관점에서 맥락 추가" 방향으로 프롬프트 변경.

---

## 배운 점 / 주의사항

**저작권에 주의하세요.** 뉴스 원문을 그대로 복사하면 저작권 문제가 됩니다. AI로 재작성해도 원문과 너무 유사하면 위험합니다. 반드시 요약·재해석 수준으로만 활용하세요.

**정책브리핑은 공공데이터라 제약이 없습니다.** 정부 정책 정보는 공공저작물로 자유롭게 활용할 수 있습니다. 출처만 표기하면 됩니다.

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
10. [[10] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest](/posts/auto-publishing-10/)
11. **[현재 글] [Auto-Publishing] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행**
12. [[12] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기](/posts/auto-publishing-12/)
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T01~T08] 전체 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
