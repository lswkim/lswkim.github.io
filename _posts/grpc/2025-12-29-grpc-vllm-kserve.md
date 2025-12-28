---
title: "[gRPC 실전/ML서빙 4] vLLM / KServe gRPC 연동"
date: 2025-12-29 03:00:00 +0900
categories: [Tech, gRPC]
tags: [grpc, vllm, kserve, llm, ml-serving]
mermaid: true
---

> **📚 gRPC 시리즈 - Part 3. 실전 구현과 ML 서빙 적용**
>
> 1. [Python gRPC 서버/클라이언트 구현](/posts/grpc-python-impl/)
> 2. [비동기 gRPC (asyncio)](/posts/grpc-async/)
> 3. [Triton Inference Server gRPC](/posts/grpc-triton/)
> 4. vLLM / KServe gRPC 연동 ← 현재 글

---

# 4. vLLM gRPC 연동

## vLLM이란?

vLLM은 LLM 추론에 특화된 고성능 서빙 엔진이다.

- PagedAttention으로 메모리 효율 극대화
- Continuous Batching으로 처리량 향상
- OpenAI 호환 API 제공

---

## vLLM 통신 방식

| 방식 | 포트 (기본) | 특징 |
| --- | --- | --- |
| OpenAI 호환 REST | 8000 | `/v1/completions`, `/v1/chat/completions` |
| gRPC | 별도 설정 | Triton 백엔드 사용 시 |

### vLLM의 특이점

vLLM은 기본적으로 **OpenAI 호환 REST API**를 제공한다.

```bash
# vLLM 서버 실행
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --port 8000

```

gRPC를 쓰려면:

1. **Triton + vLLM 백엔드** 조합
2. **직접 gRPC 래퍼 구현**

---

## 방법 1: OpenAI 호환 API (REST)

### 가장 간단한 방법

```python
import requests

response = requests.post(
    "http://localhost:8000/v1/completions",
    json={
        "model": "meta-llama/Llama-2-7b-chat-hf",
        "prompt": "안녕하세요",
        "max_tokens": 100,
        "temperature": 0.7,
    }
)

print(response.json()["choices"][0]["text"])

```

### OpenAI SDK 사용

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="dummy"  # vLLM은 API key 검증 안 함
)

response = client.completions.create(
    model="meta-llama/Llama-2-7b-chat-hf",
    prompt="안녕하세요",
    max_tokens=100,
)

print(response.choices[0].text)

```

### 스트리밍 (SSE)

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")

stream = client.chat.completions.create(
    model="meta-llama/Llama-2-7b-chat-hf",
    messages=[{"role": "user", "content": "안녕하세요"}],
    max_tokens=100,
    stream=True,  # 스트리밍 활성화
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)

```

---

## 방법 2: Triton + vLLM 백엔드 (gRPC)

### 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Client ─── gRPC (8001) ───► Triton Server ───► vLLM Backend       │
│                                                                     │
│   • Triton이 gRPC 인터페이스 제공                                     │
│   • vLLM이 실제 LLM 추론 수행                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

```

### 모델 저장소 구조

```
model_repository/
└── vllm_model/
    ├── 1/
    │   └── model.json
    └── config.pbtxt

```

### config.pbtxt

```
name: "vllm_model"
backend: "vllm"
max_batch_size: 0

input [
  {
    name: "text_input"
    data_type: TYPE_STRING
    dims: [ 1 ]
  },
  {
    name: "stream"
    data_type: TYPE_BOOL
    dims: [ 1 ]
    optional: true
  }
]

output [
  {
    name: "text_output"
    data_type: TYPE_STRING
    dims: [ -1 ]
  }
]

parameters {
  key: "model"
  value: { string_value: "meta-llama/Llama-2-7b-chat-hf" }
}

parameters {
  key: "gpu_memory_utilization"
  value: { string_value: "0.9" }
}

```

### gRPC 클라이언트 코드

```python
import numpy as np
import tritonclient.grpc as grpcclient

