---
title: "[gRPC 핵심개념 3] Channel, Metadata, Error Handling"
date: 2025-12-28 22:00:00 +0900
categories: [Tech, gRPC]
tags: [grpc, channel, metadata, error-handling]
mermaid: true
---

> **📚 gRPC 시리즈 - Part 2. gRPC 핵심 개념**
>
> 1. [.proto 파일과 코드 생성](/posts/proto-codegen/)
> 2. [4가지 통신 패턴](/posts/grpc-patterns/)
> 3. Channel, Metadata, Error Handling ← 현재 글
> 4. [gRPC vs REST 비교](/posts/grpc-vs-rest/)

---

## 왜 이걸 알아야 하는가?

gRPC 서버/클라이언트를 만들었다면 실전에서 마주치는 문제들이 있다:

- "연결은 어떻게 관리하지?" → Channel
- "인증 토큰은 어떻게 보내지?" → Metadata
- "에러 처리는 어떻게 하지?" → Error Handling

이 세 가지를 알아야 프로덕션 레벨의 gRPC 서비스를 만들 수 있다.

---

## 1. Channel

### Channel이란?

```mermaid
flowchart LR
    subgraph Channel["Channel (HTTP/2 Connection)"]
        Client["Client"]
        Server["Server"]
    end

    Client ===|"Channel"| Server

    Client --> S1["Stub 1 → GetUser()"]
    Client --> S2["Stub 2 → ListUsers()"]
    Client --> S3["Stub 3 → CreateUser()"]
```

**Channel = 서버와의 연결을 추상화한 객체**

- 하나의 Channel로 여러 Stub이 공유 가능
- HTTP/2 멀티플렉싱 덕분

### 기본 사용법

```python
import grpc

# 1. 비보안 채널 (개발용)
channel = grpc.insecure_channel('localhost:50051')

# 2. 보안 채널 (프로덕션)
credentials = grpc.ssl_channel_credentials()
channel = grpc.secure_channel('api.example.com:443', credentials)

# 3. Stub 생성
stub = user_pb2_grpc.UserServiceStub(channel)
```

### Channel 옵션

```python
channel = grpc.insecure_channel(
    'localhost:50051',
    options=[
        # 메시지 크기 제한
        ('grpc.max_send_message_length', 50 * 1024 * 1024),      # 50MB
        ('grpc.max_receive_message_length', 50 * 1024 * 1024),   # 50MB

        # Keepalive 설정
        ('grpc.keepalive_time_ms', 30000),           # 30초마다 ping
        ('grpc.keepalive_timeout_ms', 10000),        # 10초 내 응답 없으면 끊김
        ('grpc.keepalive_permit_without_calls', 1),  # 호출 없어도 keepalive

        # 재연결 설정
        ('grpc.initial_reconnect_backoff_ms', 1000),  # 첫 재연결 대기: 1초
        ('grpc.max_reconnect_backoff_ms', 30000),     # 최대 재연결 대기: 30초
    ]
)
```

### Channel 수명 관리

```python
# 방법 1: 명시적 close
channel = grpc.insecure_channel('localhost:50051')
stub = user_pb2_grpc.UserServiceStub(channel)

# 사용
response = stub.GetUser(request)

# 종료
channel.close()

# 방법 2: Context Manager (권장)
with grpc.insecure_channel('localhost:50051') as channel:
    stub = user_pb2_grpc.UserServiceStub(channel)
    response = stub.GetUser(request)
# 자동으로 close됨

# 방법 3: 싱글톤 패턴 (서비스에서 재사용)
class GrpcClient:
    _channel = None
    _stub = None

    @classmethod
    def get_stub(cls):
        if cls._channel is None:
            cls._channel = grpc.insecure_channel('localhost:50051')
            cls._stub = user_pb2_grpc.UserServiceStub(cls._channel)
        return cls._stub

    @classmethod
    def close(cls):
        if cls._channel:
            cls._channel.close()
```

