
    # AWS에서 Laya를 통한 Decision Model 학습 및 배포 환경 구성하기

    링크:
    https://aws.amazon.com/ko/blogs/tech/aws-laya/

    ## AI 요약

    ## AWS에서 Laya를 통한 Decision Model 학습 및 배포 환경 구성 요약

### 1. Decision Model의 필요성장과 Laya 개요
* **Decision Model**: 텍스트를 직접 생성하지 않고 질문에 대한 **선택지별 확률**을 한 번에 반환하는 모델입니다.
* 기존 **LLM**의 한계(느린 응답 속도, 토큰 비용, JSON 파싱 오류, 비일관성)를 극복하여 에이전트 루프나 게이트웨이 앞단에서 반복적인 판단(예: 개인정보 필터링, 라우팅)을 처리하는 데 최적화되어 있습니다.
* **Laya**: 아파치 2.0 라이선스의 오픈소스 **Decision Model**로, 약 3.2억 파라미터의 mmBERT-base 인코더 기반 구조를 가집니다. 추가 학습(Fine-tuning)을 통해 도메인 특화 모델로 구축할 수 있습니다.

```python
# Laya 추론 서버 예시 코드 발췌 (app.py)
@app.post("/invocations")
async def invocations(request: Request):
    body = await request.json()
    texts = body["texts"] if "texts" in body else [body["text"]]
    results = router.predict_batch(reqs, batch_size=32)
    return {"results": results, "latency_ms": ...}
```

---

### 2. Amazon SageMaker를 활용한 학습 및 배포 파이프라인
* **컨테이너 빌드**: 학습(`train`)과 서빙(`serve`) 기능을 모두 포함하는 단일 Docker 이미지(`pytorch/pytorch:2.14.1-cuda12.6`)를 구축하여 **Amazon ECR**에 푸시합니다.
* **학습 작업(Training Job)**: 합성 데이터 및 공개 데이터셋(KDPII)을 활용해 **Amazon SageMaker**에서 GPU(ml.g6e.xlarge 등) 기반으로 3 에폭 동안 추가 학습을 진행합니다.
* **실시간 엔드포인트 배포**: 학습 완료된 모델 아티팩트(`model.tar.gz`)를 기반으로 **SageMaker 실시간 엔드포인트**를 무중단(`UpdateEndpoint`)으로 배포합니다. 

```bash
# Docker 이미지 빌드 및 ECR 푸시 명령어
docker build --platform linux/amd64 -t laya-serve:v1 serving/laya-serve
docker tag laya-serve:v1 $ECR_REPO:v1
docker push $ECR_REPO:v1
```

---

### 3. 호출 및 결정 규칙 (ALLOW / BLOCK / REVIEW)
* **확률 기반 분기 처리**: 모델이 반환한 `sensitive.noul` 확률 및 카테고리 최대 확률을 기준으로 호출 측(LLM 게이트웨이)에서 최종 결정을 내립니다.
  * **ALLOW**: 개인정보 미포함 안전 요청
  * **BLOCK**: 개인정보 포함 위험 요청 차단
  * **REVIEW**: 확률이 애매하거나 두 질문의 답이 어긋나는 경우 인간 검토 또는 상위 모델로 이관
* **보조 규칙**: 모델의 학습 범위 밖인 API 키나 토큰 같은 민감 인증정보는 **정규식 규칙**을 별도로 두어 보완합니다.

```python
def decide(answers, p_block=0.5, p_margin=0.6):
    p_sensitive = answers["sensitive"]["noul"]
    category = answers["category"]["choice"]
    max_p = max(answers["category"]["probabilities"].values())
    
    if (p_sensitive >= p_block) != (category != "none") or max_p < p_margin:
        return {"decision": "REVIEW"}
    return {"decision": "BLOCK" if p_sensitive >= p_block else "ALLOW"}
```

---

### 4. 벤치마크 결과 및 장단점 비교
* **성능 및 속도**: 추가 학습된 Laya는 제로샷 모델이나 수백 배 큰 대형 LLM(Qwen3-235B, Claude Haiku 등)과 유사한 수준의 높은 정확도를 기록했습니다.
* **지연 시간 및 비용**: 가장 작은 GPU 인스턴스 기준 **판정 1건당 수십 ms(약 35~39 ms)** 소요되며, 토큰 과금이 아닌 **인스턴스 시간 과금** 구조를 가집니다.
* **주요 장단점**:
  * **장점**: 빠른 속도, 고정 비용, **데이터가 외부로 유출되지 않음**(보안성), 교차 검증 가능.
  * **단점**: 텍스트 생성이나 마스킹(스팬 추출) 불가, 1,024 토큰의 입력 길이 제한, 데이터 품질에 민감함.
    