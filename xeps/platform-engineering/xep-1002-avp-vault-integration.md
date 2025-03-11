# XEP-1002: AVP를 활용한 Vault 시크릿 통합

## 요약

- xquare platform에서 ArgoCD 배포 시, AVP(Argocd Vault Plugin)를 사용하여 Vault에서 시크릿을 가져와 리소스에 자동으로 값을 대체하는 기능을 제공합니다.

## 동기

- Kubernetes Secret을 사용하면 파일로 관리해야 하므로 시크릿 관리 보안이 취약해지며, 매번 시크릿을 수동으로 등록하는 작업은 번거롭고 오류가 발생하기 쉽습니다.
- 중앙 집중식 시크릿 관리 솔루션을 통해 보안을 강화하고 GitOps 워크플로우를 유지하면서 시크릿을 관리할 수 있는 방법이 필요합니다.

### 목표

- Vault를 사용하여 시크릿을 중앙에서 관리합니다.
- ArgoCD 배포 과정에서 AVP를 통해 Vault의 시크릿을 자동으로 가져와 적용합니다.
- GitOps 워크플로우 내에서 시크릿 관리를 자동화하여 보안성과 편의성을 개선합니다.
- 시크릿 관리 과정에서 민감한 정보가 Git에 노출되지 않도록 합니다.

### 목표가 아닌 것

- 기존 Kubernetes Secret의 완전한 대체가 아닌 보완적인 역할을 합니다.

## 제안

- ArgoCD에 AVP를 설치하고 Vault와 연동하여 시크릿을 자동으로 가져올 수 있는 환경을 구성합니다.
- AVP의 어노테이션 기반 시크릿 참조 방식을 활용하여 YAML 파일에서 Vault 시크릿을 참조하도록 합니다.
- ArgoCD 배포 과정에서 AVP가 Vault에서 시크릿을 가져와 매니페스트에 실제 값을 대체합니다.

### 유저 스토리 ( 선택 )

운영자는 Vault에서 시크릿을 중앙에서 관리하며, 필요한 경우 시크릿 값을 변경합니다. 변경된 시크릿은 다음 ArgoCD 동기화 시 자동으로 적용되어 애플리케이션에 반영됩니다.

## 검토 설문

### 기능 활성화

어떻게 이 기능을 활성화/비활성화 하나요

- ArgoCD에 AVP를 설치하고 Vault와 연동하여 활성화합니다.
- ArgoCD 배포에 AVP 어노테이션을 추가하여 시크릿을 참조합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
  annotations:
    avp.kubernetes.io/path: "secret/data/example"
spec:
  template:
    spec:
      containers:
        - name: example
          env:
            - name: SECRET_VALUE
              value: <path:secret/data/example#key>
