
    # AWS Emulator 시작하기 – 계정 없이 AWS 애플리케이션을 로컬에서 테스트하는 방법과 활용 사례

    링크:
    https://aws.amazon.com/ko/blogs/tech/getting-started-with-aws-emulator-local-testing-without-an-aws-account/

    ## AI 요약

    # AWS Emulator 시작하기 요약

## 1. AWS Emulator 개요
**AWS Emulator**는 AWS API를 단일 로컬 프로세스로 흉내 내어 계정이나 자격 증명 없이도 로컬 환경에서 AWS 애플리케이션을 테스트할 수 있는 도구입니다. 모든 리소스를 메모리에 처리하므로 실행 시마다 독립된 가상 계정(`000000000000`)과 깨끗한 상태를 제공합니다.

- **주요 특징**:
  - 실행마다 고유한 액세스 키 및 인증서 생성 (`~/.aws/emulator/<포트>/`)
  - 모든 데이터는 메모리에만 저장되며 종료 시 소멸
  - 미지원 작업의 경우 명확한 `HTTP 501 NotImplemented` 응답 반환
  - 운영 워크로드가 아닌 **빌드 및 테스트 전용** 도구

---

## 2. 설치 및 연결 설정

### 2.1 설치 방법
Linux 패키지(RPM/Debian)를 다운로드하여 GPG 서명 검증 후 설치합니다.
```bash
PACKAGE_FILE=aws-emulator-aarch64.rpm
BASE=https://downloads.aws.com/aws-emulator/packages/aws-emulator/LTS/latest
curl -fsSLO $BASE/$PACKAGE_FILE
curl -fsSLO $BASE/$PACKAGE_FILE.sig
sudo dnf install -y ./$PACKAGE_FILE
aws-emulator -v
```

### 2.2 실행 및 환경 변수 설정
에뮬레이터를 실행한 후, SDK와 CLI가 이를 바라보도록 환경 변수를 설정합니다.
```bash
aws-emulator -t off
```
```bash
export AWS_ENDPOINT_URL=https://127.0.0.1:9297
export AWS_SHARED_CREDENTIALS_FILE=$HOME/.aws/emulator/9297/emulator_credentials
export AWS_CONFIG_FILE=$HOME/.aws/emulator/9297/config
export AWS_CA_BUNDLE=$HOME/.aws/emulator/9297/ca.pem
export AWS_EC2_METADATA_DISABLED=true
export AWS_DISABLE_HOST_PREFIX_INJECTION=true
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE
```

---

## 3. 주요 활용 사례

- **pytest 통합 테스트**: 코드 수정 없이 세션 피스쿠처를 통해 테스트 전후에 자동으로 에뮬레이터를 기동 및 종료할 수 있습니다.
- **자격 증명 없는 CI 작업**: 실제 AWS 자격 증명을 노출할 필요가 없어 포크된 풀 리퀘스트(PR) 환경에서도 안전하게 통합 테스트가 가능합니다.
- **오류 경로 테스트**: 모의 객체(mock)로 재현하기 어려운 DynamoDB 조건부 쓰기 실패나 S3 객체 부재 등의 실제 서비스 오류 응답을 완벽히 재현합니다.
- **Lambda 함수 호출 테스트**: 내장된 RustPython 런타임을 통해 배포 없이 Lambda 함수와 AWS 서비스 간의 호출 흐름을 검증할 수 있습니다.
- **코딩 에이전트 연동**: AI 코딩 에이전트에 자격 증명을 넘기지 않고도 안전하게 반복적인 테스트와 수정 작업을 수행할 수 있습니다.

---

## 4. 적합한 용도와 맞지 않는 용도

| 적합한 용도 | 맞지 않는 용도 |
| :--- | :--- |
| **S3, DynamoDB, SQS** 등 앱 통합 테스트 | **IAM 정책, 버킷 정책** 등 보안 검증 |
| 조건부 쓰기 실패 및 오류 경로 테스트 | 복잡한 CloudFormation, Terraform 배포 검증 |
| 자격 증명이 필요 없는 **CI 환경** 구축 | 성능, 부하 테스트 및 장시간 실행 |
| **코딩 에이전트**의 안전한 반복 작업 | 버킷당 1,000개가 넘는 대량 데이터 처리 |

---

## 5. 팀 도입 시 권고사항
1. **연습 스크립트 활용**: 테스트 스크립트 내에 필수 연결 변수 8개를 정의하고 실제 자격 증명은 배제합니다.
2. **계정 확인 가드 적용**: 테스트 시작 시 계정 ID가 `000000000000`인지 검증하여 실제 AWS로의 오청 전송을 방지합니다.
3. **서비스 버전 관리**: 사용하는 AWS 서비스의 충실도 점수와 에뮬레이터 버전을 일관되게 관리합니다.
    