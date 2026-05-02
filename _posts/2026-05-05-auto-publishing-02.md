---
title: "[Auto-Publishing] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기"
date: 2026-05-05 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, AI, 파이썬, 블로그자동화, 패시브인컴, SEO키워드, ItemScout]
description: "자동 발행 시스템의 시작점인 키워드 수집을 자동화한 방법을 공개합니다. ItemScout, 판다랭크, 네이버 데이터랩을 조합해 검색 수요가 검증된 키워드 풀 5,000개를 만드는 파이프라인을 설명합니다."
---

## 개요

자동 발행 시스템에서 가장 먼저 해결해야 할 문제는 **"무엇을 쓸 것인가"**입니다.

키워드가 없으면 콘텐츠가 없고, 콘텐츠가 없으면 트래픽이 없습니다. 그리고 트래픽이 없으면 수익도 없습니다.

이번 편에서는 **검색 수요가 실제로 존재하는 키워드를 자동으로 수집하는 방법**을 설명합니다. 도구 세 가지를 조합합니다: ItemScout, 판다랭크, 네이버 DataLab.

---

## 왜 키워드 자동화가 필요했나

처음에는 수작업으로 키워드를 뽑았습니다. 네이버 쇼핑에서 카테고리를 돌아다니며 "이거 괜찮겠다"싶은 상품을 골랐습니다.

문제는 **주관이 개입된다**는 점이었습니다. 내가 보기에 좋아 보이는 상품이 검색량이 없는 경우가 많았습니다. 반대로 검색량이 엄청난 키워드를 놓치기도 했습니다.

자동화의 목표는 감이 아닌 **데이터 기반**으로 키워드를 선별하는 것입니다.

---

## 핵심 구현

### 1단계: ItemScout로 경쟁도 낮은 키워드 수집

ItemScout는 네이버 쇼핑 키워드 분석 도구입니다. 검색량, 경쟁 강도, 클릭률 데이터를 제공합니다.

```python
# sources/keyword_itemscout.py
import requests
from typing import List, Dict

ITEMSCOUT_API = "https://api.itemscout.io/v2"

class ItemScoutCollector:
    def __init__(self, api_key: str):
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        })
    
    def fetch_keywords(self, category: str, limit: int = 500) -> List[Dict]:
        """카테고리별 키워드 수집 — 경쟁도 낮고 검색량 높은 순"""
        resp = self.session.post(
            f"{ITEMSCOUT_API}/keywords/search",
            json={
                "category": category,
                "sort": "search_count",
                "competition": "low",  # 경쟁 낮은 것만
                "min_search_count": 1000,  # 월 1,000 이상
                "limit": limit,
            },
            timeout=15,
        )
        resp.raise_for_status()
        return resp.json()["keywords"]
```

수집 기준:
- **월 검색량 1,000 이상**: 트래픽이 아예 없는 키워드 제외
- **경쟁 강도 '낮음'**: 이미 대형 쇼핑몰이 꽉 잡은 키워드는 패스
- **클릭률 3% 이상**: 검색 후 실제로 클릭하는 키워드만

### 2단계: 판다랭크로 트렌드 확인

검색량이 있더라도 하락세인 키워드는 의미가 없습니다. 판다랭크로 최근 3개월 트렌드를 확인합니다.

```python
# sources/keyword_pandarank.py
class PandarankCollector:
    def filter_trending(self, keywords: List[str]) -> List[str]:
        """최근 3개월 검색량이 상승세인 키워드만 필터"""
        trending = []
        for kw in keywords:
            trend = self._get_trend(kw)
            # 3개월 전 대비 현재 검색량이 20% 이상 증가한 것만
            if trend["current"] >= trend["three_months_ago"] * 1.2:
                trending.append(kw)
        return trending
    
    def _get_trend(self, keyword: str) -> Dict:
        resp = self.session.get(
            f"{PANDARANK_API}/trend",
            params={"keyword": keyword, "period": "3m"},
            timeout=10,
        )
        return resp.json()
```