### Channel 상태 확인

```python
# 연결 상태 확인
state = channel._channel.check_connectivity_state(True)

# 상태 종류
# grpc.ChannelConnectivity.IDLE        - 대기 중
# grpc.ChannelConnectivity.CONNECTING  - 연결 중
# grpc.ChannelConnectivity.READY       - 연결됨
# grpc.ChannelConnectivity.TRANSIENT_FAILURE - 일시적 실패
# grpc.ChannelConnectivity.SHUTDOWN    - 종료됨
```

---

## 2. Metadata

### Metadata란?

**Metadata = HTTP 헤더와 유사한 키-값 쌍**

| 비교 | REST | gRPC |
| --- | --- | --- |
| 헤더 전달 | HTTP Header | Metadata |

**용도:**
- 인증 토큰 (Authorization)
- 요청 추적 ID (X-Request-ID)
- 커스텀 정보 전달

### 클라이언트에서 Metadata 전송

```python
# 방법 1: 호출 시 직접 전달
metadata = [
    ('authorization', 'Bearer eyJhbGciOiJIUzI1NiIs...'),
    ('x-request-id', 'req-123-456'),
    ('x-user-id', '12345'),
]

response = stub.GetUser(
    request,
    metadata=metadata
)

# 방법 2: Interceptor로 자동 추가 (권장)
class AuthInterceptor(grpc.UnaryUnaryClientInterceptor):
    def __init__(self, token):
        self.token = token

    def intercept_unary_unary(self, continuation, client_call_details, request):
        # 기존 metadata 가져오기
        metadata = list(client_call_details.metadata or [])

        # 토큰 추가
        metadata.append(('authorization', f'Bearer {self.token}'))

        # 새 call_details 생성
        new_details = grpc.ClientCallDetails(
            client_call_details.method,
            client_call_details.timeout,
            metadata,
            client_call_details.credentials,
            client_call_details.wait_for_ready,
            client_call_details.compression,
        )

        return continuation(new_details, request)

# 사용
channel = grpc.insecure_channel('localhost:50051')
intercept_channel = grpc.intercept_channel(channel, AuthInterceptor('my-token'))
stub = user_pb2_grpc.UserServiceStub(intercept_channel)
```

### 서버에서 Metadata 읽기

```python
class UserServiceServicer(user_pb2_grpc.UserServiceServicer):

    def GetUser(self, request, context):
        # Metadata 읽기
        metadata = dict(context.invocation_metadata())

        # 인증 토큰 확인
        auth_header = metadata.get('authorization', '')
        if not auth_header.startswith('Bearer '):
            context.abort(grpc.StatusCode.UNAUTHENTICATED, 'Invalid token')

        token = auth_header.replace('Bearer ', '')

        # 요청 ID
        request_id = metadata.get('x-request-id', 'unknown')
        print(f"Request ID: {request_id}")

        # 비즈니스 로직
        user = db.get_user(request.id)
        return user_pb2.GetUserResponse(user=user)
```

### 서버에서 Metadata 응답

```python
class UserServiceServicer(user_pb2_grpc.UserServiceServicer):

    def GetUser(self, request, context):
        # 초기 Metadata 전송 (응답 전)
        context.send_initial_metadata([
            ('x-server-version', '1.0.0'),
            ('x-processing-time', '50ms'),
        ])

        user = db.get_user(request.id)

        # Trailing Metadata 전송 (응답 후)
        context.set_trailing_metadata([
            ('x-cache-hit', 'true'),
        ])

        return user_pb2.GetUserResponse(user=user)
```

### 클라이언트에서 응답 Metadata 읽기

```python
# 응답 Metadata 받기
response, call = stub.GetUser.with_call(
    request,
    metadata=metadata
)

# 초기 metadata
initial_metadata = dict(call.initial_metadata())
print(f"Server version: {initial_metadata.get('x-server-version')}")

# Trailing metadata
trailing_metadata = dict(call.trailing_metadata())
print(f"Cache hit: {trailing_metadata.get('x-cache-hit')}")
```

