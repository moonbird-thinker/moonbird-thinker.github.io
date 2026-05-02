---
title: "[Auto-Publishing] 네이버 RSA 암호화 로그인으로 블로그·카페 자동 발행"
date: 2026-05-17 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, 네이버블로그, RSA로그인, 네이버카페]
description: "네이버 자동화의 핵심인 RSA 암호화 로그인과 CDP 쿠키 추출 방법을 설명합니다. 로그인 후 블로그·카페에 SE 에디터 API를 통해 자동 발행하는 전체 흐름을 공개합니다."
---

## 개요

네이버 자동화는 카카오와 더불어 국내 플랫폼 중 가장 어렵습니다. 네이버는 로그인 보안이 강력하고, 공식 API 없이 블로그나 카페에 글을 쓰려면 SE 에디터의 내부 API를 역엔지니어링해야 합니다.

이번 편에서는 두 가지 로그인 방식과 블로그·카페 발행 구현을 설명합니다.

---

## 네이버 로그인 전략: CDP 우선, RSA 폴백

로그인 방식은 우선순위를 두고 시도합니다:

1. **CDP 쿠키 추출** (우선): 이미 로그인된 로컬 Chrome 프로필에서 쿠키를 복사
2. **RSA 암호화 로그인** (폴백): ID/PW를 RSA 공개키로 암호화해 로그인 API 호출

CDP 방식이 더 안정적이고 추가 인증 위험이 낮습니다.

---

## 핵심 구현

### CDP 쿠키 추출 (우선순위 1)

```python
# common/auth.py
import shutil
from pathlib import Path
from playwright.sync_api import sync_playwright

NAVER_COOKIES = ["NID_AUT", "NID_SES", "NID_JKL"]
CHROME_PROFILE = Path.home() / "Library/Application Support/Google/Chrome/Default"
TEMP_PROFILE = Path(".sessions/naver_chrome_temp")

def extract_naver_cookies_from_chrome() -> dict:
    """로컬 Chrome 프로필에서 네이버 쿠키 추출"""
    
    # Chrome 프로필 복사 (singleton lock 파일 제외)
    if TEMP_PROFILE.exists():
        shutil.rmtree(TEMP_PROFILE)
    
    shutil.copytree(
        CHROME_PROFILE,
        TEMP_PROFILE,
        ignore=shutil.ignore_patterns(
            "SingletonLock", "SingletonCookie", "lockfile", "*.log"
        ),
    )
    
    cookies = {}
    
    with sync_playwright() as pw:
        context = pw.chromium.launch_persistent_context(
            user_data_dir=str(TEMP_PROFILE),
            headless=True,
        )
        page = context.new_page()
        
        # 네이버 방문으로 쿠키 활성화
        page.goto("https://www.naver.com", wait_until="domcontentloaded", timeout=15000)
        page.goto("https://blog.naver.com", wait_until="domcontentloaded", timeout=15000)
        
        # 쿠키 추출
        all_cookies = context.cookies()
        for cookie in all_cookies:
            if cookie["name"] in NAVER_COOKIES and "naver.com" in cookie["domain"]:
                cookies[cookie["name"]] = cookie["value"]
        
        context.close()
    
    # 임시 프로필 삭제
    shutil.rmtree(TEMP_PROFILE, ignore_errors=True)
    
    has_auth = "NID_AUT" in cookies and "NID_SES" in cookies
    log(f"CDP 쿠키 추출: {'성공' if has_auth else '실패'}", "ok" if has_auth else "warn")
    return cookies
```

### RSA 암호화 로그인 (폴백)

네이버 로그인 API는 비밀번호를 평문으로 받지 않습니다. RSA 공개키로 암호화해야 합니다.

```python
# common/auth.py (계속)
import rsa
import base64
import uuid
import time
import requests

NAVER_LOGIN_URL = "https://nid.naver.com/nidlogin.login"
NAVER_KEY_URL = "https://nid.naver.com/login/ext/keys.nhn"

def rsa_login(username: str, password: str) -> dict:
    """RSA 암호화 네이버 로그인"""
    session = requests.Session()
    session.headers.update({
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                     "AppleWebKit/537.36 Chrome/124.0.0.0 Safari/537.36",
    })
    
    # RSA 공개키 발급
    key_resp = session.get(NAVER_KEY_URL)
    key_data = key_resp.text.split(",")
    
    session_key = key_data[0]
    key_name = key_data[1]
    e_val = int(key_data[2], 16)
    n_val = int(key_data[3], 16)
    
    # RSA 암호화
    pub_key = rsa.PublicKey(n_val, e_val)
    message = f"{len(session_key):02d}{session_key}{len(username):02d}{username}{len(password):02d}{password}"
    encrypted = rsa.encrypt(message.encode("utf-8"), pub_key)
    encrypted_b64 = base64.b64encode(encrypted).decode("utf-8")
    
    # 브라우저 지문 (BVSD) 생성
    bvsd = {
        "uuid": str(uuid.uuid4()),
        "em": {
            "version": "1.0.0",
            "platform": "macOS",
            "app_key": "naverapp",
        },
        "ts": int(time.time() * 1000),
    }
    
    # 로그인 요청
    login_resp = session.post(
        NAVER_LOGIN_URL,
        data={
            "svctype": "0",
            "enctp": "1",
            "encpw": encrypted_b64,
            "encnm": key_name,
            "bvsd": json.dumps(bvsd),
            "smart_LEVEL": "-1",
            "id": "",
            "pw": "",
        },
    )
    
    # NID_AUT 쿠키 확인
    cookies = dict(session.cookies)
    if "NID_AUT" in cookies:
        log("RSA 로그인 성공", "ok")
        return cookies
    
    raise RuntimeError("RSA 로그인 실패 — 추가 인증 필요 가능성")
```

