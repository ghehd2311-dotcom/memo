
    링크:
    https://blog.bespinglobal.com/post/ai-agent-conversation-history-storage-and-management/?utm_source=rss&utm_medium=rss&utm_campaign=ai-agent-conversation-history-storage-and-management

    ## AI 요약

    ## AI 상담 서비스의 대화 이력 관리 및 맥락 구성 요약

Developer팀 **유수영**님이 작성한 본 문서는 대화형 AI 상담 서비스에서 연속성 있는 대화를 제공하기 위해 **대화 이력(Conversation History)**을 기록하고 관리하는 구조를 다룹니다.

---

## 1. 개요와 이력의 용도
상담 서비스에서 사용자의 이전 설명을 반복하지 않도록 하려면 대화 이력 관리가 필수적입니다. 이력은 크게 세 가지 용도로 구분됩니다.
- **화면 복원**: 사용자에게 이전 질문, 답변, 순서, 완료 상태를 다시 보여줍니다.
- **후속 질문**: "그 행사"와 같은 대명사나 맥락을 AI 모델이 이어서 이해할 수 있도록 합니다.
- **운영 점검**: 답변이 달라진 원인을 파악하기 위해 요청 식별자, 오류 상태 등을 추적합니다.

---

## 2. 저장 책임과 아키텍처
서비스 전반에 분산된 저장 코드를 통합하기 위해 **미들웨어**, **이력 관리 계층**, **조회 API**, **맥락 구성기**의 역할을 명확히 분리해야 합니다.

```python
# 예시: Turn 데이터 구조 (Dataclass)
@dataclass(frozen=True)
class Turn:
    tenant_id: str
    user_id: str
    session_id: str
    request_id: str
    agent_name: str
    turn_no: int
    question: str
    answer: str
    status: Literal["completed", "interrupted", "failed"] = "completed"
```

- **미들웨어**: 대상 요청과 응답 완료 시점을 판단하여 공통 **마스킹** 처리 및 저장을 호출합니다.
- **이력 관리 계층**: 트랜잭션, 중복 처리, 순서 관리를 담당합니다.
- **조회 API**: 소유권을 확인하고 화면에 필요한 이력을 제공합니다.
- **식별자 분리**: 대화 세션(`session_id`), 답변 근거 자료인 문서 묶음(`group_id`), 요청 구분(`request_id`)은 각각 다른 의미를 가지므로 혼용해서는 안 됩니다.

---

## 3. 구현 핵심 요소

### 저장 전 마스킹 (공통 정책)
개인정보 보호를 위해 저장 직전에 **공통 마스킹 정책**을 적용하여 전화번호나 식별 정보 등을 필터링합니다.

```python
import re

IDENTIFIER = re.compile(r"(?<!\d)\d{6}-\d{7}(?!\d)")
PHONE = re.compile(r"(?<!\d)0\d{1,2}[- ]\d{3,4}[- ]\d{4}(?!\d)")


def mask_sensitive_text(text: str) -> str:
    text = IDENTIFIER.sub("[식별정보]", text)
    return PHONE.sub("[전화번호]", text)
```

### 순서와 소유권 보장
데이터베이스 설계 시 **복합 키**와 **외래 키**를 활용해 중복 요청을 방지하고, 세션과 사용자의 소유권을 일치시켜야 합니다.

---

## 4. 모델 맥락 구성과 스트리밍 관리
- **저장소와 모델 입력의 분리**: 모든 이력을 모델에 그대로 전달하면 토큰 낭비 및 오류가 발생할 수 있습니다. 최근 N개의 턴이나 길이 제한(`max_chars`)을 두어 현재 질문에 필요한 맥락만 조합합니다.
- **스트리밍 응답**: 첫 조각이 도착했다고 해서 대화가 완료된 것은 아닙니다. 모델 생성 완료, 서버 전송 완료, 사용자 수신을 구분하여 **완료된 답변**과 **중단된 답변**을 명확히 분리해 기록해야 합니다.

---

## 5. 저장 실패 및 운영 정책
- **내구성**: 응답 성공과 저장 성공은 별개입니다. 저장 실패 시 이미 전송된 응답을 바꾸기 어렵기 때문에, 업무 요구사항에 맞춰 백그라운드 작업이나 내구성 있는 이벤트 전달 방식을 선택해야 합니다.
- **운영 지표**: 저장 대상 요청 수, 저장 시도/성공/실패율, 중복 요청 처리 수 등을 모니터링하여 시스템 안정성을 점검합니다.

---

## 6. 태그
#ConversationHistory #MultiTurn #FastAPI #Middleware #Session #Memory #AIAgent #PostgreSQL #대화이력 #상담AI
    