---
title: "[Auto-Publishing] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기"
date: 2026-05-25 09:00:00 +0900
categories: [프로젝트, Auto-Publishing]
tags: [자동화, 파이썬, 블로그자동화, 패시브인컴, Python스케줄러, registry패턴, pkgutil]
description: "pkgutil.iter_modules()를 활용한 Python 파이프라인 자동 발견(auto-discovery) 패턴을 설명합니다. 새 파이프라인 파일을 추가하면 스케줄러가 자동으로 인식하는 확장 가능한 구조입니다."
---

## 개요

파이프라인이 14개를 넘어서자 문제가 생겼습니다. 새 파이프라인을 만들 때마다 스케줄러 파일을 열어서 수동으로 등록해야 했습니다.

10개쯤 되자 스케줄러 파일이 커지고, 등록을 까먹는 실수도 생겼습니다.

**Registry 패턴 + auto-discovery**로 해결했습니다. 파일만 추가하면 자동으로 인식되는 구조입니다.

---

## 핵심 구현

### 파이프라인 인터페이스

모든 파이프라인은 동일한 `PIPELINE` 변수를 모듈 최상단에 선언합니다.

```python
# pipelines/coupang_to_wordpress.py
from dataclasses import dataclass, field
from typing import Callable

@dataclass
class Pipeline:
    name: str
    description: str
    run: Callable[[], None]
    cron: str              # crontab 형식
    enabled: bool = True
    tags: list[str] = field(default_factory=list)

def _run():
    # ... 실제 파이프라인 로직
    pass

PIPELINE = Pipeline(
    name="coupang_to_wordpress",
    description="쿠팡 상품 → AI 글쓰기 → WordPress 발행",
    run=_run,
    cron="0 9 * * *",  # 매일 오전 9시
    tags=["coupang", "wordpress", "product"],
)
```

### auto-discovery 스케줄러

```python
# scheduler.py
import pkgutil
import importlib
import schedule
import time
import pipelines  # pipelines 패키지

def discover_pipelines() -> dict[str, "Pipeline"]:
    """pipelines/ 디렉토리 안의 PIPELINE 변수를 자동으로 수집"""
    found = {}
    
    for importer, modname, ispkg in pkgutil.iter_modules(pipelines.__path__):
        if ispkg:  # 서브패키지는 건너뜀
            continue
        
        try:
            module = importlib.import_module(f"pipelines.{modname}")
        except ImportError as e:
            log(f"파이프라인 임포트 실패 — {modname}: {e}", "warn")
            continue
        
        if not hasattr(module, "PIPELINE"):
            continue
        
        pipeline = module.PIPELINE
        
        if not pipeline.enabled:
            log(f"파이프라인 비활성화 — {pipeline.name}", "info")
            continue
        
        found[pipeline.name] = pipeline
        log(f"파이프라인 등록 — {pipeline.name} ({pipeline.cron})", "ok")
    
    return found

def cron_to_schedule(pipeline: "Pipeline"):
    """cron 문자열을 schedule 라이브러리 설정으로 변환"""
    parts = pipeline.cron.split()
    minute, hour, day, month, weekday = parts
    
    if hour == "*" and minute == "*":
        # 매분
        schedule.every().minute.do(pipeline.run)
    elif minute == "0" and hour != "*":
        # 특정 시간 매일
        schedule.every().day.at(f"{int(hour):02d}:00").do(pipeline.run)
    elif minute != "*":
        # 특정 시:분
        schedule.every().day.at(f"{int(hour):02d}:{int(minute):02d}").do(pipeline.run)

def main():
    log("스케줄러 시작", "ok")
    
    pipelines_map = discover_pipelines()
    log(f"파이프라인 {len(pipelines_map)}개 등록 완료", "ok")
    
    for name, pipeline in pipelines_map.items():
        cron_to_schedule(pipeline)
    
    # 스케줄 루프
    while True:
        schedule.run_pending()
        time.sleep(30)

if __name__ == "__main__":
    main()
```