def generate(prompt, max_tokens=100, stream=False):
    client = grpcclient.InferenceServerClient(url="localhost:8001")

    # 입력 설정
    text_input = np.array([[prompt]], dtype=object)
    stream_input = np.array([[stream]], dtype=bool)

    inputs = [
        grpcclient.InferInput("text_input", [1, 1], "BYTES"),
        grpcclient.InferInput("stream", [1, 1], "BOOL"),
    ]
    inputs[0].set_data_from_numpy(text_input)
    inputs[1].set_data_from_numpy(stream_input)

    outputs = [grpcclient.InferRequestedOutput("text_output")]

    # 추론
    response = client.infer(
        model_name="vllm_model",
        inputs=inputs,
        outputs=outputs,
    )

    return response.as_numpy("text_output")[0].decode("utf-8")

# 사용
result = generate("대한민국의 수도는?")
print(result)

```

### 스트리밍 (gRPC)

```python
import queue
import numpy as np
import tritonclient.grpc as grpcclient

def stream_callback(result, error):
    if error:
        print(f"Error: {error}")
    else:
        output = result.as_numpy("text_output")
        token = output[0].decode("utf-8")
        print(token, end="", flush=True)

def stream_generate(prompt):
    client = grpcclient.InferenceServerClient(url="localhost:8001")

    text_input = np.array([[prompt]], dtype=object)
    stream_input = np.array([[True]], dtype=bool)

    inputs = [
        grpcclient.InferInput("text_input", [1, 1], "BYTES"),
        grpcclient.InferInput("stream", [1, 1], "BOOL"),
    ]
    inputs[0].set_data_from_numpy(text_input)
    inputs[1].set_data_from_numpy(stream_input)

    outputs = [grpcclient.InferRequestedOutput("text_output")]

    # 스트리밍 시작
    client.start_stream(callback=stream_callback)

    client.async_stream_infer(
        model_name="vllm_model",
        inputs=inputs,
        outputs=outputs,
    )

    client.stop_stream()

# 사용
stream_generate("오늘 날씨가 좋네요.")

```

---

## 방법 3: 직접 gRPC 서버 구현

### vLLM + 커스텀 gRPC 래퍼

```protobuf
// llm.proto
syntax = "proto3";

package llm;

message GenerateRequest {
    string prompt = 1;
    int32 max_tokens = 2;
    float temperature = 3;
}

message GenerateResponse {
    string text = 1;
}

message Token {
    string text = 1;
    bool is_finished = 2;
}

service LLMService {
    rpc Generate(GenerateRequest) returns (GenerateResponse);
    rpc StreamGenerate(GenerateRequest) returns (stream Token);
}

```

### 서버 구현

```python
import grpc
from concurrent import futures
from vllm import LLM, SamplingParams

from generated import llm_pb2, llm_pb2_grpc

class LLMServiceServicer(llm_pb2_grpc.LLMServiceServicer):

    def __init__(self, model_name):
        self.llm = LLM(model=model_name)

    def Generate(self, request, context):
        """단일 응답"""
        sampling_params = SamplingParams(
            max_tokens=request.max_tokens,
            temperature=request.temperature,
        )

        outputs = self.llm.generate([request.prompt], sampling_params)
        text = outputs[0].outputs[0].text

        return llm_pb2.GenerateResponse(text=text)

    def StreamGenerate(self, request, context):
        """스트리밍 응답"""
        sampling_params = SamplingParams(
            max_tokens=request.max_tokens,
            temperature=request.temperature,
        )

        # vLLM 스트리밍 생성
        for output in self.llm.generate(
            [request.prompt],
            sampling_params,
            stream=True
        ):
            token = output.outputs[0].text
            is_finished = output.finished

            yield llm_pb2.Token(text=token, is_finished=is_finished)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=4))

    llm_pb2_grpc.add_LLMServiceServicer_to_server(
        LLMServiceServicer("meta-llama/Llama-2-7b-chat-hf"),
        server
    )

    server.add_insecure_port("[::]:50051")
    server.start()
    print("LLM gRPC server started on 50051")
    server.wait_for_termination()

if __name__ == "__main__":
    serve()

```

### 클라이언트 구현

```python
import grpc
from generated import llm_pb2, llm_pb2_grpc

def generate(prompt, max_tokens=100):
    channel = grpc.insecure_channel("localhost:50051")
    stub = llm_pb2_grpc.LLMServiceStub(channel)

    response = stub.Generate(
        llm_pb2.GenerateRequest(
            prompt=prompt,
            max_tokens=max_tokens,
            temperature=0.7,
        )
    )

    return response.text

