---
title: "[Auto-Publishing] GitHub Pages 자동 발행 — Jekyll Markdown 자동 커밋·푸시"
date: 2026-04-09 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, GitHubPages, Jekyll자동화, git자동화]
description: "GitHub Pages + Jekyll 블로그에 Python으로 Markdown 파일을 자동 생성하고 커밋·푸시하는 파이프라인을 공개합니다. GitHub Actions와 연동해 자동 빌드·배포까지 완전 자동화합니다."
---

## 개요

이 블로그 자체가 바로 그 결과물입니다. GitHub Pages + Jekyll(Chirpy 테마)로 운영되는 이 블로그에 Python 스크립트로 자동 발행을 구현했습니다.

다른 플랫폼(티스토리, 네이버)이 브라우저 자동화가 필요한 것과 달리, GitHub Pages는 **git만 알면** 완전 자동화가 가능합니다.

---

## 왜 GitHub Pages를 포함했나

**장점:**
- 광고 없이 완전한 소유권 보유 (내 도메인, 내 코드)
- 기술 검색 유입에 강함 (코드 블록, 마크다운 렌더링)
- GitHub Actions로 빌드·배포 자동화
- 무료 호스팅

**단점:**
- 네이버 검색에서 노출이 약함 (해외 도메인)
- 글 발행 후 구글 색인까지 시간이 걸림 → [S03편](/posts/auto-publishing-s03/)에서 해결

---

## 핵심 구현

### Markdown 파일 자동 생성

Jekyll은 `_posts/` 디렉토리의 `YYYY-MM-DD-제목.md` 파일을 자동으로 포스팅합니다.

```python
# publishers/github_pages.py
from pathlib import Path
from datetime import datetime
import subprocess
import re

REPO_PATH = Path("/home/ubuntu/my-blog")
POSTS_DIR = REPO_PATH / "_posts"

def slugify(title: str) -> str:
    """제목 → URL 친화적 slug 변환"""
    # 한글 제거, 영문·숫자·하이픈만 허용
    slug = re.sub(r"[^\w\s-]", "", title.lower())
    slug = re.sub(r"[\s_]+", "-", slug)
    slug = slug.strip("-")
    # 빈 slug면 타임스탬프 사용
    return slug or f"post-{int(datetime.now().timestamp())}"

class GitHubPagesPublisher:
    def __init__(self, repo_path: Path = REPO_PATH):
        self.repo_path = repo_path
        self.posts_dir = repo_path / "_posts"
        self.posts_dir.mkdir(parents=True, exist_ok=True)
    
    def publish(self, post: BlogPost, product: Product) -> str:
        """Markdown 파일 생성 → git commit → push"""
        
        # 파일명 생성
        date_str = datetime.now().strftime("%Y-%m-%d")
        slug = slugify(post.title)
        filename = f"{date_str}-{slug[:50]}.md"
        filepath = self.posts_dir / filename
        
        # Front matter + 본문
        content = self._build_jekyll_post(post, product)
        filepath.write_text(content, encoding="utf-8")
        
        log(f"Jekyll 포스트 생성: {filename}", "ok")
        
        # git 커밋 & 푸시
        self._git_commit_push(filename, post.title)
        
        # 최종 URL 반환
        return f"https://myblog.github.io/posts/{slug}/"
    
    def _build_jekyll_post(self, post: BlogPost, product: Product) -> str:
        """Jekyll front matter + Markdown 본문 조합"""
        tags_str = "\n".join(f"  - {t}" for t in post.tags)
        
        front_matter = f"""---
title: "{post.title}"
date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')} +0900
categories: [리뷰, 상품추천]
tags:
{tags_str}
description: "{post.description}"
---

"""
        
        # 제휴 링크 버튼 추가
        affiliate_btn = f"""
---

> **[👉 쿠팡에서 최저가 확인하기]({product.affiliate_url})**{{: .btn .btn-primary target="_blank"}}

"""
        
        return front_matter + post.content + affiliate_btn
    
    def _git_commit_push(self, filename: str, title: str):
        """git add → commit → push"""
        def run(cmd: list[str]):
            result = subprocess.run(
                cmd,
                cwd=str(self.repo_path),
                capture_output=True,
                text=True,
            )
            if result.returncode != 0:
                raise RuntimeError(f"git 실패: {result.stderr}")
            return result.stdout
        
        # 스테이징
        run(["git", "add", f"_posts/{filename}"])
        
        # 커밋
        commit_msg = f"feat: 자동 발행 — {title[:60]}"
        run(["git", "commit", "-m", commit_msg])
        
        # 푸시
        run(["git", "push", "origin", "main"])
        log("GitHub Pages 자동 커밋·푸시 완료", "ok")
```

