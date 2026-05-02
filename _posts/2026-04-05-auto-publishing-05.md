---
title: "[Auto-Publishing] Claude CLI와 Gemini API로 상품 소개글 자동 생성하기"
date: 2026-04-05 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, AI, 파이썬, 블로그자동화, 패시브인컴, ClaudeAPI, GeminiAPI, AI글쓰기]
description: "쿠팡·알리익스프레스 상품 정보를 Claude CLI와 Gemini API로 자동 글쓰기로 변환하는 파이프라인을 공개합니다. 프롬프트 설계, API 비용 절감 전략, 품질 검증까지 다룹니다."
---

## 개요

크롤링으로 상품 정보를 수집했다면, 다음 단계는 그것을 **읽히는 글**로 변환하는 것입니다.

상품명, 가격, 특징 데이터만 있는 상태에서 "이 상품이 왜 좋은지, 누가 사면 좋은지"를 담은 블로그 글이 되어야 합니다.

이 작업을 AI에게 맡깁니다. Claude와 Gemini를 상황에 따라 선택적으로 사용하는 이유와 방법을 설명합니다.

---

## 왜 Claude와 Gemini를 함께 쓰나

두 AI를 동시에 사용하는 이유는 **비용과 품질의 균형** 때문입니다.

| 항목 | Claude (Sonnet) | Gemini (Flash) |
|------|----------------|----------------|
| 글 품질 | 높음 | 보통 |
| 속도 | 느림 | 빠름 |
| 가격 | 비쌈 | 저렴 |
| 적합한 용도 | 프리미엄 포스팅 | 대량 발행 |

전략은 간단합니다: **고단가 상품은 Claude, 일반 상품은 Gemini**.

---

## 핵심 구현

### 공통 Writer 인터페이스

두 AI를 교체 가능하게 만들기 위해 추상 인터페이스를 정의합니다.

```python
# ai/base_writer.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class Product:
    name: str
    price: str
    features: list[str]
    url: str
    image: str
    category: str

@dataclass  
class BlogPost:
    title: str
    content: str  # HTML 또는 Markdown
    description: str  # SEO 메타 설명 (150자)
    tags: list[str]

class BaseWriter(ABC):
    @abstractmethod
    def write(self, product: Product) -> BlogPost:
        pass
```

### Claude Sonnet Writer

Claude는 길고 자연스러운 한국어 글쓰기에 강합니다.

```python
# ai/claude_writer.py
import subprocess
import json

class ClaudeWriter(BaseWriter):
    def write(self, product: Product) -> BlogPost:
        prompt = self._build_prompt(product)
        
        # Claude CLI 사용 (API 키 환경변수 자동 인식)
        result = subprocess.run(
            ["claude", "-p", prompt, "--output-format", "json"],
            capture_output=True,
            text=True,
            timeout=60,
        )
        
        if result.returncode != 0:
            raise RuntimeError(f"Claude CLI 오류: {result.stderr}")
        
        response = json.loads(result.stdout)
        return self._parse_response(response["result"], product)
    
    def _build_prompt(self, product: Product) -> str:
        return f"""
다음 상품에 대한 블로그 소개글을 작성해주세요.

상품명: {product.name}
가격: {product.price}
카테고리: {product.category}
주요 특징:
{chr(10).join(f'- {f}' for f in product.features)}

요구사항:
1. 제목: SEO 최적화된 클릭하고 싶은 제목 (50자 이내)
2. 본문: 1,000~1,500자, 자연스러운 구어체 한국어
   - 이 상품이 필요한 독자 상황 설명
   - 핵심 기능 3가지 상세 설명  
   - 가격 대비 가치 언급
3. SEO 설명: 150자 이내 메타 설명
4. 태그: 관련 키워드 5개

JSON 형식으로 반환: {{"title": "", "content": "", "description": "", "tags": []}}
"""
```

### Gemini Flash Writer

Gemini는 속도와 비용이 장점입니다. 대량 발행에 적합합니다.

