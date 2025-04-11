# XEP-1004: GoCD에서 Argo Workflows 및 Argo Events로 마이그레이션

## 요약

- xquare platform의 CI/CD 파이프라인을 GoCD에서 Argo Workflows 및 Argo Events로 마이그레이션하여 빌드 및 배포 시간을 대폭 단축하고 Kubernetes 환경과의 통합성을 강화합니다.
- 기존 10분 가량 소요되던 빌드 및 배포 시간을 4분 수준으로 단축하여 개발 생산성을 향상시키고, GoCD에서 발생하던 버그 및 문제점들을 해결합니다.
- Kaniko를 도입하여 Docker in Docker 방식의 보안 취약점과 성능 이슈를 해결하고, 더 안정적인 컨테이너 이미지 빌드 환경을 구축합니다.

## 동기

- GoCD에서 빌드 및 배포 과정이 평균 10분 이상 소요되어 개발 생산성 저하의 원인이 되었습니다.
- GoCD는 Git polling 방식으로 코드 변경을 감지하기 때문에 약 3분 가량의 시간이 단순히 코드 변경을 감지하는 데만 소요되었습니다.
- GoCD의 YAML Plugin에서 발생하는 불안정한 동작과 예측할 수 없는 버그로 인해 파이프라인 관리에 어려움이 있었습니다.
- Kubernetes 환경에서 GoCD는 Cloud Native 아키텍처와의 통합이 원활하지 않아 운영 복잡성을 증가시켰습니다.

### 목표

- CI/CD 파이프라인을 Cloud Native 아키텍처인 Argo Workflows 및 Argo Events로 마이그레이션하여 Kubernetes 통합성을 강화합니다.
- 빌드 및 배포 시간을 10분에서 4분 이하로 단축하여 개발 생산성을 향상시킵니다.
- Kaniko를 도입하여 Docker in Docker 관련 보안 및 성능 이슈를 해결합니다.
- 파이프라인 정의를 Kubernetes Custom Resource(CR)로 관리하여 버전 관리 및 일관성을 확보합니다.
- ArgoCD와의 통합을 통해 GitOps 기반의 배포 프로세스를 확립합니다.

### 목표가 아닌 것

- 기존 GoCD 파이프라인의 로직이나 구조를 그대로 유지하는 것은 목표가 아닙니다.
- 모든 CI/CD 기능을 한번에 마이그레이션하는 것이 아니라 단계적 접근을 취합니다.

## 제안

- Argo Workflows를 CI 도구로 도입하여 빌드 파이프라인을 구현합니다.
- Argo Events를 활용하여 Git 이벤트 기반의 자동화된 워크플로우 트리거링을 구현합니다.
- Git polling 대신 웹훅 기반 이벤트 감지 방식을 도입하여 코드 변경 감지 시간을 약 3분에서 거의 즉각적인 수준으로 단축합니다.
- Kaniko를 활용하여 Docker in Docker 문제를 해결하고, 안정적이고 효율적인 컨테이너 이미지 빌드 환경을 구축합니다.
- 파이프라인 정의를 Kubernetes CR로 관리하여 버전 관리 및 일관성을 확보합니다.
- ArgoCD와의 통합을 통해 GitOps 기반의 배포 프로세스를 확립합니다.

### 유저 스토리 ( 선택 )

**스토리1**
개발자로서, 코드를 저장소에 푸시한 후 빌드 및 배포까지의 시간이 단축되어 더욱 빠른 피드백 사이클을 원합니다. Argo Workflows를 통해 빌드 시간이 크게 단축되어 생산성이 향상됩니다.

**스토리2**
운영자로서, CI/CD 파이프라인 자체를 Kubernetes 리소스로 관리하여 일관된 방식으로 버전 관리하고 싶습니다. Argo Workflows의 CR 기반 정의를 통해 이를 실현할 수 있습니다.

## 검토 설문

### 기능 활성화

어떻게 이 기능을 활성화/비활성화 하나요

- 활성화: Argo Workflows 및 Argo Events Operator를 클러스터에 설치하고, 필요한 CR(Workflow Templates, Sensors 등)을 적용합니다.
- 비활성화: 관련 CR을 삭제하고 필요시 Operator를 제거합니다.

기능을 활성화 하면 기본 동작이 변경되나요?

- 예, CI/CD 파이프라인의 기본 동작이 변경됩니다. 코드 푸시 이벤트가 발생하면 GoCD 대신 Argo Workflows가 트리거되어 빌드 및 배포 과정을 수행합니다.

기능이 활성화되고 롤백할 수 있나요?

- 예, 기존 GoCD 파이프라인을 일정 기간 유지하여 필요시 롤백이 가능하도록 구성합니다. 점진적인 마이그레이션을 통해 위험을 최소화합니다.

기능 활성화/비활성화에 대한 테스트가 있나요?

- 예, 실제 프로젝트에 대한 비교 테스트를 통해 Argo Workflows와 GoCD의 빌드 및 배포 시간, 안정성을 검증했습니다.

### 모니터링 요구사항

운영자가 해당 기능이 사용 중인지 어떻게 확인할 수 있나요?

- Argo Workflows UI를 통해 워크플로우 실행 상태를 실시간으로 확인할 수 있습니다.
- Kubernetes API를 통해 워크플로우 리소스 상태를 확인할 수 있습니다.
- 로그 및 메트릭을 통해 성능 및 상태를 모니터링할 수 있습니다.