### SSH 키 설정 (비밀번호 없이 push)

서버에서 git push가 자동으로 되려면 GitHub에 SSH 키가 등록되어 있어야 합니다.

```bash
# 서버에서 SSH 키 생성
ssh-keygen -t ed25519 -C "auto-publish" -f ~/.ssh/github_autopub -N ""

# 공개키 출력 → GitHub Settings > SSH Keys에 추가
cat ~/.ssh/github_autopub.pub

# ~/.ssh/config에 설정 추가
echo "
Host github.com
  IdentityFile ~/.ssh/github_autopub
  StrictHostKeyChecking no
" >> ~/.ssh/config

# 저장소 원격 URL을 SSH로 변경
git remote set-url origin git@github.com:username/blog.git
```

### GitHub Actions 빌드 자동화

`.github/workflows/pages.yml`이 있으면 push 시 자동으로 Jekyll 빌드 + GitHub Pages 배포가 됩니다. Chirpy 테마는 이 파일이 기본 포함되어 있습니다.

```yaml
# .github/workflows/pages.yml (이미 존재하는 파일)
name: "Build and Deploy"
on:
  push:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build with Jekyll
        uses: actions/jekyll-build-pages@v1
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

---

## 실패 사례 & 해결책

**실패 1: 한글 제목으로 생성된 파일명이 URL로 동작 안 함**

`2026-05-19-무선이어폰-추천.md`는 URL에서 인코딩 문제가 발생합니다.

→ **해결**: `slugify()` 함수로 영문 slug만 파일명에 사용. 제목은 front matter `title:`에 한글로 유지.

**실패 2: 동시에 여러 파이프라인이 push하면 충돌**

멀티 파이프라인이 동시에 같은 저장소에 push하면 git 충돌이 발생합니다.

→ **해결**: `threading.Lock()`으로 git 작업을 직렬화. 한 번에 하나의 파이프라인만 push.

```python
import threading

_git_lock = threading.Lock()

def _git_commit_push(self, filename: str, title: str):
    with _git_lock:  # 동시 push 방지
        # ... git 작업
```

---

## 배운 점 / 주의사항

**GitHub Pages 빌드는 평균 1~3분이 걸립니다.** 발행 후 바로 URL을 확인하면 아직 반영되지 않습니다. 색인은 더 오래 걸립니다 → [S02편](/posts/auto-publishing-s02/)에서 색인 가속 방법을 다룹니다.

**front matter의 날짜는 미래 날짜로 설정하면 안 됩니다.** Jekyll은 미래 날짜 포스트를 기본적으로 숨깁니다. `future: true`를 `_config.yml`에 추가하거나, 오늘 날짜를 사용하세요.

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
9. **[현재 글] [Auto-Publishing] GitHub Pages 자동 발행 — Jekyll Markdown 자동 커밋·푸시**
10. [[10] SNS 4개 동시 자동화 — Twitter·Threads·Instagram·Pinterest](/posts/auto-publishing-10/)
11. [[11] 뉴스픽·정책브리핑 RSS로 정보성 콘텐츠 자동 수집·발행](/posts/auto-publishing-11/)
12. [[12] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기](/posts/auto-publishing-12/)
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T01~T08] 전체 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
