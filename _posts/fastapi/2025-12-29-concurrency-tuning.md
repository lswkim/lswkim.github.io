---
title: "[FastAPI 실전심화 3] 동시 요청 처리와 성능 튜닝"
date: 2025-12-29 12:00:00 +0900
categories: [Tech, FastAPI]
tags: [python, fastapi, concurrency, performance, tuning]
mermaid: true
---

> **📚 FastAPI 시리즈 - Part 5. 실전 심화**
>
> 1. [동기 함수 vs 비동기 함수 선택 기준](/posts/sync-async-choice/)
> 2. [BackgroundTasks와 작업 큐](/posts/background-tasks/)
> 3. 동시 요청 처리와 성능 튜닝 ← 현재 글
> 4. [FastAPI 예외처리](/posts/exception-handling/)
> 5. [프로젝트 구조 설계](/posts/project-structure/)
> 6. [Python 객체/리소스 관리 패턴](/posts/resource-management/)

---

# 3. 동시 요청 처리와 성능 튜닝

## 왜 이 개념이 중요한가?

- 프로덕션 환경에서 안정적인 서비스 운영
- 리소스를 효율적으로 사용
- 병목 구간 파악 및 최적화

---

## FastAPI 동시성 모델 복습

```mermaid
flowchart TB
    subgraph Worker["Uvicorn Worker (프로세스)"]
        subgraph EventLoop["이벤트 루프 (메인 스레드)"]
            EL1[async def 엔드포인트들]
            EL2[수천 개 동시 처리 가능<br/>I/O 바운드]
        end

        subgraph ThreadPool["스레드풀 (기본 40개)"]
            TP1[def 엔드포인트들]
            TP2[동기 블로킹 작업 처리]
        end
    end
```

---

## 워커 수 설정

### 기본 공식

| 환경 | 워커 수 권장 |
| --- | --- |
| **I/O 바운드** (대부분의 API 서버) | 1 ~ CPU 코어 수 (비동기로 이미 동시 처리 가능) |
| **CPU 바운드 혼합** | CPU 코어 수 |
| **Kubernetes 환경** | 1 (Pod 수로 스케일링) |

### 설정 방법

```bash
# Uvicorn 단독
uvicorn app:app --workers 4

# Gunicorn + Uvicorn
gunicorn app:app -w 4 -k uvicorn.workers.UvicornWorker

```

### 워커 수에 따른 동시 처리

4코어 서버, async def 엔드포인트 (I/O 바운드) 기준:

| 워커 수 | 동시 처리 능력 |
| --- | --- |
| 워커 1개 | 수천 개 동시 연결 처리 가능 |
| 워커 4개 | 수천 × 4 = 더 많은 동시 연결 |

> I/O 바운드면 워커 1~2개로도 충분한 경우가 많다. 워커를 늘리면 메모리 사용량도 증가한다 (프로세스당).

---

## 스레드풀 설정

### 기본 스레드풀 크기

```python
# Uvicorn 기본: 40개
# def 엔드포인트 동시 처리 수 = 스레드풀 크기

```

### 스레드풀 크기 조정

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

@app.on_event("startup")
async def startup():
    loop = asyncio.get_event_loop()
    # 스레드풀 크기를 100으로 변경
    loop.set_default_executor(ThreadPoolExecutor(max_workers=100))

```

### 언제 스레드풀을 늘려야 하나?

| 상황 | 조정 |
| --- | --- |
| def 엔드포인트가 많고, 동시 요청 40개 이상 자주 발생, 응답 시간 느려짐 | **늘려야 함** |
| 메모리 부족하거나, 대부분 async def 사용 | **줄여야 함** |

> **주의:** 스레드가 많다고 무조건 좋은 것이 아니다. 컨텍스트 스위칭 오버헤드 증가하고, 스레드당 메모리 사용 (약 8MB 스택).

---

## 연결 제한 설정

### Uvicorn 설정

```bash
uvicorn app:app \
    --limit-concurrency 1000 \    # 최대 동시 연결
    --limit-max-requests 10000 \  # 워커당 최대 요청 후 재시작
    --timeout-keep-alive 5        # Keep-Alive 타임아웃

```

### Gunicorn 설정

```python
# gunicorn.conf.py
workers = 4
worker_class = "uvicorn.workers.UvicornWorker"

# 워커 관리
max_requests = 1000           # 요청 후 워커 재시작 (메모리 누수 방지)
max_requests_jitter = 50      # 동시 재시작 방지
timeout = 30                  # 요청 타임아웃
graceful_timeout = 30         # 종료 대기 시간
keepalive = 5                 # Keep-Alive 타임아웃