### 세션 유효성 검증

```python
def is_naver_session_valid(cookies: dict) -> bool:
    """네이버 로그인 세션 유효성 확인"""
    if "NID_AUT" not in cookies:
        return False
    
    session = requests.Session()
    session.cookies.update(cookies)
    
    resp = session.get(
        "https://www.naver.com",
        allow_redirects=True,
        timeout=10,
    )
    
    # 로그인 마커 확인
    markers = ["로그아웃", "MY 영역", "gnb_my", "nid_my"]
    return any(marker in resp.text for marker in markers)
```

### 네이버 블로그 SE 에디터 API 발행

```python
# publishers/naver_blog.py
BLOG_API = "https://blog.naver.com/BlogApi.naver"

class NaverBlogPublisher:
    def __init__(self, cookies: dict):
        self.session = requests.Session()
        self.session.cookies.update(cookies)
        self.session.headers.update({
            "Referer": "https://blog.naver.com",
            "User-Agent": "Mozilla/5.0 Chrome/124.0.0.0",
        })
    
    def publish(self, post: BlogPost) -> str:
        """SE 에디터 내부 API로 블로그 발행"""
        
        # SE 에디터 JSON 포맷으로 변환
        se_content = self._to_se_format(post.content)
        
        payload = {
            "blogId": BLOG_ID,
            "title": post.title,
            "contents": se_content,
            "categoryNo": "0",
            "openType": "1",  # 전체 공개
            "publishType": "now",
            "tagList": ",".join(post.tags[:5]),
        }
        
        resp = self.session.post(
            f"{BLOG_API}?blogId={BLOG_ID}&action=writePost",
            json=payload,
            timeout=30,
        )
        resp.raise_for_status()
        
        result = resp.json()
        post_no = result["logNo"]
        return f"https://blog.naver.com/{BLOG_ID}/{post_no}"
    
    def _to_se_format(self, markdown_content: str) -> str:
        """Markdown → SE 에디터 JSON 변환"""
        # SE 에디터는 자체 JSON 포맷을 사용
        paragraphs = markdown_content.split("\n\n")
        components = []
        
        for para in paragraphs:
            if para.startswith("## "):
                components.append({
                    "type": "heading2",
                    "text": para[3:],
                })
            elif para.startswith("### "):
                components.append({
                    "type": "heading3", 
                    "text": para[4:],
                })
            else:
                components.append({
                    "type": "paragraph",
                    "text": para,
                })
        
        return json.dumps({"components": components})
```

---

## 실패 사례 & 해결책

**실패 1: RSA 로그인 후 추가 보안 인증 요구**

새로운 IP나 새 기기에서 로그인하면 네이버가 "이 기기를 기억하시겠습니까?" 팝업을 냅니다. 자동화 스크립트는 이를 처리하지 못합니다.

→ **해결**: CDP 방식으로 기존 인증된 Chrome 프로필 쿠키를 재사용. RSA 방식은 폴백으로만 사용.

**실패 2: SE 에디터 API 구조 변경**

네이버는 SE 에디터를 지속적으로 업데이트합니다. API 엔드포인트나 파라미터가 바뀌면 발행이 실패합니다.

→ **해결**: 발행 실패 감지 시 텔레그램 알림. Chrome 개발자 도구로 SE 에디터 요청을 다시 캡처해 업데이트.

---

## 배운 점 / 주의사항

**네이버는 로그인 제한이 강합니다.** 동일 IP에서 단시간에 많은 로그인 시도를 하면 일시적으로 차단됩니다. 세션을 최대한 오래 유지하세요.

**카페 발행은 별도 API가 필요합니다.** 블로그와 카페는 API 구조가 다릅니다. 카페는 커뮤니티 성격이 강해서 자동 발행 시 운영 정책 위반 가능성을 반드시 확인하세요.

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
8. **[현재 글] [Auto-Publishing] 네이버 RSA 암호화 로그인으로 블로그·카페 자동 발행**
9. [[09] GitHub Pages 자동 발행 — Jekyll Markdown 자동 커밋·푸시](/posts/auto-publishing-09/)
10. [[10] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest](/posts/auto-publishing-10/)
11. [[11] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행](/posts/auto-publishing-11/)
12. [[12] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기](/posts/auto-publishing-12/)
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T01~T08] 전체 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