---

## 3. Error Handling

### gRPC Status Code

| 코드 | 의미 | HTTP 대응 |
| --- | --- | --- |
| OK (0) | 성공 | 200 |
| CANCELLED (1) | 클라이언트가 취소 | 499 |
| UNKNOWN (2) | 알 수 없는 에러 | 500 |
| INVALID_ARGUMENT (3) | 잘못된 인자 | 400 |
| DEADLINE_EXCEEDED (4) | 타임아웃 | 504 |
| NOT_FOUND (5) | 리소스 없음 | 404 |
| ALREADY_EXISTS (6) | 이미 존재 | 409 |
| PERMISSION_DENIED (7) | 권한 없음 | 403 |
| RESOURCE_EXHAUSTED (8) | 리소스 부족 (쿼터 등) | 429 |
| UNAUTHENTICATED (16) | 인증 필요 | 401 |
| INTERNAL (13) | 서버 내부 에러 | 500 |
| UNAVAILABLE (14) | 서비스 사용 불가 | 503 |

### 서버에서 에러 반환

```python
class UserServiceServicer(user_pb2_grpc.UserServiceServicer):

    def GetUser(self, request, context):
        # 방법 1: context.abort() - 즉시 종료
        if request.id <= 0:
            context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                'User ID must be positive'
            )

        user = db.get_user(request.id)

        if user is None:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f'User {request.id} not found'
            )

        return user_pb2.GetUserResponse(user=user)

    def CreateUser(self, request, context):
        # 방법 2: context.set_code() + set_details()
        if not request.email:
            context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
            context.set_details('Email is required')
            return user_pb2.CreateUserResponse()

        try:
            user = db.create_user(request.name, request.email)
            return user_pb2.CreateUserResponse(user=user)
        except DuplicateEmailError:
            context.set_code(grpc.StatusCode.ALREADY_EXISTS)
            context.set_details(f'Email {request.email} already exists')
            return user_pb2.CreateUserResponse()
```

### 클라이언트에서 에러 처리

```python
import grpc

# 방법 1: try-except로 처리
try:
    response = stub.GetUser(user_pb2.GetUserRequest(id=999))
    print(response.user)

except grpc.RpcError as e:
    status_code = e.code()
    details = e.details()

    if status_code == grpc.StatusCode.NOT_FOUND:
        print(f"유저를 찾을 수 없다: {details}")

    elif status_code == grpc.StatusCode.INVALID_ARGUMENT:
        print(f"잘못된 요청: {details}")

    elif status_code == grpc.StatusCode.UNAUTHENTICATED:
        print("인증이 필요하다")

    elif status_code == grpc.StatusCode.UNAVAILABLE:
        print("서버에 연결할 수 없다")

    else:
        print(f"에러 발생: {status_code} - {details}")

# 방법 2: 에러별 분기 처리
def get_user_safe(stub, user_id):
    try:
        response = stub.GetUser(user_pb2.GetUserRequest(id=user_id))
        return response.user, None

    except grpc.RpcError as e:
        return None, {
            'code': e.code().name,
            'message': e.details()
        }

user, error = get_user_safe(stub, 999)
if error:
    print(f"Error: {error['code']} - {error['message']}")
```

### 타임아웃 설정

```python
# 클라이언트: 호출별 타임아웃
try:
    response = stub.GetUser(
        user_pb2.GetUserRequest(id=123),
        timeout=5.0  # 5초 타임아웃
    )
except grpc.RpcError as e:
    if e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
        print("요청 시간 초과")
```

### 재시도 로직

