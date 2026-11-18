
    # SGLang을 활용한 다국어 및 Multimodal Decision Model 구성하기

    링크:
    https://aws.amazon.com/ko/blogs/tech/decision-model-with-sglang/

    ## AI 요약

    ## # SGLang을 활용한 다국어 및 Multimodal Decision Model 구성 요약

생성형 AI 트래픽의 상당 부분은 문장 생성보다 **“판단”**(카테고리 분류, 상태 확인, 조치 선택 등)에 해당합니다. 본문은 **SGLang**과 Open Weight VLM인 **Qwen3-VL-8B**를 활용하여, 닫힌 선택지 판단을 전담하는 **Decision Model** 아키텍처를 구축하고 이를 **Amazon Bedrock AgentCore**에 연동하는 방법을 다룹니다.

---

## ## 1. 핵심 아키텍처 및 동작 원리

* **Prefill 전용(Decode 제거) 처리**: 
  LLM 호출 비용의 대부분을 차지하는 순차 생성(Decode) 단계를 거치지 않고, 입력 병렬 처리인 **Prefill 단계**만 활용합니다. 선택지에 Single-token Label(A, B, C 등)을 부여하여 Prefill 시점의 Logits에서 답과 **Confidence**(확신도)를 동시에 도출합니다.
* **Wvocab Prior 활용**: 
  사전학습된 어휘 투사 행렬을 그대로 활용하므로, 별도의 **Fine-tuning** 없이도 다국어 라벨을 **Zero-shot**으로 처리할 수 있습니다.
* **솔루션 구성**:
  * **Amazon Bedrock AgentCore Runtime**: 서버리스 AI Agent 호스팅, 세션 격리 및 VPC 연결 제공.
  * **Amazon EC2 (g6.xlarge)**: **SGLang**과 Qwen3-VL-8B를 탑재하여 텍스트 및 이미지 판단, 일반 생성까지 단일 서버로 수행.
  * **Amazon Bedrock (Claude Sonnet 5.5)**: 최종 응답 생성 및 저신뢰(`Confidence < Threshold`) 요청의 **Escalation** 처리.

---

## ## 2. 성능 및 비용 실측 결과

은행 상담 Agent 160개 대화 시나리오를 통한 A/B 테스트 결과, 기존 Frontier Model(Claude Haiku 4.5) 단독 구성 대비 뛰어난 효율을 기록했습니다.

* **지연 시간(Latency) 단축**: 
  판단 1회당 Latency가 **약 15배 단축**(531~552ms ➔ 34~38ms)되었으며, 대화 전체(e2e) p50 Latency는 **33.6% 감소**했습니다.
* **비용 절감**: 
  판단 노드의 Bedrock 호출 감소로 인해 GPU 상각 비용을 포함하고도 **대화당 비용 18% 절감**을 달성했습니다.
* **품질 유지**: 
  Blind Pairwise 평가 결과, 최종 응답 품질은 기존 방식과 통계적 유의차 없이 대등한 수준을 유지했습니다. (단, 안전 관련 **Guardrail(D1)** 판단은 저신뢰 시 Frontier Model로 넘기는 **Hybrid 구성** 권장)

---

## ## 3. Multimodal 및 한국어 판단 처리

* **이미지 판단 경로**: 
  SGLang의 OpenAI 호환 `/v1/chat/completions` 엔드포인트를 사용하며, 외부 URL 접근이 불가능한 VPC 환경에서는 **base64 Data URL** 형식으로 이미지를 전달하여 보안과 데이터 통제를 충족합니다.
* **한국어 판단 품질 확보 (두 가지 접근)**:
  1. **접근 A (기본 구성)**: **무Fine-tuning + Calibration**. Wvocab Prior를 활용해 Zero-shot으로 즉시 사용하되, 과신(Overconfident)되는 Confidence는 **Reliability Table**(신뢰도 보정 표) 기반의 **Threshold 보정**을 적용합니다.
  2. **접근 B**: 경량 모델(예: Laya 322M)을 한국어 도메인 데이터로 반복 **Fine-tuning**. 단, Task나 선택지가 바뀔 때마다 재학습과 재검증이 필요합니다.

---

## ## 4. 주요 구성 및 코드 스니펫 예시

### SGLang 서버 실행 환경
```bash
# Amazon EC2 g6.xlarge (NVIDIA L4 24GB), Ubuntu 22.04, SGLang v0.5.21
python -m sglang.launch_server \
    --model-path Qwen/Qwen3-VL-8B \
    --port 30000
```

### Decision Schema 및 API 호출 특징
* **자유 텍스트**: `state` 필드에 입력
* **선택지 정의**: 애플리케이션 개발 시점에 지정
* **응답 형태**: Typed Answer와 **Confidence** 자동 반환

---

## ## 5. 결론 및 도입 효과

* **일관된 다국어/Multimodal 처리**: 단일 SGLang 서버로 한국어 텍스트 판단과 이미지 판단을 모두 수십 밀리초(ms) 단위로 처리 가능합니다.
* **보안 및 데이터 통제**: 판단 트래픽이 Internet Gateway가 없는 **VPC 내부**에 머물며, **AWS PrivateLink**를 통한 안전한 Bedrock 연동을 지원합니다.
* **경제성**: 일정 규모(월 수십만~백만 건 이상의 판단 요청) 이상인 프로덕션 환경에서 인스턴스 고정비 기반의 **비용 효율성**을 극대화할 수 있습니다.
    