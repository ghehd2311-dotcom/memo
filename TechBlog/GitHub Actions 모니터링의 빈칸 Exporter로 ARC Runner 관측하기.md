
    링크:
    https://medium.com/daangn/github-actions-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81%EC%9D%98-%EB%B9%88%EC%B9%B8-exporter%EB%A1%9C-arc-runner-%EA%B4%80%EC%B8%A1%ED%95%98%EA%B8%B0-cbe1a46a9e71?source=rss----4505f82a2dbd---4

    ## AI 요약

    ## GitHub Actions 모니터링의 빈칸: Exporter로 ARC Runner 관측하기

당근 SRE팀은 Kubernetes 환경 기반의 **ARC(Actions Runner Controller)**를 통해 GitHub Actions self-hosted Runner를 관리하고 있습니다. 하지만 기본 제공 메트릭만으로는 대기 중인 **Job** 개수, 대기 시간, Pod와 Job의 매핑 관계를 파악하기 어려워 이를 해결하기 위해 **action-runner-job-exporter**를 개발 및 개선했습니다.

---

## Exporter 구성 요소

exporter는 세 개의 핵심 바이너리로 구성되어 Runner와 Job의 상태를 관측합니다.

- **Collector**: Kubernetes informer로 `EphemeralRunner`의 상태를 추적하여 Runner Pod와 **Job** 이름을 연결합니다.
  ```text
  arc_runner_job_info{namespace, pod, runner_name, repository, job_name, workflow_ref, workflow_run_id}
  ```
- **Webhook**: GitHub의 `workflow_job` 이벤트를 직접 수신하여 **Queued**, **In_Progress**, **Completed** 등 대기 중인 Job의 상태를 추적하고 메트릭으로 제공합니다.
- **StateStore**: Webhook의 재시작 시 상태 유실을 방지하고, 사내망에서 안전하게 S3 스냅샷 저장을 담당합니다.

---

## 주요 운영 개선 사항

1. **데이터별 보관 기간 분리 및 스냅샷 최적화**
   - 기존의 단일 `--webhook-ttl` 설정을 분리하여 **tombstone**과 **Delivery ID**의 보관 기간을 줄이고, 스냅샷 복원 유효 기한(`--snapshot-max-age`)을 도입하여 OOMKilled 및 잘못된 상태 복원 문제를 해결했습니다.
2. **실패한 웹훅 이벤트 자동 재전송**
   - StateStore가 GitHub App을 통해 5분마다 전송 기록을 확인하고, 최근 30분 내 실패한 `in_progress`, `completed` 이벤트를 재전송하도록 보완했습니다.
3. **리컨사일(Reconcile)을 통한 상태 동기화**
   - 웹훅 지연이나 누락으로 완료된 Job이 대기 중인 것으로 잘못 집계되는 문제를 막기 위해, 일정 시간 변화가 없는 Job은 Actions API를 조회하여 실제 상태로 동기화합니다.

---

## 주요 활용 사례

- **Job별 리소스 사용량 최적화**: Collector를 통해 CPU·메모리 사용량과 Job 이름을 연결하여, 각 Job에 적절한 크기의 **Runner**를 선택할 수 있는 대시보드를 구축했습니다.
- **ARC 설정 병목 탐지**: **Webhook 메트릭**과 ARC 컨트롤러 메트릭을 함께 분석하여, 동시 처리 수 설정(`runnerMaxConcurrentReconciles`) 부족으로 인한 지연을 파악하고 값을 상향 조정하여 병목을 해결했습니다.
    