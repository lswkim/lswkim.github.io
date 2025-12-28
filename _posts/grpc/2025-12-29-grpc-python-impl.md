---
title: "[gRPC 실전/ML서빙 1] Python gRPC 서버/클라이언트 구현"
date: 2025-12-29 00:00:00 +0900
categories: [Tech, gRPC]
tags: [grpc, python, server, client, implementation]
mermaid: true
---

> **📚 gRPC 시리즈 - Part 3. 실전 구현과 ML 서빙 적용**
>
> 1. Python gRPC 서버/클라이언트 구현 ← 현재 글
> 2. [비동기 gRPC (asyncio)](/posts/grpc-async/)
> 3. [Triton Inference Server gRPC](/posts/grpc-triton/)
> 4. [vLLM / KServe gRPC 연동](/posts/grpc-vllm-kserve/)

---

# 1. Python gRPC 서버/클라이언트 구현

## 이 문서의 목표

실제로 동작하는 gRPC 서버와 클라이언트를 처음부터 끝까지 만들어본다.

---

## 프로젝트 구조

```
grpc-tutorial/
│
├── protos/
│   └── user.proto              # 서비스 정의
│
├── generated/                  # 자동 생성 코드
│   ├── __init__.py
│   ├── user_pb2.py
│   └── user_pb2_grpc.py
│
├── server/
│   └── main.py                 # gRPC 서버
│
├── client/
│   └── main.py                 # gRPC 클라이언트
│
├── scripts/
│   └── generate.sh             # 코드 생성 스크립트
│
└── requirements.txt

```

---

## Step 1: 환경 설정

### requirements.txt

```
grpcio==1.60.0
grpcio-tools==1.60.0

```

### 설치

```bash
pip install -r requirements.txt

```

---

## Step 2: Proto 파일 작성

### protos/user.proto

```protobuf
syntax = "proto3";

package user;

// ──────────────────────────────────────────────────────────────
// 메시지 정의
// ──────────────────────────────────────────────────────────────

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    UserStatus status = 4;
}

enum UserStatus {
    USER_STATUS_UNSPECIFIED = 0;
    USER_STATUS_ACTIVE = 1;
    USER_STATUS_INACTIVE = 2;
}

// ──────────────────────────────────────────────────────────────
// 요청/응답 메시지
// ──────────────────────────────────────────────────────────────

// GetUser
message GetUserRequest {
    int64 id = 1;
}

message GetUserResponse {
    User user = 1;
}

// ListUsers
message ListUsersRequest {
    int32 page_size = 1;
}

message ListUsersResponse {
    repeated User users = 1;
}

// CreateUser
message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message CreateUserResponse {
    User user = 1;
}

// DeleteUser
message DeleteUserRequest {
    int64 id = 1;
}

message DeleteUserResponse {
    bool success = 1;
}

// ──────────────────────────────────────────────────────────────
// 서비스 정의
// ──────────────────────────────────────────────────────────────

service UserService {
    // 단일 유저 조회
    rpc GetUser(GetUserRequest) returns (GetUserResponse);

    // 유저 목록 조회
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);

    // 유저 생성
    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);

    // 유저 삭제
    rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
}

```

---

## Step 3: 코드 생성

### scripts/generate.sh

```bash
#!/bin/bash

# 프로젝트 루트에서 실행
PROTO_DIR="./protos"
OUT_DIR="./generated"

# 출력 디렉토리 생성
mkdir -p $OUT_DIR

# __init__.py 생성
touch $OUT_DIR/__init__.py

# 코드 생성
python -m grpc_tools.protoc \
    -I$PROTO_DIR \
    --python_out=$OUT_DIR \
    --grpc_python_out=$OUT_DIR \
    $PROTO_DIR/user.proto

echo "Generated:"
ls -la $OUT_DIR

```

### 실행

```bash
chmod +x scripts/generate.sh
./scripts/generate.sh

```

### 생성 결과

```
generated/
├── __init__.py
├── user_pb2.py          # 메시지 클래스 (User, GetUserRequest 등)
└── user_pb2_grpc.py     # 서비스 클래스 (UserServiceStub, UserServiceServicer)

```

---

## Step 4: 서버 구현

### server/main.py

