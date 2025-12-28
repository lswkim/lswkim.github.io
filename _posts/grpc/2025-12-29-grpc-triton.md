---
title: "[gRPC 실전/ML서빙 3] Triton Inference Server gRPC"
date: 2025-12-29 02:00:00 +0900
categories: [Tech, gRPC]
tags: [grpc, triton, inference-server, nvidia, ml-serving]
mermaid: true
---

> **📚 gRPC 시리즈 - Part 3. 실전 구현과 ML 서빙 적용**
>
> 1. [Python gRPC 서버/클라이언트 구현](/posts/grpc-python-impl/)
> 2. [비동기 gRPC (asyncio)](/posts/grpc-async/)
> 3. Triton Inference Server gRPC ← 현재 글
> 4. [vLLM / KServe gRPC 연동](/posts/grpc-vllm-kserve/)

---

# 3. Triton Inference Server gRPC 연동

## 왜 이걸 알아야 하는가?

Triton Inference Server는 NVIDIA의 ML 모델 서빙 솔루션이다.

- 고성능 추론이 필요할 때 → gRPC 사용
- 실제 MLOps 현장에서 가장 많이 쓰는 조합
- KServe, vLLM도 비슷한 패턴

gRPC를 배웠으니, 실제 ML 서빙에 적용해본다.

---

## Triton 통신 방식

### 지원하는 프로토콜

| 프로토콜 | 포트 (기본) | 용도 |
| --- | --- | --- |
| HTTP/REST | 8000 | 디버깅, 간단한 테스트 |
| gRPC | 8001 | 프로덕션, 고성능 |
| Metrics | 8002 | Prometheus 메트릭 |

### 언제 gRPC를 쓰나?

```
┌─────────────────────────────────────────────────────────────────────┐
│                      프로토콜 선택 기준                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   REST (8000)                                                       │
│   • curl로 빠른 테스트                                               │
│   • 디버깅, 개발 환경                                                │
│   • 트래픽 적은 서비스                                               │
│                                                                     │
│   gRPC (8001)                                                       │
│   • 프로덕션 환경                                                    │
│   • 대용량 텐서 전송                                                 │
│   • 낮은 지연시간 필요                                               │
│   • 배치 추론                                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

```

---

## Triton gRPC 클라이언트

### 설치

```bash
pip install tritonclient[grpc]

# 또는 전체 설치
pip install tritonclient[all]

```

### 기본 구조

```python
import tritonclient.grpc as grpcclient

# 클라이언트 생성
client = grpcclient.InferenceServerClient(
    url="localhost:8001",  # gRPC 포트
    verbose=False
)

# 서버 상태 확인
if client.is_server_live():
    print("Server is live")

if client.is_server_ready():
    print("Server is ready")

```

---

## 추론 요청 보내기

### 1. 입력 데이터 준비

```python
import numpy as np
import tritonclient.grpc as grpcclient

# 입력 데이터 (예: 이미지 분류)
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)

# InferInput 객체 생성
inputs = [
    grpcclient.InferInput(
        name="input",           # 모델의 입력 이름
        shape=[1, 3, 224, 224], # 입력 shape
        datatype="FP32"         # 데이터 타입
    )
]

# 데이터 설정
inputs[0].set_data_from_numpy(input_data)

```

### 2. 출력 설정

```python
# 출력 정의
outputs = [
    grpcclient.InferRequestedOutput(
        name="output",  # 모델의 출력 이름
    )
]

```

### 3. 추론 요청

```python
# 추론 실행
response = client.infer(
    model_name="resnet50",
    inputs=inputs,
    outputs=outputs,
)

# 결과 가져오기
output_data = response.as_numpy("output")
print(f"Output shape: {output_data.shape}")
print(f"Prediction: {np.argmax(output_data)}")

```

---

## 전체 예시 코드

### 이미지 분류 추론