```

- AVP를 제거하거나 어노테이션을 제거하여 비활성화할 수 있습니다.

기능을 활성화 하면 기본 동작이 변경되나요?

- ArgoCD 배포 과정에 AVP 플러그인이 추가되어 배포 전에 Vault에서 시크릿을 가져와 매니페스트에 실제 값을 대체하는 단계가 추가됩니다.
- 배포된 최종 리소스에는 실제 시크릿 값이 포함되어 있습니다.

기능이 활성화되고 롤백할 수 있나요?

- AVP 어노테이션을 제거하고 기존 Kubernetes Secret 방식으로 되돌릴 수 있습니다.
- 롤백 시에는 수동으로 Kubernetes Secret을 생성해야 합니다.

기능 활성화/비활성화에 대한 테스트가 있나요?

- AVP 통합 테스트를 위한 테스트 애플리케이션과 시크릿을 준비하여 실제 동작을 검증합니다.
- ArgoCD 배포 과정에서 AVP가 정상적으로 시크릿을 가져오는지 테스트합니다.

### 모니터링 요구사항

운영자가 해당 기능이 사용 중인지 어떻게 확인할 수 있나요?

- ArgoCD UI에서 애플리케이션 배포 로그를 확인하여 AVP가 시크릿을 대체했는지 확인합니다.
- ArgoCD Pod 로그에서 AVP 관련 로그를 확인하여 정상 작동 여부를 확인합니다.

이 기능을 위한 합리적인 SLO는 무엇인가요?

- Vault 가용성 99.9% 이상 유지
- AVP 시크릿 대체 성공률 99.5% 이상 유지

운영자가 서비스 상태를 확인하기 위해 사용할 수 있는 SLI는 무엇인가요?

- [x] Metrics
  - Metric name: AVP Secret Substitution Success Rate
  - 수집 방법: ArgoCD 로그 분석 및 Prometheus 메트릭
- [x] Others
  - Details: ArgoCD 배포 성공률 모니터링

### 의존성

이 기능은 특정 서비스에 따라 달라지나요?

- Vault 서비스에 의존적입니다.
- ArgoCD 서비스에 의존적입니다.

### 확장성

이 기능을 활성화/사용하면 API호출이 발생합니까?

- ArgoCD 배포 과정에서 AVP가 Vault API를 호출하여 시크릿을 가져옵니다.

이 기능을 활성화/사용하면 새로운 API 유형이 도입됩니까?

- 새로운 API 유형은 도입되지 않습니다. 기존 Vault API를 활용합니다.

이 기능을 활성화/사용하면 클라우드(AWS) 리소스 변경사항이 존재합니까?

- 없습니다. 기존 클러스터 내에서 운영됩니다.

이 기능을 활성화/사용하면 모든 구성 요소에서 리소스 사용량(CPU, RAM, 디스크, IO 등)이 무시할 수 없을 정도로 증가합니까?

- ArgoCD Pod에 AVP 플러그인이 추가되어 약간의 리소스 사용량 증가가 있지만 무시할 수 있는 수준입니다.

이 기능을 활성화/사용하면 일부 노드 리소스의 리소스가 소진될 수 있습니까?

- 아니요, 리소스 사용량 증가가 미미하여 노드 리소스가 소진될 가능성은 낮습니다.

### 트러블슈팅

해당 기능이 작동하지 않을때 무엇을 확인해야 하나요?

- ArgoCD Pod 로그에서 AVP 관련 오류를 확인합니다.
- Vault 서비스 상태 및 접근 권한을 확인합니다.
- AVP 어노테이션이 올바르게 설정되었는지 확인합니다.
- Vault에 해당 시크릿이 존재하는지 확인합니다.
- ArgoCD와 Vault 사이의 네트워크 연결을 확인합니다.

## 구현 내역

- ArgoCD에 AVP 설치

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-plugin
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: argocd-vault-plugin-helm
    spec:
      allowConcurrency: true
      discover:
        find:
          command:
            - sh
            - "-c"
            - "find . -name 'Chart.yaml' && find . -name 'values.yaml'"
      init:
        command:
          - bash
          - "-c"
          - |
            helm repo add bitnami https://charts.bitnami.com/bitnami
            helm dependency build
      generate:
        command:
          - sh
          - "-c"
          - |
            helm template $ARGOCD_APP_NAME -n $ARGOCD_APP_NAMESPACE ${ARGOCD_ENV_HELM_ARGS} . --include-crds |
            argocd-vault-plugin generate -s argocd:argocd-vault-plugin-credentials -
      lockRepo: false
```

- ArgoCD에 Vault 연결 설정

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-vault-plugin-credentials
  namespace: argocd
type: Opaque
stringData:
  AVP_AUTH_TYPE: "k8s"
  AVP_K8S_ROLE: "argocd"
  AVP_TYPE: "vault"
  VAULT_ADDR: "http://vault.vault.svc.cluster.local:8200"
```

- ArgoCD 애플리케이션에 AVP 적용

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-app
  namespace: argocd
spec:
  source:
    plugin:
      name: argocd-vault-plugin
  # 기타 설정...
```

- YAML 파일에 AVP 어노테이션 추가

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
  annotations:
    avp.kubernetes.io/path: "secret/data/example"
type: Opaque
stringData:
  username: <path:secret/data/example#username>
  password: <path:secret/data/example#password>
```

## 단점

- ArgoCD 배포 과정에 추가적인 단계가 포함되어 배포 시간이 약간 증가할 수 있습니다.
- AVP와 Vault 사이의 연결 문제가 발생할 경우 배포가 실패할 수 있습니다.
- Vault 서비스가 중단되면 새로운 배포가 불가능해질 수 있습니다.
- 추가적인 관리 포인트(Vault)가 생깁니다.

## 대안

1. Sealed Secrets

   - 장점: 암호화된 시크릿을 Git에 저장 가능
   - 단점: 중앙 집중식 관리가 어려움, 시크릿 업데이트 프로세스가 복잡함

2. External Secrets Operator

   - 장점: 다양한 시크릿 관리 시스템과 통합 가능
   - 단점: 추가적인 오퍼레이터 관리 필요, 설정이 복잡함

3. SOPS (Secrets OPerationS)
   - 장점: 파일 기반 암호화 지원, Git과 통합 용이
   - 단점: 키 관리 부담, 디코딩 로직이 별도로 필요함

## 필요한 인프라 ( 선택 )

- Vault 서비스: 시크릿을 저장하고 관리하기 위한 서비스
- ArgoCD: GitOps 기반 배포 관리 도구
- ArgoCD Vault Plugin (AVP): ArgoCD와 Vault를 연동하는 플러그인
