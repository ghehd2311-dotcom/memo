
    링크:
    https://blog.bespinglobal.com/post/ai-agent-api-test-design-to-validate-consultation-flow-and-errors/?utm_source=rss&utm_medium=rss&utm_campaign=ai-agent-api-test-design-to-validate-consultation-flow-and-errors

    ## AI 요약

    ## 🤖 AI 고객 컨택 에이전트 테스트 설계 요약

이 문서는 이벤트 검색, 상세 질의, 사용자별 기록(즐겨찾기·최근 본 기록)을 제공하는 **AI 에이전트** 시스템의 테스트 전략과 설계를 다룹니다. 개별 API의 성공 여부를 넘어 상담사가 실제 업무를 원활하게 이어갈 수 있는지 확인하는 **상담 흐름 중심의 테스트**를 강조합니다.

---

## 1. 테스트의 핵심 방향: API 성공을 넘어 상담 흐름으로

- **업무 연속성 검증**: 검색, 질의, 저장 API가 각각 정상이어도 사이의 연결(예: 검색 결과의 `bundle_id` 전달)이 끊기면 상담이 중단됩니다. 따라서 테스트 기준을 "호출 성공"에서 **"업무 수행 가능 여부"**로 확장해야 합니다.
- **다층적 테스트 범위**:
  - **단위/정적 검증**: 날짜 경계, 묶음 정규화, 프롬프트 누락 등 작은 규칙은 빠르고 일정하게 확인합니다.
  - **API 계약 테스트**: 헤더(`X-User-ID`, `X-Session-ID`), 요청/응답 구조, 오류 코드를 검증합니다.
  - **통합 및 시나리오 테스트**: DB, 검색기, 실제 인증 시스템과의 연계 및 검색부터 저장·재조회까지의 흐름을 검증합니다.

---

## 2. 주요 컴포넌트별 테스트 포인트

### 🔍 검색 API (`/events/search`)
- **결과 구조 검증**: `documents` 배열, 유효한 `bundle_id`, 첨부 문서(`documents_by_attach`)가 올바른 이벤트에 속하는지 확인합니다.
- **기간 필터(Date Filter)**: 날짜 형식(`YYYYMMDD`)뿐만 아니라 윤년, 존재하지 않는 날짜, 시작일/종료일 역전 등 **경계값 조건**을 엄격하게 테스트합니다.

### 💬 상세 질의 API (`/events/query`)
- **컨텍스트 및 헤더 격리**: 사용자 ID, 그룹 ID, 세션 ID, 묶음 ID의 역할이 정확히 동작하는지, 타인의 세션이나 이력을 침범하지 않는지 검증합니다.
- **예외 처리**: 필수 헤더 누락 시 `400`, 본문 검증 실패 시 `422` 등 클라이언트가 명확히 인지할 수 있는 오류 코드를 반환해야 합니다.

```python
from fastapi import FastAPI, Header, HTTPException, Depends
from pydantic import BaseModel, Field, StringConstraints
from typing import Annotated

Text = Annotated[str, StringConstraints(strip_whitespace=True, min_length=1)]

class QueryInput(BaseModel):
    query: Text
    bundle_ids: list[Text] = Field(min_length=1)

def request_context(
    user_id: Annotated[str | None, Header(alias="X-User-ID")] = None,
    session_id: Annotated[str | None, Header(alias="X-Session-ID")] = None,
):
    if not user_id or not user_id.strip():
        raise HTTPException(400, detail={"code": "USER_REQUIRED"})
    if not session_id or not session_id.strip():
        raise HTTPException(400, detail={"code": "SESSION_REQUIRED"})
    return {"user_id": user_id, "session_id": session_id}
```

### 👤 사용자 데이터 격리 (즐겨찾기 & 최근 본 기록)
- 복수의 사용자(`상담사 A`, `상담사 B`)를 상정하여 동일한 이력 ID나 묶음 ID를 요청했을 때 **권한 정책에 따라 거부**되는지, 타인의 데이터가 노출되지 않는지 확인합니다.

---

## 3. 테스트 대역(Test Double) 활용과 품질 회귀

- **테스트 대역**: 외부 모델이나 DB 응답을 기다리지 않고, 정해진 결과와 입력값(`calls`)을 기록하는 `FakeService`를 활용해 API 계약과 경계 조건을 빠르게 검증합니다.
- **품질과 성능의 동시 검증**:
  - API 계약 통과와 별개로 **평균 응답 시간(Latency)** 및 `p95` 지연 분포, 동시 요청 처리량을 함께 모니터링합니다.
  - 검색 평가 지표(예: `Recall@k`)를 활용하여 답변 품질의 회귀를 방지합니다.

```python
def recall_at_k(expected_ids, ranked_ids, k):
    if k < 1:
        raise ValueError("k는 양수여야 합니다.")
    expected = set(expected_ids)
    if not expected:
        return None  
    return len(expected & set(ranked_ids[:k])) / len(expected)
```

---

## 4. 결론 및 체크리스트

배포 전에는 다음 사항을 반드시 점검해야 합니다:
1. **문서 일치성**: 대표 문서와 첨부 문서가 정확히 같은 이벤트에 속하는가?
2. **오류 구분**: 입력 오류, 빈 검색, 외부 연결 장애가 명확히 구분되는가?
3. **데이터 격리**: 다른 사용자의 세션, 검색 이력, 즐겨찾기가 섞이지 않는가?
4. **품질 동조**: API 계약 검증과 주요 질의 품질 평가가 모두 기준을 만족하는가?

> **핵심 키워드**: **#상담AI**, **#AIAgent**, **#APITest**, **#FastAPI**, **#Pytest**, **#ContractTest**, **#RegressionTest**, **#EventSearch**, **#RAG**, **#Favorite**, **#RecentView**, **#테스트자동화**, **#품질회귀**, **#QA**
    