
    링크:
    https://blog.bespinglobal.com/post/ai-agent-service-architecture-from-search-to-consultation-history/?utm_source=rss&utm_medium=rss&utm_campaign=ai-agent-service-architecture-from-search-to-consultation-history

    ## AI 요약

    ## 🤖 상담 AI Agent 서비스 설계 요약

이 문서는 **FastAPI**와 **서비스 컨테이너**, 공통 Agent 구성요소를 활용하여 상담사의 검색, 상세 답변, 기록 저장 흐름을 유기적으로 연결하고, 책임에 따라 서비스를 분리하는 아키텍처 설계 사례를 다룹니다.

---

## 1. 주요 설계 및 아키텍처 핵심

### 🔍 검색과 상세 질문의 분리
- **이벤트 검색**: 후보 이벤트를 빠르게 찾는 데 집중하며, 모델 의존성이 필수가 아닙니다. (검색 이력 관리)
- **선택 이벤트 상세 질문**: 선택된 묶음 문서(`bundle_id`)를 근거로 모델이 상담 답변을 생성합니다. (대화 및 답변 상태 관리)

### 📦 서비스 컨테이너와 앱 수명 관리 (`Lifespan`)
- 생성 비용이 큰 DB 및 Agent 연결을 공유하되, 사용자별 상태(ID, 질문 등)는 공유하지 않고 인자로 전달합니다.
- 동시 요청 시 초기화 구간을 보호하며, FastAPI의 `lifespan`을 통해 앱 시작 시 자원 준비 및 종료 시 자원 해제를 관리합니다.

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager

def create_app(service_factory, *, eager=True):
    @asynccontextmanager
    async def lifespan(app):
        container = ServiceContainer(service_factory)
        app.state.container = container
        try:
            if eager:
                await container.get_service()
            yield
        finally:
            await container.aclose()
    
    app = FastAPI(title="상담 지원 서비스", lifespan=lifespan)
    return app
```

### 🛡️ 라우터(Router)와 보안 경계
- **HTTP 계약**과 **업무 실행**을 분리합니다.
- 클라이언트가 전달하는 본문의 `user_id`를 신뢰하지 않고, 인증된 **`Principal`** 객체를 기반으로 접근 권한을 검증합니다.

```python
@router.post("/events/search", response_model=QueryResponse)
async def search_event(
    body: SearchRequest, 
    request: Request,
    principal: Annotated[Principal, Depends(require_principal)]
):
    service = await request.app.state.container.get_service()
    return await service.search(query=body.query, top_k=body.top_k, principal=principal)
```

---

## 2. 데이터 관리 및 장애 격리

### 🗄️ 검색 이력과 대화 이력의 분리
- **검색 이력**: 검색 조건과 결과 스냅샷 저장 (`search_history`)
- **즐겨찾기 / 최근 본 이벤트**: 사용자별 관심 이벤트 관리
- **대화 이력**: 질문과 답변의 흐름 재구성
- 보안을 위해 즐겨찾기 메타데이터 보완 시 반드시 **소유권 확인**(`user_id` 검증)을 거칩니다.

### ⚠️ 기능별 장애 격리 (Fault Isolation)
- 모델 초기화 실패가 **검색 기능 전체의 장애**로 이어지지 않도록 의존성을 분리합니다.
- 공유 자원(DB 등)의 물리적 장애 범위를 고려하고, 기능별 준비 상태를 명확히 구분하여 운영 가용성을 높입니다.

---

## 3. 결론

상담 AI 서비스는 사용자의 끊김 없는 흐름을 보장하면서도, **변경 이유와 자원 수명**에 따라 책임을 명확히 나누어야 합니다. 명확한 경계 설정을 통해 대역 테스트와 실제 연결 검증을 효과적으로 수행할 수 있습니다.

---

## 🏷️ 태그
**#상담AI** **#AIAgent** **#ServiceArchitecture** **#FastAPI** **#RAG** **#DependencyInjection** **#ConversationHistory** **#PostgreSQL** **#EventHistory** **#서비스컨테이너** **#운영관측**
    