```python
# ai/gemini_writer.py
import google.generativeai as genai

class GeminiWriter(BaseWriter):
    def __init__(self, api_key: str):
        genai.configure(api_key=api_key)
        self.model = genai.GenerativeModel("gemini-1.5-flash")
    
    def write(self, product: Product) -> BlogPost:
        prompt = self._build_prompt(product)
        
        response = self.model.generate_content(
            prompt,
            generation_config=genai.GenerationConfig(
                temperature=0.7,
                response_mime_type="application/json",
            ),
        )
        
        data = json.loads(response.text)
        return BlogPost(
            title=data["title"],
            content=data["content"],
            description=data["description"],
            tags=data["tags"],
        )
```

### 자동 AI 선택 로직

```python
# ai/writer_factory.py
def get_writer(product: Product) -> BaseWriter:
    """상품 가격에 따라 AI 자동 선택"""
    price_num = int(product.price.replace(",", "").replace("원", ""))
    
    # 5만원 이상 고단가 상품은 Claude
    if price_num >= 50000:
        return ClaudeWriter()
    
    # 일반 상품은 Gemini (비용 절감)
    return GeminiWriter(api_key=GEMINI_API_KEY)
```

---

## 프롬프트 설계 원칙

처음 프롬프트는 단순했습니다: "이 상품 소개글 써줘". 결과물은 판매 페이지 카피 같았고, 블로그 글처럼 보이지 않았습니다.

반복 개선을 거쳐 효과적이었던 요소들:

**① 독자 상황 설정**: "무선 이어폰을 찾는 독자가 출퇴근 중에 이 글을 읽는다고 가정하고..."
**② 구체적 길이 제한**: "1,000~1,500자" (짧으면 얕고, 길면 산만해짐)
**③ 구조 지정**: 서론 → 기능 설명 → 가격 가치 → 구매 링크 순서
**④ 금지 표현 명시**: "과장 광고 문구 사용 금지, '최고', '압도적' 같은 표현 제외"

---

## 실패 사례 & 해결책

**실패 1: AI가 상품 특징을 꾸며냄**

Gemini가 실제로 없는 기능을 지어냈습니다. "방수 기능 있음"이라고 썼는데 실제 상품에는 없었습니다.

→ **해결**: 프롬프트에 "제공된 특징 목록 외의 내용을 추가하지 말 것" 명시. 발행 전 상품명 + 핵심 기능 교차 검증 단계 추가.

**실패 2: Claude CLI 타임아웃**

복잡한 프롬프트에서 Claude CLI가 60초를 초과했습니다.

→ **해결**: `timeout=120`으로 늘리고, 실패 시 Gemini로 자동 폴백.

```python
def write_with_fallback(product: Product) -> BlogPost:
    try:
        return ClaudeWriter().write(product)
    except (RuntimeError, TimeoutError) as e:
        log(f"Claude 실패, Gemini로 폴백: {e}", "warn")
        return GeminiWriter(GEMINI_API_KEY).write(product)
```

---

## 배운 점 / 주의사항

**AI 글쓰기는 검수가 필요합니다.** 완전 자동화를 목표로 하더라도, 초기에는 10~20%를 샘플링해서 품질을 확인하세요. AI가 틀린 정보를 자신 있게 쓰는 경우가 있습니다.

**API 비용을 모니터링하세요.** Gemini Flash는 저렴하지만 Claude는 의외로 빠르게 청구됩니다. 일일 토큰 한도를 설정해두는 것이 안전합니다.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2. [[02] ItemScout·판다랭크·DataLab으로 키워드 풀 5,000개 만들기](/posts/auto-publishing-02/)
3. [[03] 쿠팡 크롤링 — Access Denied 뚫기: Chrome CDP로 WAF 우회](/posts/auto-publishing-03/)
4. [[04] 알리익스프레스 크롤링 — Playwright로 CAPTCHA 우회](/posts/auto-publishing-04/)
5. **[현재 글] [Auto-Publishing] Claude CLI와 Gemini API로 상품 소개글 자동 생성하기**
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