```python
import numpy as np
import tritonclient.grpc as grpcclient
from tritonclient.utils import InferenceServerException

class TritonClient:
    """Triton gRPC 클라이언트 래퍼"""

    def __init__(self, url="localhost:8001"):
        self.client = grpcclient.InferenceServerClient(
            url=url,
            verbose=False
        )

    def check_health(self):
        """서버 상태 확인"""
        try:
            live = self.client.is_server_live()
            ready = self.client.is_server_ready()
            return {"live": live, "ready": ready}
        except InferenceServerException as e:
            return {"error": str(e)}

    def get_model_metadata(self, model_name):
        """모델 메타데이터 조회"""
        try:
            metadata = self.client.get_model_metadata(model_name)
            return {
                "name": metadata.name,
                "versions": metadata.versions,
                "inputs": [
                    {"name": inp.name, "shape": inp.shape, "datatype": inp.datatype}
                    for inp in metadata.inputs
                ],
                "outputs": [
                    {"name": out.name, "shape": out.shape, "datatype": out.datatype}
                    for out in metadata.outputs
                ],
            }
        except InferenceServerException as e:
            return {"error": str(e)}

    def infer(self, model_name, input_data, input_name="input", output_name="output"):
        """추론 실행"""
        try:
            # 입력 설정
            inputs = [
                grpcclient.InferInput(
                    name=input_name,
                    shape=input_data.shape,
                    datatype="FP32"
                )
            ]
            inputs[0].set_data_from_numpy(input_data.astype(np.float32))

            # 출력 설정
            outputs = [
                grpcclient.InferRequestedOutput(name=output_name)
            ]

            # 추론
            response = self.client.infer(
                model_name=model_name,
                inputs=inputs,
                outputs=outputs,
            )

            return response.as_numpy(output_name)

        except InferenceServerException as e:
            print(f"Inference error: {e}")
            return None

def main():
    client = TritonClient("localhost:8001")

    # 1. 서버 상태 확인
    print("=== Server Health ===")
    health = client.check_health()
    print(health)

    # 2. 모델 메타데이터 확인
    print("\n=== Model Metadata ===")
    metadata = client.get_model_metadata("resnet50")
    print(metadata)

    # 3. 추론 실행
    print("\n=== Inference ===")
    input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
    output = client.infer("resnet50", input_data)

    if output is not None:
        print(f"Output shape: {output.shape}")
        print(f"Top prediction: {np.argmax(output)}")

if __name__ == "__main__":
    main()

```

---

## 배치 추론

### 여러 샘플 한 번에 처리

```python
def batch_infer(client, model_name, batch_data):
    """배치 추론"""

    # batch_data shape: (batch_size, 3, 224, 224)
    batch_size = batch_data.shape[0]

    inputs = [
        grpcclient.InferInput(
            name="input",
            shape=batch_data.shape,  # (N, 3, 224, 224)
            datatype="FP32"
        )
    ]
    inputs[0].set_data_from_numpy(batch_data)

    outputs = [grpcclient.InferRequestedOutput(name="output")]

    response = client.infer(
        model_name=model_name,
        inputs=inputs,
        outputs=outputs,
    )

    return response.as_numpy("output")  # (N, num_classes)

# 사용 예시
batch_size = 8
batch_data = np.random.randn(batch_size, 3, 224, 224).astype(np.float32)

results = batch_infer(client, "resnet50", batch_data)
print(f"Batch results shape: {results.shape}")  # (8, 1000)

```

---

## 비동기 추론

### AsyncIO 기반

```python
import asyncio
import tritonclient.grpc.aio as grpcclient_aio

async def async_infer(url, model_name, input_data):
    """비동기 추론"""

    # 비동기 클라이언트
    client = grpcclient_aio.InferenceServerClient(url=url)

    inputs = [
        grpcclient_aio.InferInput(
            name="input",
            shape=input_data.shape,
            datatype="FP32"
        )
    ]
    inputs[0].set_data_from_numpy(input_data)

    outputs = [grpcclient_aio.InferRequestedOutput(name="output")]

    # 비동기 추론
    response = await client.infer(
        model_name=model_name,
        inputs=inputs,
        outputs=outputs,
    )

    await client.close()

    return response.as_numpy("output")

async def main():
    url = "localhost:8001"
    model_name = "resnet50"

    # 여러 요청 동시 실행
    tasks = []
    for i in range(10):
        input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
        tasks.append(async_infer(url, model_name, input_data))

    results = await asyncio.gather(*tasks)
    print(f"Completed {len(results)} inferences")

if __name__ == "__main__":
    asyncio.run(main())

```