### 3단계: 네이버 DataLab으로 최종 검증

DataLab은 네이버 공식 트렌드 API입니다. 판다랭크 트렌드를 DataLab으로 교차 검증합니다.

```python
# sources/keyword_datalab.py
import hmac
import hashlib

class NaverDataLabCollector:
    """네이버 데이터랩 검색어 트렌드 API"""
    
    DATALAB_URL = "https://openapi.naver.com/v1/datalab/search"
    
    def __init__(self, client_id: str, client_secret: str):
        self.client_id = client_id
        self.client_secret = client_secret
    
    def get_trend_score(self, keyword: str) -> float:
        """100점 만점 기준 최근 상대 검색량 반환"""
        payload = {
            "startDate": "2026-02-01",
            "endDate": "2026-04-30",
            "timeUnit": "month",
            "keywordGroups": [
                {"groupName": keyword, "keywords": [keyword]}
            ],
        }
        resp = requests.post(
            self.DATALAB_URL,
            json=payload,
            headers={
                "X-Naver-Client-Id": self.client_id,
                "X-Naver-Client-Secret": self.client_secret,
            },
            timeout=10,
        )
        data = resp.json()
        # 최근 달 ratio 값 (0~100)
        return data["results"][0]["data"][-1]["ratio"]
```

### 통합 파이프라인

세 단계를 연결해 최종 키워드 풀을 만듭니다.

```python
# pipelines/keyword_collection.py
def collect_keywords() -> List[str]:
    categories = ["무선이어폰", "블루투스스피커", "스마트워치", "노트북거치대"]
    
    all_keywords = []
    for category in categories:
        # 1단계: ItemScout 수집
        raw = ItemScoutCollector(ITEMSCOUT_KEY).fetch_keywords(category, limit=500)
        kw_list = [item["keyword"] for item in raw]
        
        # 2단계: 판다랭크 트렌드 필터
        trending = PandarankCollector().filter_trending(kw_list)
        
        # 3단계: DataLab 점수 30 이상만
        datalab = NaverDataLabCollector(NAVER_ID, NAVER_SECRET)
        verified = [kw for kw in trending if datalab.get_trend_score(kw) >= 30]
        
        all_keywords.extend(verified)
    
    # 중복 제거 후 반환
    return list(dict.fromkeys(all_keywords))
```

실제로 이 파이프라인을 돌리면 카테고리 4개에서 약 5,000개 키워드가 나옵니다.

---

## 실패 사례 & 해결책

**실패 1: API 요금 폭탄**

ItemScout API는 호출 건수로 과금됩니다. 처음에 limit을 2,000으로 설정했다가 하루 만에 월 예산을 초과했습니다.

→ **해결**: limit을 500으로 낮추고, 수집한 키워드를 SQLite에 캐시. 같은 키워드를 7일 이내에 재조회하지 않도록 제한.

**실패 2: 검색량은 많지만 구매 의도가 없는 키워드**

"무선이어폰 추천"은 검색량이 많지만 실제 구매로 이어지지 않는 경우가 많습니다. "무선이어폰 20000원 이하"처럼 가격이 붙은 키워드의 전환율이 훨씬 높았습니다.

→ **해결**: 키워드 필터에 구매 의도 패턴 추가 (`가격`, `추천`, `후기`, `비교` 등 접미사 분류).

---

## 배운 점 / 주의사항

**검색량 ≠ 수익**입니다. 검색량이 많아도 광고가 많거나, 쇼핑 탭에 경쟁자가 많으면 유입이 안 됩니다. ItemScout의 경쟁 강도 필터가 생각보다 중요합니다.

**DataLab API 호출 제한**은 하루 1,000건입니다. 키워드 5,000개를 검증하려면 5일이 걸립니다. 스케줄러로 매일 1,000건씩 나눠서 처리하는 것이 안정적입니다.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2. **[현재 글] [Auto-Publishing] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기**
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
