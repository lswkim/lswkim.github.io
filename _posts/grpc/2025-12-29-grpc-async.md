---
title: "[gRPC 실전/ML서빙 2] 비동기 gRPC (asyncio)"
date: 2025-12-29 01:00:00 +0900
categories: [Tech, gRPC]
tags: [grpc, python, asyncio, async, concurrent]
mermaid: true
---

> **📚 gRPC 시리즈 - Part 3. 실전 구현과 ML 서빙 적용**
>
> 1. [Python gRPC 서버/클라이언트 구현](/posts/grpc-python-impl/)
> 2. 비동기 gRPC (asyncio) ← 현재 글
> 3. [Triton Inference Server gRPC](/posts/grpc-triton/)
> 4. [vLLM / KServe gRPC 연동](/posts/grpc-vllm-kserve/)

---

# 2. 비동기 gRPC (asyncio)

## 왜 비동기 gRPC를 알아야 하는가?

동기 gRPC는 ThreadPoolExecutor 기반이다. I/O 바운드 작업이 많으면 비효율적일 수 있다.

- DB 쿼리, 외부 API 호출이 많은 서비스 → 비동기가 효율적
- FastAPI처럼 `async def`로 작성하고 싶다 → `grpc.aio` 사용
- 높은 동시성이 필요하다 → 비동기가 유리

---

## 동기 vs 비동기 비교

### 동기 gRPC (기본)

```python
import grpc
from concurrent import futures

class UserServiceServicer(user_pb2_grpc.UserServiceServicer):

    def GetUser(self, request, context):  # 일반 함수
        user = db.get_user(request.id)    # 블로킹
        return user_pb2.GetUserResponse(user=user)

# ThreadPoolExecutor 사용
server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))

```

- 요청마다 스레드 할당
- 블로킹 I/O 시 스레드가 대기
- 동시 요청 수 = max_workers 제한

### 비동기 gRPC (grpc.aio)

```python
import grpc.aio

class UserServiceServicer(user_pb2_grpc.UserServiceServicer):

    async def GetUser(self, request, context):  # async 함수
        user = await db.get_user(request.id)    # 논블로킹
        return user_pb2.GetUserResponse(user=user)

# asyncio 이벤트 루프 사용
server = grpc.aio.server()

```

- 단일 스레드, 이벤트 루프 기반
- I/O 대기 중 다른 요청 처리 가능
- 높은 동시성 처리 가능

---

## 언제 비동기를 쓸까?

| 상황 | 추천 |
| --- | --- |
| DB 쿼리가 많음 | ✅ 비동기 |
| 외부 API 호출이 많음 | ✅ 비동기 |
| CPU 연산이 많음 | ❌ 동기 (멀티프로세스) |
| 단순 인메모리 작업 | 둘 다 OK |
| 기존 동기 라이브러리 사용 | ❌ 동기 |

---

## 비동기 서버 구현

### 기본 구조

```python
import grpc.aio
import asyncio
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from generated import user_pb2
from generated import user_pb2_grpc

class UserServiceServicer(user_pb2_grpc.UserServiceServicer):
    """비동기 UserService 구현체"""

    def __init__(self):
        self.users = {}
        self.next_id = 1
        self._add_sample_data()

    def _add_sample_data(self):
        samples = [
            ("홍길동", "hong@example.com"),
            ("김철수", "kim@example.com"),
        ]
        for name, email in samples:
            user = user_pb2.User(
                id=self.next_id,
                name=name,
                email=email,
                status=user_pb2.USER_STATUS_ACTIVE
            )
            self.users[self.next_id] = user
            self.next_id += 1

    async def GetUser(self, request, context):
        """비동기 유저 조회"""
        print(f"[GetUser] id={request.id}")

        # 비동기 I/O 시뮬레이션 (실제로는 async DB 호출)
        await asyncio.sleep(0.01)

        user = self.users.get(request.id)

        if user is None:
            await context.abort(
                grpc.StatusCode.NOT_FOUND,
                f"User {request.id} not found"
            )

        return user_pb2.GetUserResponse(user=user)

    async def ListUsers(self, request, context):
        """비동기 유저 목록 조회"""
        print(f"[ListUsers] page_size={request.page_size}")

        await asyncio.sleep(0.01)

        users = list(self.users.values())
        if request.page_size > 0:
            users = users[:request.page_size]

        return user_pb2.ListUsersResponse(users=users)

    async def CreateUser(self, request, context):
        """비동기 유저 생성"""
        print(f"[CreateUser] name={request.name}")

        await asyncio.sleep(0.01)

        if not request.name or not request.email:
            await context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                "Name and email are required"
            )

        user = user_pb2.User(
            id=self.next_id,
            name=request.name,
            email=request.email,
            status=user_pb2.USER_STATUS_ACTIVE
        )

        self.users[self.next_id] = user
        self.next_id += 1

        return user_pb2.CreateUserResponse(user=user)

async def serve():
    """비동기 서버 실행"""

    # 비동기 서버 생성
    server = grpc.aio.server(
        options=[
            ('grpc.max_send_message_length', 50 * 1024 * 1024),
            ('grpc.max_receive_message_length', 50 * 1024 * 1024),
        ]
    )

    # 서비스 등록
    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(),
        server
    )

    # 포트 바인딩
    port = 50051
    server.add_insecure_port(f'[::]:{port}')

    # 서버 시작
    await server.start()
    print(f"Async server started on port {port}")

    # 종료 대기
    try:
        await server.wait_for_termination()
    except asyncio.CancelledError:
        print("Shutting down...")
        await server.stop(grace=5)

if __name__ == '__main__':
    asyncio.run(serve())

```