---

## 스트리밍 추론 (LLM)

### Decoupled 모델 (토큰 스트리밍)

```python
import queue
import tritonclient.grpc as grpcclient

def stream_callback(result, error):
    """스트리밍 콜백"""
    if error:
        print(f"Error: {error}")
    else:
        output = result.as_numpy("output_ids")
        print(f"Token: {output}")

def stream_infer(client, model_name, input_text):
    """스트리밍 추론 (LLM)"""

    # 입력 설정
    input_data = np.array([[input_text]], dtype=object)

    inputs = [
        grpcclient.InferInput(
            name="text_input",
            shape=[1, 1],
            datatype="BYTES"
        )
    ]
    inputs[0].set_data_from_numpy(input_data)

    outputs = [grpcclient.InferRequestedOutput(name="output_ids")]

    # 스트리밍 추론 시작
    client.start_stream(callback=stream_callback)

    client.async_stream_infer(
        model_name=model_name,
        inputs=inputs,
        outputs=outputs,
    )

    # 완료 대기
    client.stop_stream()

```

---

## FastAPI + Triton 통합

### Gateway 패턴

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np
import tritonclient.grpc as grpcclient

app = FastAPI()

# Triton 클라이언트 (싱글톤)
triton_client = grpcclient.InferenceServerClient(url="triton-server:8001")

class PredictRequest(BaseModel):
    data: list  # 입력 데이터

class PredictResponse(BaseModel):
    prediction: int
    confidence: float

@app.get("/health")
async def health():
    """헬스 체크"""
    try:
        live = triton_client.is_server_live()
        ready = triton_client.is_server_ready()
        return {"triton_live": live, "triton_ready": ready}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))

@app.post("/predict", response_model=PredictResponse)
async def predict(request: PredictRequest):
    """추론 API"""
    try:
        # 입력 변환
        input_data = np.array(request.data, dtype=np.float32)
        input_data = input_data.reshape(1, 3, 224, 224)

        # Triton 입력 설정
        inputs = [
            grpcclient.InferInput("input", input_data.shape, "FP32")
        ]
        inputs[0].set_data_from_numpy(input_data)

        outputs = [grpcclient.InferRequestedOutput("output")]

        # 추론
        response = triton_client.infer(
            model_name="resnet50",
            inputs=inputs,
            outputs=outputs,
        )

        # 결과 처리
        output = response.as_numpy("output")
        prediction = int(np.argmax(output))
        confidence = float(np.max(output))

        return PredictResponse(
            prediction=prediction,
            confidence=confidence
        )

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

```

---

## 에러 처리

### 공통 에러 패턴

```python
from tritonclient.utils import InferenceServerException

def safe_infer(client, model_name, input_data):
    """에러 처리가 포함된 추론"""

    try:
        # 모델 준비 상태 확인
        if not client.is_model_ready(model_name):
            return {"error": f"Model {model_name} is not ready"}

        # 추론 실행
        inputs = [grpcclient.InferInput("input", input_data.shape, "FP32")]
        inputs[0].set_data_from_numpy(input_data)
        outputs = [grpcclient.InferRequestedOutput("output")]

        response = client.infer(
            model_name=model_name,
            inputs=inputs,
            outputs=outputs,
            client_timeout=10.0,  # 타임아웃 설정
        )

        return {"success": True, "output": response.as_numpy("output")}

    except InferenceServerException as e:
        # Triton 서버 에러
        return {"error": f"Triton error: {e.message()}"}

    except Exception as e:
        # 기타 에러
        return {"error": f"Unknown error: {str(e)}"}

```

### 재시도 로직

```python
import time

def infer_with_retry(client, model_name, input_data, max_retries=3):
    """재시도 로직이 포함된 추론"""

    for attempt in range(max_retries):
        try:
            inputs = [grpcclient.InferInput("input", input_data.shape, "FP32")]
            inputs[0].set_data_from_numpy(input_data)
            outputs = [grpcclient.InferRequestedOutput("output")]

            response = client.infer(
                model_name=model_name,
                inputs=inputs,
                outputs=outputs,
            )

            return response.as_numpy("output")

        except InferenceServerException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # 지수 백오프
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise

```

---

## K8s 환경 설정

### 서비스 연결

```yaml
# triton-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: triton-server
spec:
  selector:
    app: triton
  ports:
    - name: http
      port: 8000
      targetPort: 8000
    - name: grpc
      port: 8001
      targetPort: 8001
    - name: metrics
      port: 8002
      targetPort: 8002

```

```python
# K8s 내부에서 연결
client = grpcclient.InferenceServerClient(
    url="triton-server:8001",  # 서비스 이름으로 접근
)

```

---

## 성능 팁

### 1. 연결 재사용

```python
# ❌ 매번 새 클라이언트 생성
def infer(data):
    client = grpcclient.InferenceServerClient(url="localhost:8001")
    # ...

# ✅ 클라이언트 재사용
client = grpcclient.InferenceServerClient(url="localhost:8001")

def infer(data):
    # 기존 client 사용
    # ...

```

### 2. 배치 처리

```python
# ❌ 개별 요청
for data in dataset:
    result = infer(data.reshape(1, ...))

# ✅ 배치 요청
batch = np.stack(dataset)
results = infer(batch)  # 한 번에 처리

```

### 3. 비동기 처리

```python
# ❌ 순차 처리
results = []
for data in inputs:
    results.append(await infer(data))

# ✅ 동시 처리
tasks = [infer(data) for data in inputs]
results = await asyncio.gather(*tasks)

```

---

## 핵심 정리

### Triton gRPC 사용 흐름

1. 클라이언트 생성: `InferenceServerClient(url="host:8001")`
2. 입력 설정: `InferInput` + `set_data_from_numpy()`
3. 출력 설정: `InferRequestedOutput`
4. 추론 실행: `client.infer()`
5. 결과 추출: `response.as_numpy()`

### REST vs gRPC 선택

| 상황 | 선택 |
| --- | --- |
| 개발/디버깅 | REST (8000) |
| 프로덕션 | gRPC (8001) |
| 대용량 텐서 | gRPC |
| 배치 추론 | gRPC |
| 스트리밍 (LLM) | gRPC |

| 상황 | 추천 | 이유 |
| --- | --- | --- |
| 디버깅/테스트 | HTTP | curl로 바로 테스트 |
| 단일 요청, 작은 데이터 | HTTP | 차이 미미, 개발 편함 |
| 팀이 gRPC 경험 없음 | HTTP | 러닝커브 없음 |
| 다른 팀과 협업 | HTTP | 스펙 공유 쉬움 |
| 대용량 텐서/배치 추론 | gRPC | 페이로드 효율적 전송 |
| 반복 호출 많음 | gRPC | 연결 재사용, 헤더 압축 |
| LLM 토큰 스트리밍 | gRPC | 네이티브 스트리밍 지원 |
| 극한의 저지연 필요 | gRPC | 바이너리 직렬화 |

### 주요 클래스

| 클래스 | 용도 |
| --- | --- |
| `InferenceServerClient` | 동기 클라이언트 |
| `InferenceServerClient` (aio) | 비동기 클라이언트 |
| `InferInput` | 입력 텐서 정의 |
| `InferRequestedOutput` | 출력 텐서 정의 |

---

## 기타 내용

## HTTP가 유리한 경우

| 상황 | 이유 |
| --- | --- |
| **디버깅/테스트** | curl로 바로 테스트 가능 |
| **단일 요청** | 프로토콜 오버헤드 차이 미미 |
| **작은 입력 데이터** | 텍스트 분류, 작은 이미지 등 |
| **팀이 gRPC 경험 없음** | 러닝커브 없이 빠른 개발 |
| **다른 팀과 협업** | 스펙 공유가 쉬움 |

---

## gRPC가 유리한 경우

| 상황 | 이유 |
| --- | --- |
| **대용량 텐서** | 이미지 배치, 고해상도 |
| **반복 호출이 많음** | 연결 재사용, 헤더 압축 |
| **배치 추론** | 큰 페이로드 효율적 전송 |
| **스트리밍 (LLM)** | 토큰 단위 응답 |
| **극한의 저지연 필요** | 실시간 서비스 |