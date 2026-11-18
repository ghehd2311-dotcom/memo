
    # Databricks Lakehouse 아키텍처를 다시 정리하며 배운 점(손재홍_AI플랫폼엔지니어링)

    링크:
    https://blog.bespinglobal.com/post/lessons-learned-from-revisiting-databricks-lakehouse-architecture/?utm_source=rss&utm_medium=rss&utm_campaign=lessons-learned-from-revisiting-databricks-lakehouse-architecture

    ## AI 요약

    ## Databricks Lakehouse 아키텍처 재조명

AI플랫폼엔지니어링팀 손재홍님이 작성한 Databricks **Lakehouse** 아키텍처 학습 내용의 핵심 요약입니다.

### 1. Lakehouse의 정의와 도입 배경
* **Data Lake의 특징**: 저렴하고 대량의 다양한 원천 데이터(로그, 반정형 데이터 등)를 저장하기 좋으나, 품질 관리, 스키마 관리, 트랜잭션, 권한 관리가 약함.
* **Data Warehouse의 특징**: 정제된 데이터를 안정적으로 분석하고 BI/리포트에 강하나, 데이터 복제와 파이프라인 관리로 인해 데이터가 늘어날수록 비용과 복잡도가 증가함.
* **Lakehouse**: 클라우드 오브젝트 스토리지 위에 구축되면서도 **Data Lake**의 개방성·확장성과 **Data Warehouse**의 안정성·거버넌스를 모두 갖춘 구조입니다. 단순한 저장소의 개념을 넘어 **데이터 운영 방식**에 초점을 맞춘 아키텍처입니다.

### 2. Delta Lake의 역할과 핵심 기능
단순한 파일(Parquet 등) 저장을 넘어 테이블처럼 운영할 수 있게 해주는 핵심 계층입니다.
* 메타데이터와 **트랜잭션 로그**를 활용
* 주요 제공 기능:
  - **ACID 트랜잭션**
  - **스키마 강제 (Schema Enforcement)**
  - **Time Travel (버전 관리)**

### 3. 운영 관점에서의 이점
* **장애 추적 용이성**: 버전 및 커밋 단위로 상태를 파악할 수 있어 배치 작업 실패 시 원인 추적이 단순해짐.
* **스키마 변경 대응**: 컬럼 추가나 타입 변경 등의 이슈를 테이블 관리 기능 내에서 다루어 Downstream(ETL/리포트) 오류 방지.
* **ML/AI 연계성**: 다양한 형태의 데이터(로그, 텍스트, 이미지 등)를 유연하게 담으면서도 재현성을 확보하고, BI와 ML/AI가 동일한 기반 데이터를 공유하도록 지원.

### 4. 실무 운영 시 고려사항 및 향후 과제
Lakehouse가 모든 문제를 자동으로 해결해주지 않으므로, 데이터가 한곳에 모이는 만큼 명확한 관리 기준이 필요합니다.

* **핵심 운영 포인트**:
  - 테이블 변경 이력 추적
  - 배치 실패 시점 파악
  - 데이터 품질 및 스키마 변경 사전 감지

* **추가 학습 예정 기술**:
  - `Unity Catalog`
  - `Lakeflow`
  - `Workflows`
  - `Delta Live Tables`
    