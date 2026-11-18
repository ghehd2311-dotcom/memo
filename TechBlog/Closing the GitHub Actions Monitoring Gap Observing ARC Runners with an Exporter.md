
    링크:
    https://medium.com/daangn/closing-the-github-actions-monitoring-gap-observing-arc-runners-with-an-exporter-0a8f39c3b70b?source=rss----4505f82a2dbd---4

    ## AI 요약

    ## 당근페이(Karrot) SRE 팀의 **GitHub Actions** 모니터링 개선기 요약

당근페이 SRE 팀은 쿠버네티스 기반의 **ARC(Actions Runner Controller)**를 활용해 사내 CI 시스템을 운영하고 있습니다. 기존 시스템은 대기 중인 **Job**이나 **Runner Pod**의 상세 상태를 파악하기 어려워, 이를 해결하기 위해 자체 모니터링 익스포터인 **`action-runner-job-exporter`**를 개발했습니다.

---

## 1. 익스포터의 3가지 핵심 컴포넌트

익스포터는 각각 독립된 바이너리인 3개의 컴포넌트로 구성됩니다.

*   **Collector**: 
    *   **EphemeralRunner**와 **Runner Pod**의 이름이 동일하다는 점을 활용해, 상태(status) 정보에서 Job 이름과 Pod을 매핑합니다.
    *   `arc_runner_job_info` 메트릭을 노출하며, 이를 컨테이너 리소스 메트릭과 조인해 각 Job이 사용하는 CPU/메모리/네트워크 사용량을 추적할 수 있습니다.
*   **Webhook**: 
    *   GitHub 조직(Organization)으로부터 `workflow_job` 이벤트를 직접 수신하여 대기 중인(queued) 및 실행 중인(in_progress) **Job**의 상태를 추적합니다.
    *   상태의 순서(`queued` → `in_progress` → `completed`)를 관리하며, 만료된 상태는 **tombstones**로 일정 시간 유지하여 순서가 뒤바뀐 이벤트로 인한 오인식을 방지합니다.
*   **StateStore**: 
    *   인터넷에 노출된 **Webhook** 대신 내부 네트워크에서 안전하게 **S3**와 통신하며 상태 스냅샷을 보존합니다.
    *   롤아웃 시 `ETag / If-Match`를 활용한 S3 조건부 쓰기로 데이터 유실 및 덮어쓰기를 방지합니다.

---

## 2. 운영 중 발생한 주요 문제와 해결책

### ① 스냅샷 비대화 및 Stale 상태 문제
*   **원인**: `--webhook-ttl` 설정 하나로 진행 중인 Job, Tombstone, Delivery ID 보관 기간 및 스냅샷 복구 창을 모두 제어하여, 불필요한 Tombstone이 쌓여 8MiB 제한을 초과하고 OOMKilled가 발생했습니다.
*   **해결**: 
    *   데이터 유형별로 보관 기간을 분리 (`--webhook-ttl 6h`, `--tombstone-ttl 3h`, `--dedup-ttl 10m`, `--snapshot-max-age 10m`).
    *   상태 복구 및 웹훅 수신 여부를 감지하는 메트릭 추가.

### ② 실패한 웹훅 및 상태 불일치 문제
*   **원인**: 네트워크 이슈 등으로 GitHub 웹훅 누락 시, Job이 잘못된 상태로 수 시간 동안 방치되는 현상 발생.
*   **해결**:
    *   **StateStore**가 GitHub App을 통해 5분마다 최근 30분간 실패한 웹훅의 재요청(Redelivery) 수행.
    *   10분 이상 상태 변화가 없는 **Job**을 Actions API와 주기적으로 대조하여 상태를 보정하는 **Reconciliation(상태 조율)** 로직 도입.

---

## 3. 러너 리소스 최적화 및 병목 현상 해결

*   **러너 적정 크기 산정 (Right-sizing)**:
    *   **Collector**를 통해 수집된 Job별 실제 자원 사용량과 설정된 요청량을 비교하여, 최적의 **Runner** 사양을 추천하는 대시보드 구축 (불필요한 리소스 낭비 및 스케줄링 지연 방지).
*   **ARC 컨트롤러 병목 해결**:
    *   대기열이 급증했을 때 원인을 분석한 결과, EphemeralRunner 컨트롤러의 최대 동시 처리 수(`runnerMaxConcurrentReconciles`)가 기본값인 `2`로 고정되어 병목이 발생함을 확인했습니다.
    *   해당 값을 `5`로 상향 조정하여 컨트롤러 처리량을 높이고 큐 백로그를 성공적으로 해소했습니다.

```bash
# ARC 컨트롤러의 동시 처리 설정 예시 (runnerMaxConcurrentReconciles)
runnerMaxConcurrentReconciles: 5
```

---

## 4. 결론

당근페이 SRE 팀은 자체 개발한 **`action-runner-job-exporter`**를 통해 개발자와 운영자에게 **GitHub Actions**의 가시성을 제공하고 있습니다. 이를 통해 장애 대응뿐만 아니라 **Runner** 자원 최적화, 대기 시간 단축, 병목 구간 식별 등 인프라 운영 전반의 효율성을 크게 향상시켰습니다.
    