```

---

## 벤치마킹

### 도구 선택

| 도구 | 특징 | 사용 |
| --- | --- | --- |
| **wrk** | 빠름, 간단 | 기본 부하 테스트 |
| **Locust** | Python, 시나리오 작성 | 복잡한 시나리오 |
| **k6** | JavaScript, 현대적 | CI/CD 통합 |
| **ab** | Apache 기본 | 간단한 테스트 |
| **hey** | Go 기반, 빠름 | 빠른 테스트 |

### wrk 기본 사용

```bash
# 설치 (Ubuntu)
apt-get install wrk

# 기본 테스트: 12스레드, 400연결, 30초
wrk -t12 -c400 -d30s http://localhost:8000/api/endpoint

# 결과 예시
# Requests/sec: 15234.12
# Latency Avg: 26.12ms
# Latency 99%: 123.45ms

```

### Locust 사용

```python
# locustfile.py
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(1, 3)  # 요청 간 대기 시간

    @task(3)  # 가중치 3
    def get_items(self):
        self.client.get("/items")

    @task(1)  # 가중치 1
    def create_item(self):
        self.client.post("/items", json={"name": "test"})

    @task(2)
    def get_item(self):
        self.client.get("/items/1")

```

```bash
# 실행
locust -f locustfile.py --host=http://localhost:8000

# 웹 UI: http://localhost:8089

```

### k6 사용

```javascript
// load_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 100 },  // 램프업
        { duration: '1m', target: 100 },   // 유지
        { duration: '30s', target: 0 },    // 램프다운
    ],
};

export default function () {
    const res = http.get('http://localhost:8000/items');
    check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 200ms': (r) => r.timings.duration < 200,
    });
    sleep(1);
}

```

```bash
# 실행
k6 run load_test.js

```

---

## 성능 지표 (Metrics)

### 핵심 지표

| 지표 | 설명 | 목표 (예시) |
| --- | --- | --- |
| **RPS** | 초당 요청 수 | 높을수록 좋음 |
| **Latency (Avg)** | 평균 응답 시간 | < 100ms |
| **Latency (p99)** | 99% 응답 시간 | < 500ms |
| **Error Rate** | 에러 비율 | < 0.1% |
| **Throughput** | 처리량 | 요구사항 충족 |

### 지표 해석

| 상태 | 지표 예시 |
| --- | --- |
| **좋은 상태** | Avg Latency: 50ms, p99 Latency: 150ms, Error Rate: 0% |
| **문제 징후** | p99가 Avg보다 10배 이상 높음 → 가끔 느린 요청 있음, 부하 증가 시 Latency 급증 → 병목 있음, Error Rate 증가 → 리소스 부족 또는 버그 |

---

## 병목 구간 파악

### 일반적인 병목 구간

```mermaid
flowchart TB
    subgraph DB["1. 데이터베이스"]
        DB1[슬로우 쿼리]
        DB2[연결 풀 부족]
        DB3[인덱스 누락]
    end

    subgraph API["2. 외부 API"]
        API1[느린 응답]
        API2[타임아웃 미설정]
        API3[동기 호출]
    end

    subgraph App["3. 애플리케이션"]
        App1[async def + 블로킹 호출]
        App2[CPU 바운드 작업]
        App3[메모리 누수]
    end

    subgraph Infra["4. 인프라"]
        Infra1[워커 부족]
        Infra2[메모리 부족]
        Infra3[네트워크 대역폭]
    end
```

### 프로파일링 도구

```python
# 1. cProfile (기본)
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()
# 코드 실행
profiler.disable()
stats = pstats.Stats(profiler).sort_stats('cumulative')
stats.print_stats(10)

# 2. py-spy (프로덕션 프로파일링)
# pip install py-spy
# py-spy top --pid <PID>
# py-spy record -o profile.svg --pid <PID>

# 3. line_profiler (라인별 분석)
# pip install line_profiler
@profile
def slow_function():
    ...

```

### 요청별 타이밍 측정

```python
import time
import logging

logger = logging.getLogger(__name__)

@app.middleware("http")
async def timing_middleware(request, call_next):
    start = time.perf_counter()

    response = await call_next(request)

    duration = time.perf_counter() - start

    # 느린 요청 로깅
    if duration > 1.0:  # 1초 이상
        logger.warning(f"Slow request: {request.url.path} took {duration:.2f}s")

    response.headers["X-Response-Time"] = f"{duration:.4f}"
    return response

```

---

## 데이터베이스 최적화

### 연결 풀 설정

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=10,           # 기본 연결 수
    max_overflow=20,        # 추가 허용 연결
    pool_timeout=30,        # 연결 대기 타임아웃
    pool_recycle=1800,      # 연결 재생성 주기 (초)
    pool_pre_ping=True,     # 연결 유효성 체크
)

```

### 비동기 DB (asyncpg)

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=10,
    max_overflow=20,
)