```python
from concurrent import futures
import grpc
import sys
import os

# generated 모듈 import를 위한 경로 추가
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from generated import user_pb2
from generated import user_pb2_grpc

class UserServiceServicer(user_pb2_grpc.UserServiceServicer):
    """UserService 구현체"""

    def __init__(self):
        # 간단한 인메모리 저장소
        self.users = {}
        self.next_id = 1

        # 테스트 데이터
        self._add_sample_data()

    def _add_sample_data(self):
        """샘플 데이터 추가"""
        samples = [
            ("홍길동", "hong@example.com"),
            ("김철수", "kim@example.com"),
            ("이영희", "lee@example.com"),
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

    def GetUser(self, request, context):
        """단일 유저 조회"""
        print(f"[GetUser] id={request.id}")

        user_id = request.id

        # 유효성 검사
        if user_id <= 0:
            context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                "User ID must be positive"
            )

        # 유저 조회
        user = self.users.get(user_id)

        if user is None:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f"User {user_id} not found"
            )

        return user_pb2.GetUserResponse(user=user)

    def ListUsers(self, request, context):
        """유저 목록 조회"""
        print(f"[ListUsers] page_size={request.page_size}")

        users = list(self.users.values())

        # page_size 적용
        if request.page_size > 0:
            users = users[:request.page_size]

        return user_pb2.ListUsersResponse(users=users)

    def CreateUser(self, request, context):
        """유저 생성"""
        print(f"[CreateUser] name={request.name}, email={request.email}")

        # 유효성 검사
        if not request.name:
            context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                "Name is required"
            )

        if not request.email:
            context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                "Email is required"
            )

        # 이메일 중복 체크
        for user in self.users.values():
            if user.email == request.email:
                context.abort(
                    grpc.StatusCode.ALREADY_EXISTS,
                    f"Email {request.email} already exists"
                )

        # 유저 생성
        user = user_pb2.User(
            id=self.next_id,
            name=request.name,
            email=request.email,
            status=user_pb2.USER_STATUS_ACTIVE
        )

        self.users[self.next_id] = user
        self.next_id += 1

        return user_pb2.CreateUserResponse(user=user)

    def DeleteUser(self, request, context):
        """유저 삭제"""
        print(f"[DeleteUser] id={request.id}")

        user_id = request.id

        if user_id not in self.users:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f"User {user_id} not found"
            )

        del self.users[user_id]

        return user_pb2.DeleteUserResponse(success=True)

def serve():
    """서버 실행"""

    # 서버 생성
    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=10),
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
    server.start()
    print(f"Server started on port {port}")
    print("Press Ctrl+C to stop")

    # 종료 대기
    try:
        server.wait_for_termination()
    except KeyboardInterrupt:
        print("\nShutting down...")
        server.stop(grace=5)

if __name__ == '__main__':
    serve()

```

---

## Step 5: 클라이언트 구현

### client/main.py

```python
import grpc
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from generated import user_pb2
from generated import user_pb2_grpc

class UserClient:
    """UserService 클라이언트"""

    def __init__(self, host='localhost:50051'):
        self.channel = grpc.insecure_channel(
            host,
            options=[
                ('grpc.max_send_message_length', 50 * 1024 * 1024),
                ('grpc.max_receive_message_length', 50 * 1024 * 1024),
            ]
        )
        self.stub = user_pb2_grpc.UserServiceStub(self.channel)

    def get_user(self, user_id):
        """유저 조회"""
        try:
            response = self.stub.GetUser(
                user_pb2.GetUserRequest(id=user_id),
                timeout=5.0
            )
            return response.user
        except grpc.RpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
            return None

    def list_users(self, page_size=10):
        """유저 목록 조회"""
        try:
            response = self.stub.ListUsers(
                user_pb2.ListUsersRequest(page_size=page_size),
                timeout=5.0
            )
            return list(response.users)
        except grpc.RpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
            return []

    def create_user(self, name, email):
        """유저 생성"""
        try:
            response = self.stub.CreateUser(
                user_pb2.CreateUserRequest(name=name, email=email),
                timeout=5.0
            )
            return response.user
        except grpc.RpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
            return None

    def delete_user(self, user_id):
        """유저 삭제"""
        try:
            response = self.stub.DeleteUser(
                user_pb2.DeleteUserRequest(id=user_id),
                timeout=5.0
            )
            return response.success
        except grpc.RpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
            return False

    def close(self):
        """연결 종료"""
        self.channel.close()

def main():
    client = UserClient()

    print("=" * 50)
    print("1. 유저 목록 조회")
    print("=" * 50)
    users = client.list_users()
    for user in users:
        print(f"  - {user.id}: {user.name} ({user.email})")

    print("\n" + "=" * 50)
    print("2. 단일 유저 조회 (id=1)")
    print("=" * 50)
    user = client.get_user(1)
    if user:
        print(f"  - {user.id}: {user.name} ({user.email})")

    print("\n" + "=" * 50)
    print("3. 유저 생성")
    print("=" * 50)
    new_user = client.create_user("박지민", "park@example.com")
    if new_user:
        print(f"  Created: {new_user.id}: {new_user.name} ({new_user.email})")

    print("\n" + "=" * 50)
    print("4. 유저 목록 다시 조회")
    print("=" * 50)
    users = client.list_users()
    for user in users:
        print(f"  - {user.id}: {user.name} ({user.email})")

    print("\n" + "=" * 50)
    print("5. 존재하지 않는 유저 조회 (id=999)")
    print("=" * 50)
    user = client.get_user(999)

    print("\n" + "=" * 50)
    print("6. 유저 삭제 (id=1)")
    print("=" * 50)
    success = client.delete_user(1)
    print(f"  Deleted: {success}")

    print("\n" + "=" * 50)
    print("7. 최종 유저 목록")
    print("=" * 50)
    users = client.list_users()
    for user in users:
        print(f"  - {user.id}: {user.name} ({user.email})")

    client.close()

if __name__ == '__main__':
    main()

```