```python
import time

def call_with_retry(stub, request, max_retries=3, backoff=1.0):
    """재시도 로직이 포함된 호출"""

    retryable_codes = [
        grpc.StatusCode.UNAVAILABLE,
        grpc.StatusCode.DEADLINE_EXCEEDED,
    ]

    for attempt in range(max_retries):
        try:
            return stub.GetUser(request, timeout=5.0)

        except grpc.RpcError as e:
            if e.code() not in retryable_codes:
                raise  # 재시도 불가능한 에러

            if attempt == max_retries - 1:
                raise  # 마지막 시도 실패

            # 백오프 후 재시도
            wait_time = backoff * (2 ** attempt)
            print(f"재시도 {attempt + 1}/{max_retries}, {wait_time}초 후...")
            time.sleep(wait_time)
```

---

## 실전 예시: 인증 + 에러 처리

### 서버

```python
class UserServiceServicer(user_pb2_grpc.UserServiceServicer):

    def _authenticate(self, context):
        """인증 처리 공통 함수"""
        metadata = dict(context.invocation_metadata())
        auth_header = metadata.get('authorization', '')

        if not auth_header:
            context.abort(grpc.StatusCode.UNAUTHENTICATED, 'Token required')

        if not auth_header.startswith('Bearer '):
            context.abort(grpc.StatusCode.UNAUTHENTICATED, 'Invalid token format')

        token = auth_header.replace('Bearer ', '')

        try:
            user_id = verify_token(token)  # 토큰 검증
            return user_id
        except TokenExpiredError:
            context.abort(grpc.StatusCode.UNAUTHENTICATED, 'Token expired')
        except InvalidTokenError:
            context.abort(grpc.StatusCode.UNAUTHENTICATED, 'Invalid token')

    def GetUser(self, request, context):
        # 인증
        current_user_id = self._authenticate(context)

        # 권한 확인
        if request.id != current_user_id:
            context.abort(grpc.StatusCode.PERMISSION_DENIED, 'Cannot access other users')

        # 비즈니스 로직
        user = db.get_user(request.id)
        if not user:
            context.abort(grpc.StatusCode.NOT_FOUND, 'User not found')

        return user_pb2.GetUserResponse(user=user)
```

### 클라이언트

```python
class UserClient:
    def __init__(self, host, token):
        self.channel = grpc.insecure_channel(host)
        self.stub = user_pb2_grpc.UserServiceStub(self.channel)
        self.metadata = [('authorization', f'Bearer {token}')]

    def get_user(self, user_id):
        try:
            response = self.stub.GetUser(
                user_pb2.GetUserRequest(id=user_id),
                metadata=self.metadata,
                timeout=5.0
            )
            return {'success': True, 'user': response.user}

        except grpc.RpcError as e:
            code = e.code()

            if code == grpc.StatusCode.UNAUTHENTICATED:
                return {'success': False, 'error': 'auth_failed', 'message': e.details()}

            elif code == grpc.StatusCode.PERMISSION_DENIED:
                return {'success': False, 'error': 'forbidden', 'message': e.details()}

            elif code == grpc.StatusCode.NOT_FOUND:
                return {'success': False, 'error': 'not_found', 'message': e.details()}

            else:
                return {'success': False, 'error': 'unknown', 'message': str(e)}

    def close(self):
        self.channel.close()
```

---

## 핵심 정리

### Channel

- 서버와의 연결 추상화
- 하나의 Channel로 여러 Stub 공유 가능
- 옵션으로 keepalive, 메시지 크기 등 설정
- Context Manager 사용 권장

### Metadata

- HTTP Header와 유사
- 인증 토큰, 추적 ID 등 전달
- Interceptor로 자동 추가 가능
- 서버에서 `context.invocation_metadata()`로 읽기

### Error Handling

- gRPC 전용 상태 코드 사용 (NOT_FOUND, INVALID_ARGUMENT 등)
- 서버: `context.abort()` 또는 `set_code()`
- 클라이언트: `grpc.RpcError` 예외 처리
- 타임아웃: `timeout` 파라미터