```

### 연결 풀 크기 계산

**공식 (기본):**
```
pool_size = (워커 수 × 2) + max_overflow
```

**예시 (워커 4개):**
- pool_size = 10
- max_overflow = 20
- 최대 연결 = 10 + 20 = 30

> **주의:** DB 서버의 max_connections 확인하고, 여러 앱이 같은 DB 사용 시 분배를 고려한다.

---

## 외부 API 호출 최적화

### 연결 재사용

```python
import httpx

# ❌ 매 요청마다 연결 생성
@app.get("/external")
async def bad_external():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com")
    return response.json()

# ✅ 연결 재사용 (앱 수명 동안)
http_client = None

@app.on_event("startup")
async def startup():
    global http_client
    http_client = httpx.AsyncClient(
        timeout=10.0,
        limits=httpx.Limits(max_connections=100)
    )

@app.on_event("shutdown")
async def shutdown():
    await http_client.aclose()

@app.get("/external")
async def good_external():
    response = await http_client.get("https://api.example.com")
    return response.json()

```

### 타임아웃 설정

```python
http_client = httpx.AsyncClient(
    timeout=httpx.Timeout(
        connect=5.0,    # 연결 타임아웃
        read=10.0,      # 읽기 타임아웃
        write=10.0,     # 쓰기 타임아웃
        pool=5.0,       # 풀에서 연결 대기 타임아웃
    )
)

```

---

## 캐싱

### Redis 캐싱

```python
import redis.asyncio as redis
import json

redis_client = redis.Redis(host='localhost', port=6379, db=0)

async def get_cached_or_fetch(key: str, fetch_func, ttl: int = 300):
    # 캐시 확인
    cached = await redis_client.get(key)
    if cached:
        return json.loads(cached)

    # 캐시 미스: 데이터 가져오기
    data = await fetch_func()

    # 캐시 저장
    await redis_client.setex(key, ttl, json.dumps(data))

    return data

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return await get_cached_or_fetch(
        f"user:{user_id}",
        lambda: db.get_user(user_id),
        ttl=300
    )

```

### 인메모리 캐싱 (간단한 경우)

```python
from functools import lru_cache
from cachetools import TTLCache
import asyncio

# 동기 함수용
@lru_cache(maxsize=100)
def get_settings():
    return load_settings()

# 비동기용 TTL 캐시
cache = TTLCache(maxsize=100, ttl=300)

async def get_cached_data(key: str):
    if key in cache:
        return cache[key]

    data = await fetch_data(key)
    cache[key] = data
    return data

```

---

## 최적화 체크리스트

### 배포 전 체크리스트

- [ ] async def 안에서 블로킹 호출 없는지 확인
- [ ] **데이터베이스**
  - [ ] 연결 풀 설정
  - [ ] 슬로우 쿼리 확인
  - [ ] 인덱스 확인
- [ ] **외부 API**
  - [ ] 타임아웃 설정
  - [ ] 연결 재사용
  - [ ] 재시도 로직
- [ ] **캐싱**
  - [ ] 자주 조회되는 데이터 캐싱
  - [ ] TTL 설정
- [ ] **서버 설정**
  - [ ] 워커 수 적절한지
  - [ ] 스레드풀 크기 적절한지
  - [ ] 타임아웃 설정
- [ ] **모니터링**
  - [ ] 응답 시간 로깅
  - [ ] 에러 추적
  - [ ] 리소스 모니터링

### 환경별 권장 설정

| 환경 | 워커 | 스레드풀 | 특이사항 |
| --- | --- | --- | --- |
| 개발 | 1 | 기본(40) | `--reload` 사용 |
| 소규모 프로덕션 | 2~4 | 40~100 | Uvicorn 단독 |
| 대규모 프로덕션 | CPU 수 | 상황에 맞게 | Gunicorn + Uvicorn |
| Kubernetes | 1 | 40 | Pod 수로 스케일링 |

---

## 핵심 정리

| 항목 | 권장 |
| --- | --- |
| **워커 수** | I/O 바운드: 1~CPU 수, K8s: 1 |
| **스레드풀** | 기본 40, 필요 시 조정 |
| **DB 연결 풀** | pool_size=10, max_overflow=20 |
| **외부 API** | 연결 재사용, 타임아웃 필수 |
| **캐싱** | 자주 조회 데이터는 Redis |

### 성능 튜닝 순서

1. 벤치마킹으로 현재 상태 파악
2. 병목 구간 식별 (DB? API? 앱?)
3. 해당 구간 최적화
4. 다시 벤치마킹
5. 반복

### 가장 흔한 병목

| 순위 | 원인 |
| --- | --- |
| 1위 | 데이터베이스 (슬로우 쿼리, 연결 풀 부족) |
| 2위 | async def + 블로킹 호출 |
| 3위 | 외부 API (느린 응답, 타임아웃 없음) |
| 4위 | 캐싱 없음 (매번 같은 데이터 조회) |