---

## Step 6: 실행

### 터미널 1: 서버 실행

```bash
python server/main.py

```

```
출력:
Server started on port 50051
Press Ctrl+C to stop

```

### 터미널 2: 클라이언트 실행

```bash
python client/main.py

```

```
출력:
==================================================
1. 유저 목록 조회
==================================================
  - 1: 홍길동 (hong@example.com)
  - 2: 김철수 (kim@example.com)
  - 3: 이영희 (lee@example.com)

==================================================
2. 단일 유저 조회 (id=1)
==================================================
  - 1: 홍길동 (hong@example.com)

==================================================
3. 유저 생성
==================================================
  Created: 4: 박지민 (park@example.com)

==================================================
4. 유저 목록 다시 조회
==================================================
  - 1: 홍길동 (hong@example.com)
  - 2: 김철수 (kim@example.com)
  - 3: 이영희 (lee@example.com)
  - 4: 박지민 (park@example.com)

==================================================
5. 존재하지 않는 유저 조회 (id=999)
==================================================
Error: StatusCode.NOT_FOUND - User 999 not found

==================================================
6. 유저 삭제 (id=1)
==================================================
  Deleted: True

==================================================
7. 최종 유저 목록
==================================================
  - 2: 김철수 (kim@example.com)
  - 3: 이영희 (lee@example.com)
  - 4: 박지민 (park@example.com)

```

---

## 추가: grpcurl로 테스트

### 설치

```bash
# Mac
brew install grpcurl

# Linux
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

```

### 사용

```bash
# 서비스 목록 조회 (reflection 필요)
grpcurl -plaintext localhost:50051 list

# 메서드 호출
grpcurl -plaintext \
  -d '{"id": 1}' \
  localhost:50051 user.UserService/GetUser

# 유저 생성
grpcurl -plaintext \
  -d '{"name": "테스트", "email": "test@example.com"}' \
  localhost:50051 user.UserService/CreateUser

```

### Reflection 활성화 (grpcurl 사용 시 필요)

```python
# server/main.py에 추가
from grpc_reflection.v1alpha import reflection

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))

    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(),
        server
    )

    # Reflection 활성화
    SERVICE_NAMES = (
        user_pb2.DESCRIPTOR.services_by_name['UserService'].full_name,
        reflection.SERVICE_NAME,
    )
    reflection.enable_server_reflection(SERVICE_NAMES, server)

    # ... 나머지 동일

```

```bash
pip install grpcio-reflection

```

---

## 핵심 정리

### 개발 순서

1. `.proto` 파일 작성
2. `protoc`으로 코드 생성
3. 서버: `Servicer` 상속 → 메서드 구현
4. 클라이언트: `Stub` 사용 → RPC 호출

### 서버 핵심 코드

```python
# 서비스 구현
class MyServiceServicer(my_pb2_grpc.MyServiceServicer):
    def MyMethod(self, request, context):
        # 비즈니스 로직
        return my_pb2.MyResponse(...)

# 서버 실행
server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
my_pb2_grpc.add_MyServiceServicer_to_server(MyServiceServicer(), server)
server.add_insecure_port('[::]:50051')
server.start()

```

### 클라이언트 핵심 코드

```python
# 연결
channel = grpc.insecure_channel('localhost:50051')
stub = my_pb2_grpc.MyServiceStub(channel)

# 호출
response = stub.MyMethod(my_pb2.MyRequest(...), timeout=5.0)

```