---

## 비동기 클라이언트 구현

### 기본 구조

```python
import grpc.aio
import asyncio
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from generated import user_pb2
from generated import user_pb2_grpc

class AsyncUserClient:
    """비동기 UserService 클라이언트"""

    def __init__(self, host='localhost:50051'):
        self.host = host
        self.channel = None
        self.stub = None

    async def connect(self):
        """연결"""
        self.channel = grpc.aio.insecure_channel(self.host)
        self.stub = user_pb2_grpc.UserServiceStub(self.channel)

    async def close(self):
        """연결 종료"""
        if self.channel:
            await self.channel.close()

    async def get_user(self, user_id):
        """유저 조회"""
        try:
            response = await self.stub.GetUser(
                user_pb2.GetUserRequest(id=user_id),
                timeout=5.0
            )
            return response.user
        except grpc.aio.AioRpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
            return None

    async def list_users(self, page_size=10):
        """유저 목록 조회"""
        try:
            response = await self.stub.ListUsers(
                user_pb2.ListUsersRequest(page_size=page_size),
                timeout=5.0
            )
            return list(response.users)
        except grpc.aio.AioRpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
            return []

    async def create_user(self, name, email):
        """유저 생성"""
        try:
            response = await self.stub.CreateUser(
                user_pb2.CreateUserRequest(name=name, email=email),
                timeout=5.0
            )
            return response.user
        except grpc.aio.AioRpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
            return None

async def main():
    client = AsyncUserClient()
    await client.connect()

    print("=" * 50)
    print("1. 유저 목록 조회")
    print("=" * 50)
    users = await client.list_users()
    for user in users:
        print(f"  - {user.id}: {user.name} ({user.email})")

    print("\n" + "=" * 50)
    print("2. 단일 유저 조회")
    print("=" * 50)
    user = await client.get_user(1)
    if user:
        print(f"  - {user.id}: {user.name} ({user.email})")

    print("\n" + "=" * 50)
    print("3. 유저 생성")
    print("=" * 50)
    new_user = await client.create_user("박지민", "park@example.com")
    if new_user:
        print(f"  Created: {new_user.id}: {new_user.name}")

    await client.close()

if __name__ == '__main__':
    asyncio.run(main())

```

---

## 동시 요청 처리

### 비동기의 진짜 장점

```python
async def fetch_multiple_users(client, user_ids):
    """여러 유저를 동시에 조회"""

    # 동시에 요청 (병렬 실행)
    tasks = [client.get_user(uid) for uid in user_ids]
    results = await asyncio.gather(*tasks)

    return results

async def main():
    client = AsyncUserClient()
    await client.connect()

    # 10명의 유저를 동시에 조회
    user_ids = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    import time

    # 순차 실행
    start = time.time()
    for uid in user_ids:
        await client.get_user(uid)
    print(f"순차 실행: {time.time() - start:.2f}초")

    # 동시 실행
    start = time.time()
    await fetch_multiple_users(client, user_ids)
    print(f"동시 실행: {time.time() - start:.2f}초")

    await client.close()

```

```
출력 (예시):
순차 실행: 0.12초
동시 실행: 0.02초

```

---

## 비동기 스트리밍

### Server Streaming (비동기)

```python
# 서버
class UserServiceServicer(user_pb2_grpc.UserServiceServicer):

    async def ListUsersStream(self, request, context):
        """유저 목록 스트리밍"""
        for user_id, user in self.users.items():
            await asyncio.sleep(0.1)  # 시뮬레이션
            yield user

# 클라이언트
async def stream_users(client):
    async for user in client.stub.ListUsersStream(
        user_pb2.ListUsersRequest()
    ):
        print(f"Received: {user.name}")

```

