---
title: "[gRPC] 핵심 요약"
date: 2025-12-29 04:00:00 +0900
categories: [Tech, gRPC]
tags: [grpc, summary, cheatsheet]
mermaid: true
---

> **📚 gRPC 시리즈 - 핵심 요약**
>
> **Part 1. 기초 개념**
> - [RPC란 무엇인가](/posts/rpc-concept/)
> - [HTTP/2와 gRPC](/posts/http2/)
> - [Protocol Buffers 완벽 가이드](/posts/protobuf/)
> - [IDL과 직렬화](/posts/idl-serialization/)
>
> **Part 2. gRPC 핵심 개념**
> - [.proto 파일과 코드 생성](/posts/proto-codegen/)
> - [4가지 통신 패턴](/posts/grpc-patterns/)
> - [Channel, Metadata, Error Handling](/posts/grpc-advanced/)
> - [gRPC vs REST 비교](/posts/grpc-vs-rest/)
>
> **Part 3. 실전 구현과 ML 서빙 적용**
> - [Python gRPC 서버/클라이언트 구현](/posts/grpc-python-impl/)
> - [비동기 gRPC (asyncio)](/posts/grpc-async/)
> - [Triton Inference Server gRPC](/posts/grpc-triton/)
> - [vLLM / KServe gRPC 연동](/posts/grpc-vllm-kserve/)

---

# gRPC 핵심 요약

## gRPC란?

Google이 만든 **RPC 프레임워크** (HTTP/2 + Protobuf 기반)

```
RPC = 원격 함수를 로컬처럼 호출하는 개념
gRPC = RPC의 구현체 (Google 버전)
```

---

## 왜 쓰는가?

| REST | gRPC |
| --- | --- |
| 텍스트 (JSON) | 바이너리 (Protobuf) |
| 매번 헤더 전송 | 헤더 압축 (HPACK) |
| 요청-응답만 | 스트리밍 지원 |
| 브라우저 직접 호출 ✅ | 브라우저 직접 호출 ❌ |

**→ 내부 서비스 간 통신에서 성능 이점**

---

## 언제 쓰는가?

| 상황 | 선택 |
| --- | --- |
| 외부 API (브라우저, 모바일) | REST |
| 내부 마이크로서비스 통신 | gRPC |
| 대용량 데이터/배치 처리 | gRPC |
| LLM 토큰 스트리밍 | gRPC |
| 빠른 개발/디버깅 | REST |
| Triton 고성능 추론 | gRPC |
| vLLM | REST (충분) |

---

## 개발 흐름

```
1. .proto 파일 작성 (서비스 + 메시지 정의)
        ↓
2. protoc로 코드 생성 (자동)
        ↓
3. 서버: Servicer 상속 → 메서드 구현
   클라이언트: Stub으로 호출

```

---

## 최소 코드

### proto 파일

```protobuf
service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

```

### 서버

```python
class UserServiceServicer(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        return user_pb2.GetUserResponse(user=...)

server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
user_pb2_grpc.add_UserServiceServicer_to_server(UserServiceServicer(), server)
server.add_insecure_port('[::]:50051')
server.start()

```

### 클라이언트

```python
channel = grpc.insecure_channel('localhost:50051')
stub = user_pb2_grpc.UserServiceStub(channel)
response = stub.GetUser(user_pb2.GetUserRequest(id=1))

```

---

## 4가지 통신 패턴

| 패턴 | proto 키워드 | 용도 |
| --- | --- | --- |
| Unary | 없음 | 일반 API |
| Server Streaming | `returns (stream X)` | LLM 토큰 스트리밍 |
| Client Streaming | `(stream X) returns` | 파일 업로드 |
| Bidirectional | 양쪽 stream | 실시간 채팅 |

---

## 실무 패턴

```
브라우저/모바일 ─── REST ───► FastAPI ─── gRPC ───► 내부 서비스
                            (Gateway)              Triton 등

```

**외부는 REST, 내부는 gRPC**

---

## 동기 vs 비동기

|  | 동기 | 비동기 |
| --- | --- | --- |
| 모듈 | `grpc` | `grpc.aio` |
| 적합 | CPU 바운드 | I/O 바운드 |

---

## 핵심 한 줄

> gRPC = 내부 서비스 간 고성능 통신용
> 
> 
> **외부 API는 REST, 내부 통신은 gRPC**
>