### 파이프라인 목록 확인 CLI

```python
# scheduler.py (추가)
import argparse

def list_pipelines():
    """등록된 파이프라인 목록 출력"""
    pipelines_map = discover_pipelines()
    
    print(f"\n{'이름':<30} {'스케줄':<15} {'설명'}")
    print("-" * 80)
    for name, p in sorted(pipelines_map.items()):
        print(f"{name:<30} {p.cron:<15} {p.description}")
    print()

def run_once(pipeline_name: str):
    """특정 파이프라인 즉시 1회 실행"""
    pipelines_map = discover_pipelines()
    if pipeline_name not in pipelines_map:
        print(f"파이프라인을 찾을 수 없습니다: {pipeline_name}")
        return
    
    p = pipelines_map[pipeline_name]
    log(f"즉시 실행 — {p.name}", "info")
    p.run()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true")
    parser.add_argument("--run", type=str)
    args = parser.parse_args()
    
    if args.list:
        list_pipelines()
    elif args.run:
        run_once(args.run)
    else:
        main()
```

사용 예시:
```bash
# 파이프라인 목록 확인
python scheduler.py --list

# 특정 파이프라인 즉시 실행
python scheduler.py --run coupang_to_wordpress

# 스케줄러 실행
python scheduler.py
```

### systemd 서비스 등록 (서버 상시 실행)

```ini
# /etc/systemd/system/auto-publish.service
[Unit]
Description=Auto Publishing Scheduler
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/auto-publishing
ExecStart=/home/ubuntu/venv/bin/python scheduler.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable auto-publish
sudo systemctl start auto-publish
sudo systemctl status auto-publish
```

---

## 실패 사례 & 해결책

**실패 1: 파이프라인 예외가 스케줄러를 죽임**

파이프라인 하나에서 예외가 발생하면 전체 스케줄러가 멈췄습니다.

→ **해결**: 각 파이프라인 실행을 try/except로 감싸서 격리.

```python
def safe_run(pipeline: Pipeline):
    try:
        pipeline.run()
    except Exception as e:
        log(f"파이프라인 실패 — {pipeline.name}: {e}", "error")
        notify_telegram(f"⚠️ 파이프라인 실패: {pipeline.name}\n{e}")
```

**실패 2: 동시에 같은 파이프라인이 두 번 실행됨**

이전 실행이 늦어지는 동안 다음 스케줄이 동작해서 중복 실행됐습니다.

→ **해결**: `threading.Lock()`으로 파이프라인당 한 번에 하나만 실행.

---

## 배운 점 / 주의사항

**`pkgutil.iter_modules()`는 파일이 유효한 Python 모듈인지는 확인하지 않습니다.** `__init__.py`가 없는 파일이 있으면 임포트 오류가 납니다. `try/except ImportError`는 필수입니다.

**cron 파싱은 단순화해서 구현했습니다.** 복잡한 cron 표현식(예: `*/5 * * * *`)은 `croniter` 라이브러리를 사용하는 것이 낫습니다.

---

## 시리즈 전체 목차

### 1부 — 핵심 아키텍처 (14편)

1. [[01] AI 자동 발행 시스템 구축기 — 전체 아키텍처 설계](/posts/auto-publishing-01/)
2~11편 목록 생략
12. **[현재 글] [Auto-Publishing] Registry 패턴으로 파이프라인 자동 발견 스케줄러 만들기**
13. [[13] 텔레그램·카카오톡 병행 알림 + OAuth 자동 갱신 구현](/posts/auto-publishing-13/)
14. [[14] 플랫폼별 인증 전략 총정리 — CDP·RSA·HMAC·JWT·Playwright](/posts/auto-publishing-14/)

### 2부 — 트러블슈팅 다이어리 / 3부 — SEO 심화

- [[T01~T08] 전체 트러블슈팅 다이어리](/posts/auto-publishing-t01/)
- [[S01~S04] 백링크와 색인 심화](/posts/auto-publishing-s01/)