def stream_generate(prompt, max_tokens=100):
    channel = grpc.insecure_channel("localhost:50051")
    stub = llm_pb2_grpc.LLMServiceStub(channel)

    for token in stub.StreamGenerate(
        llm_pb2.GenerateRequest(
            prompt=prompt,
            max_tokens=max_tokens,
            temperature=0.7,
        )
    ):
        print(token.text, end="", flush=True)
        if token.is_finished:
            break

# 사용
print(generate("안녕하세요"))
print()
stream_generate("오늘 점심 뭐 먹을까?")

```

---

## 방법 비교

| 방법 | 장점 | 단점 |
| --- | --- | --- |
| **OpenAI 호환 REST** | 간단, 표준 SDK 사용 | gRPC 대비 오버헤드 |
| **Triton + vLLM** | Triton 생태계 활용, 고성능 | 설정 복잡 |
| **직접 gRPC 구현** | 완전한 커스터마이징 | 개발/유지보수 부담 |

---

## 언제 뭘 쓸까?

| 상황 | 추천 |
| --- | --- |
| 빠른 개발/PoC | OpenAI 호환 REST |
| 이미 Triton 사용 중 | Triton + vLLM 백엔드 |
| 극한의 커스터마이징 필요 | 직접 gRPC 구현 |
| 단순 채팅 서비스 | OpenAI 호환 REST |
| 대규모 프로덕션 | Triton + vLLM 또는 직접 구현 |

---

## FastAPI + vLLM 통합 예시

### REST Gateway (가장 실용적)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from openai import OpenAI
import json

app = FastAPI()

vllm_client = OpenAI(
    base_url="http://vllm-server:8000/v1",
    api_key="dummy"
)

class ChatRequest(BaseModel):
    message: str
    max_tokens: int = 100

@app.post("/chat")
async def chat(request: ChatRequest):
    """일반 응답"""
    response = vllm_client.chat.completions.create(
        model="meta-llama/Llama-2-7b-chat-hf",
        messages=[{"role": "user", "content": request.message}],
        max_tokens=request.max_tokens,
    )
    return {"response": response.choices[0].message.content}

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    """스트리밍 응답"""

    def generate():
        stream = vllm_client.chat.completions.create(
            model="meta-llama/Llama-2-7b-chat-hf",
            messages=[{"role": "user", "content": request.message}],
            max_tokens=request.max_tokens,
            stream=True,
        )

        for chunk in stream:
            if chunk.choices[0].delta.content:
                yield f"data: {json.dumps({'text': chunk.choices[0].delta.content})}\n\n"

        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")

```

---

## 핵심 정리

### vLLM 통신 방식

- **기본**: OpenAI 호환 REST API
- **gRPC**: Triton 백엔드 또는 직접 구현

### 선택 기준

| 상황 | 선택 |
| --- | --- |
| 간단한 서비스 | OpenAI 호환 REST |
| Triton 환경 | Triton + vLLM |
| 완전한 제어 필요 | 직접 gRPC 구현 |

### 스트리밍

- REST: SSE (Server-Sent Events)
- gRPC: Server Streaming RPC

---

## 기타내용

## vLLM에 gRPC가 필요한가

### 불필요

| 특징 | 설명 |
| --- | --- |
| **기본이 REST** | vLLM이 OpenAI 호환 API를 네이티브로 제공 |
| **스트리밍 지원** | REST에서도 SSE로 토큰 스트리밍 가능 |
| **생태계** | OpenAI SDK 그대로 사용 가능 |
| **성능 충분** | LLM 추론 자체가 병목, 프로토콜 차이 미미 |

---

### vLLM vs Triton 차이

|  | vLLM | Triton |
| --- | --- | --- |
| **기본 프로토콜** | REST (OpenAI 호환) | REST + gRPC 둘 다 |
| **gRPC 필요성** | 낮음 | 높음 (대용량 텐서) |
| **주 사용처** | LLM 전용 | 범용 모델 서빙 |

---

### 왜 Triton은 gRPC가 유리한가?

```
Triton: 이미지 배치 32장 (15MB) → gRPC 효과 큼
vLLM:   텍스트 프롬프트 (수 KB) → REST로 충분

```

---

### 한 줄 정리

> vLLM은 OpenAI 호환 REST로 충분
> 
> 
> **gRPC는 Triton처럼 대용량 텐서 전송할 때 의미 있음**
>