이 기능을 위한 합리적인 SLO는 무엇인가요?

- 워크플로우 실행 성공률 99.5% 이상 유지
- 평균 빌드 및 배포 시간 5분 이내 유지
- 워크플로우 시작 지연시간 30초 이내 유지

운영자가 서비스 상태를 확인하기 위해 사용할 수 있는 SLI는 무엇인가요?

- [x] Metrics
  - Metric name: workflow_completion_time, workflow_success_rate
  - 수집 방법: Argo Workflows Prometheus 통합을 통한 메트릭 수집
- [x] Others
  - Argo Workflows UI를 통한 워크플로우 실행 상태 확인
  - Kubernetes 이벤트 로그 분석

### 의존성

이 기능은 특정 서비스에 따라 달라지나요?

- Kubernetes 클러스터
- Git 저장소 (GitHub/GitLab)
- ArgoCD (배포 단계 통합)
- ECR(Elastic Container Registry)

### 확장성

이 기능을 활성화/사용하면 API호출이 발생합니까?

- Kubernetes API 호출이 증가합니다.
- Git 저장소 API 호출 패턴이 변경됩니다(GoCD의 주기적 Git polling 방식에서 Argo Events의 웹훅 기반 이벤트 감지로 변경).

이 기능을 활성화/사용하면 새로운 API 유형이 도입됩니까?

- Argo Workflows API
- Argo Events API

이 기능을 활성화/사용하면 클라우드(AWS) 리소스 변경사항이 존재합니까?

- 새로운 EKS 리소스(Argo Workflows/Events Controller)가 추가됩니다.
- 기존 GoCD 관련 리소스는 단계적으로 제거될 예정입니다.

이 기능을 활성화/사용하면 모든 구성 요소에서 리소스 사용량(CPU, RAM, 디스크, IO 등)이 무시할 수 없을 정도로 증가합니까?

- 오히려 GoCD보다 더 효율적인 리소스 사용이 예상됩니다. Argo Workflows는 필요한 작업만 실행할 때 리소스를 할당하고 해제하는 방식으로 동작합니다.

이 기능을 활성화/사용하면 일부 노드 리소스의 리소스가 소진될 수 있습니까?

- 병렬 워크플로우 실행이 증가할 경우 일시적으로 높은 리소스 사용이 발생할 수 있지만, Kubernetes의 리소스 관리 및 제한 기능을 통해 이를 제어할 수 있습니다.

### 트러블슈팅

해당 기능이 작동하지 않을때 무엇을 확인해야 하나요?

- Argo Workflows/Events Controller의 로그를 확인합니다.
- 워크플로우 실행 상태 및 이벤트 로그를 확인합니다.
- Git 웹훅 설정 및 연결 상태를 확인합니다.
- 워크플로우 템플릿 및 센서 정의가 올바른지 확인합니다.
- ArgoCD와의 통합 상태를 확인합니다.

## 구현 내역

- Argo Workflows 및 Argo Events를 Kubernetes 클러스터에 설치하여 CI/CD 파이프라인 인프라를 구축했습니다.
- Kaniko 기반의 컨테이너 이미지 빌드 워크플로우를 구현하여 Docker in Docker 방식의 문제(보안 이슈, 성능 저하, 권한 문제)를 해결했습니다.
- Git 저장소에 웹훅을 설정하여 코드 변경 시 즉시 빌드 파이프라인이 트리거되도록 구성했습니다.
- Argo Workflows와 ArgoCD 간의 통합을 구현하여 빌드 완료 후 자동으로 배포가 진행되도록 했습니다.
- 워크플로우 템플릿, 이벤트 소스, 센서 등의 CR을 통해 파이프라인을 정의하고 Git 저장소에서 관리하도록 했습니다.

## 단점

- 새로운 시스템으로의 학습 곡선이 존재합니다.
- 기존 GoCD에 익숙한 개발자들은 적응 기간이 필요합니다.
- 복잡한 파이프라인의 경우 Argo Workflows로 마이그레이션하는 과정에서 추가적인 작업이 필요할 수 있습니다.
- 웹훅 기반 트리거 방식은 Git 서버 연결 문제 시 이벤트 누락 가능성이 있어 보완 메커니즘이 필요할 수 있습니다.

## 대안

1. Jenkins X

   - 장점: Kubernetes 네이티브 CI/CD 솔루션
   - 단점: 복잡한 설정, 높은 리소스 요구사항

2. Tekton

   - 장점: 경량화된 Kubernetes 네이티브 파이프라인
   - 단점: 생태계가 성숙 단계, UI 부족

3. GitHub Actions
   - 장점: GitHub와의 긴밀한 통합, 간편한 설정
   - 단점: Kubernetes 환경과의 통합 제한적, 자체 호스팅 복잡성

## 필요한 인프라 ( 선택 )

- Kubernetes 클러스터 (EKS)
- Argo Workflows 및 Argo Events 컨트롤러
- S3 또는 MinIO (워크플로우 아티팩트 저장)
- 서비스 계정 및 RBAC 설정
- Git 저장소(GitHub/GitLab) 웹훅 설정
- ArgoCD 통합을 위한 설정