### Bidirectional Streaming (비동기)

```python
# 서버
class ChatServiceServicer(chat_pb2_grpc.ChatServiceServicer):

    async def Chat(self, request_iterator, context):
        async for message in request_iterator:
            print(f"Received: {message.content}")

            # 응답 생성
            response = f"Echo: {message.content}"
            yield chat_pb2.ChatMessage(content=response)

# 클라이언트
async def chat(stub):
    async def generate_messages():
        messages = ["안녕", "반가워", "잘가"]
        for msg in messages:
            yield chat_pb2.ChatMessage(content=msg)
            await asyncio.sleep(0.5)

    async for response in stub.Chat(generate_messages()):
        print(f"Server: {response.content}")

```

---

## Context Manager 패턴

### 권장하는 사용 방식

```python
class AsyncUserClient:

    async def __aenter__(self):
        await self.connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.close()

# 사용
async def main():
    async with AsyncUserClient() as client:
        users = await client.list_users()
        for user in users:
            print(user.name)
    # 자동으로 close됨

```

---

## 동기 vs 비동기 성능 비교

### 테스트 시나리오

```python
"""
100개의 동시 요청, 각 요청당 10ms I/O 대기
"""

# 동기 서버 (max_workers=10)
# → 10개씩 처리, 총 10번 = 약 100ms

# 비동기 서버
# → 100개 동시 처리 = 약 10ms

```

### 실제 벤치마크 (예시)

| 시나리오 | 동기 (10 workers) | 비동기 |
| --- | --- | --- |
| 100 동시 요청 (I/O 10ms) | ~100ms | ~15ms |
| 1000 동시 요청 (I/O 10ms) | ~1000ms | ~50ms |
| 100 동시 요청 (CPU 10ms) | ~100ms | ~1000ms |

**결론:**

- I/O 바운드 → 비동기가 유리
- CPU 바운드 → 동기(멀티프로세스)가 유리

---

## 주의사항

### 1. 동기 라이브러리 사용 금지

```python
# ❌ 잘못된 예
async def GetUser(self, request, context):
    user = requests.get(f"http://api/users/{request.id}")  # 블로킹!
    return user_pb2.GetUserResponse(user=user)

# ✅ 올바른 예
async def GetUser(self, request, context):
    async with aiohttp.ClientSession() as session:
        async with session.get(f"http://api/users/{request.id}") as resp:
            user = await resp.json()
    return user_pb2.GetUserResponse(user=user)

```

### 2. CPU 작업은 별도 처리

```python
async def GetUser(self, request, context):
    # CPU 작업은 executor로 분리
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(
        None,  # 기본 ThreadPoolExecutor
        cpu_heavy_function,
        request.data
    )
    return user_pb2.GetUserResponse(result=result)

```

### 3. abort 사용법 차이

```python
# 동기
context.abort(grpc.StatusCode.NOT_FOUND, "Not found")

# 비동기
await context.abort(grpc.StatusCode.NOT_FOUND, "Not found")

```

---

## FastAPI와 함께 사용

### gRPC + FastAPI 동시 실행

```python
import asyncio
import grpc.aio
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok"}

async def start_grpc_server():
    server = grpc.aio.server()
    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(),
        server
    )
    server.add_insecure_port('[::]:50051')
    await server.start()
    print("gRPC server started on 50051")
    await server.wait_for_termination()

async def start_rest_server():
    config = uvicorn.Config(app, host="0.0.0.0", port=8000)
    server = uvicorn.Server(config)
    await server.serve()

async def main():
    await asyncio.gather(
        start_grpc_server(),
        start_rest_server(),
    )

if __name__ == '__main__':
    asyncio.run(main())

```

---

## 핵심 정리

### 동기 vs 비동기

|  | 동기 | 비동기 |
| --- | --- | --- |
| **모듈** | `grpc` | `grpc.aio` |
| **함수** | `def` | `async def` |
| **서버** | `grpc.server()` | `grpc.aio.server()` |
| **채널** | `grpc.insecure_channel()` | `grpc.aio.insecure_channel()` |
| **예외** | `grpc.RpcError` | `grpc.aio.AioRpcError` |
| **적합** | CPU 바운드 | I/O 바운드 |

### 핵심 코드

```python
# 비동기 서버
server = grpc.aio.server()
await server.start()
await server.wait_for_termination()

# 비동기 클라이언트
channel = grpc.aio.insecure_channel('localhost:50051')
response = await stub.GetUser(request)

```

### 선택 기준

- DB/외부 API 호출 많음 → 비동기
- CPU 연산 많음 → 동기 + 멀티프로세스
- 기존 동기 라이브러리 의존 → 동기

---