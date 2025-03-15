# XEP-1003: Vault Auto-Unseal with AWS KMS

## 요약

- xquare platform의 Vault 서비스에 AWS KMS를 활용한 Auto-Unseal 기능을 구현하여 서버 재시작 시 자동으로 Unseal되도록 구성합니다. 이를 통해 운영 편의성과 가용성을 향상시킵니다.

## 동기

- Vault는 초기 시작 또는 재시작 시 Unseal 키를 수동으로 입력해야 하는 과정이 필요합니다. 이 과정은 운영자의 개입을 필요로 하며, 특히 시스템 장애 상황에서 복구 시간을 지연시키는 요인이 됩니다.
- 클러스터 환경에서 Vault pod가 재시작될 때마다 수동 Unseal 작업은 운영 부담을 가중시키고 가용성을 저하시킵니다.

### 목표

- AWS KMS를 활용하여 Vault Auto-Unseal을 구현합니다.
- 서버 재시작 시 자동으로 Unseal되도록 하여 운영 편의성을 향상시킵니다.
- 고가용성 Vault 클러스터 운영을 위한 기반을 마련합니다.
- IAM Role을 통한 안전한 인증 체계를 구성합니다.

### 목표가 아닌 것

- Vault 클러스터의 확장(HA 구성)은 이 제안의 범위에 포함되지 않습니다.

## 제안

- AWS KMS를 Vault의 Auto-Unseal 메커니즘으로 활용합니다.
- Terraform을 통해 필요한 AWS 리소스(KMS 키, IAM 역할 등)를 프로비저닝합니다.
- Kubernetes의 ServiceAccount와 AWS IAM 역할을 연동하여 Vault가 KMS에 안전하게 접근할 수 있도록 합니다.
- Helm 차트를 통해 Vault 서버에 Auto-Unseal 구성을 적용합니다.

### 유저 스토리 ( 선택 )

**스토리1**
운영자로서, Vault 서버가 재시작되더라도 수동 개입 없이 자동으로 Unseal되기를 원합니다. 이를 통해 장애 상황에서도 빠른 복구가 가능하고 운영 부담이 줄어듭니다.

**스토리2**
보안 담당자로서, Vault의 Unseal 키가 안전하게 관리되고, 접근 권한이 엄격하게 제어되기를 원합니다. AWS KMS를 통한 관리는 키 관리의 보안성을 향상시킵니다.

## 검토 설문

### 기능 활성화

어떻게 이 기능을 활성화/비활성화 하나요

- 활성화: Terraform을 통해 AWS KMS 키와 IAM 역할을 생성하고, Vault Helm 차트에 Auto-Unseal 관련 환경 변수를 설정합니다.
- 비활성화: Vault Helm 차트에서 Auto-Unseal 관련 환경 변수를 제거하고 기존 방식으로 되돌립니다.

기능을 활성화 하면 기본 동작이 변경되나요?

- 네, Vault의 시작 동작이 변경됩니다. 서버 재시작 시 AWS KMS를 통해 자동으로 Unseal 과정이 진행되어 수동 개입이 필요 없게 됩니다.

기능이 활성화되고 롤백할 수 있나요?

- 예, 롤백 가능합니다. Vault 구성을 변경하여 KMS Auto-Unseal을 비활성화하고 수동 Unseal 방식으로 되돌릴 수 있습니다.
- 주의사항: 롤백 시 기존 Unseal 키를 보관하고 있어야 합니다.

기능 활성화/비활성화에 대한 테스트가 있나요?

- Vault 서버를 재시작하여 자동으로 Unseal되는지 테스트합니다.
- 다양한 오류 상황(IAM 권한 문제, KMS 접근 실패 등)에서의 동작 테스트가 필요합니다.

### 모니터링 요구사항

운영자가 해당 기능이 사용 중인지 어떻게 확인할 수 있나요?

- Vault 서버 로그에서 "awskms" 관련 로그를 확인하여 Auto-Unseal이 활성화되어 있는지 확인할 수 있습니다.
- AWS CloudTrail에서 KMS 키 사용 로그를 확인하여 Vault의 Unseal 작업을 모니터링할 수 있습니다.

이 기능을 위한 합리적인 SLO는 무엇인가요?

- Vault 서버 재시작 후 Auto-Unseal 성공률 99.9% 이상
- Auto-Unseal 과정 완료 시간 30초 이내

운영자가 서비스 상태를 확인하기 위해 사용할 수 있는 SLI는 무엇인가요?

- [x] Metrics
  - Metric name: vault_core_unsealed
  - 수집 방법: Prometheus를 통해 Vault 메트릭 수집
- [x] Others
  - AWS CloudTrail의 KMS 키 사용 로그
  - Vault 서버의 시작 로그에서 Unseal 관련 메시지

### 의존성

이 기능은 특정 서비스에 따라 달라지나요?

- AWS KMS 서비스에 의존적입니다.
- AWS IAM 서비스에 의존적입니다.
- EKS IRSA(IAM Roles for Service Accounts) 기능에 의존적입니다.

### 확장성

이 기능을 활성화/사용하면 API호출이 발생합니까?

- 예, Vault 서버가 시작되거나 재시작될 때 AWS KMS API 호출이 발생합니다.

이 기능을 활성화/사용하면 새로운 API 유형이 도입됩니까?

- 아니요, 기존 AWS KMS API를 사용합니다.

이 기능을 활성화/사용하면 클라우드(AWS) 리소스 변경사항이 존재합니까?

- 예, 다음과 같은 AWS 리소스가 생성됩니다:
  - AWS KMS 키
  - IAM 역할 및 정책

이 기능을 활성화/사용하면 모든 구성 요소에서 리소스 사용량(CPU, RAM, 디스크, IO 등)이 무시할 수 없을 정도로 증가합니까?

- 아니요, Auto-Unseal은 Vault 서버 시작 시에만 실행되므로 지속적인 리소스 사용량 증가는 없습니다.

이 기능을 활성화/사용하면 일부 노드 리소스의 리소스가 소진될 수 있습니까?

- 아니요, Auto-Unseal 과정은 리소스 집약적이지 않으며 일시적으로만 실행됩니다.

### 트러블슈팅

해당 기능이 작동하지 않을때 무엇을 확인해야 하나요?

1. Vault 서버 로그에서 Auto-Unseal 관련 오류 메시지를 확인합니다.
2. IAM 역할 설정이 올바른지 확인합니다.
3. ServiceAccount에 IAM 역할 주석(annotation)이 올바르게 설정되었는지 확인합니다.
4. KMS 키에 대한 권한이 올바르게 설정되었는지 확인합니다.
5. AWS KMS 서비스 상태를 확인합니다.
6. 네트워크 연결 상태(Vault Pod에서 AWS API 엔드포인트로의 연결)를 확인합니다.

## 구현 내역

1. **AWS KMS 키 및 IAM 역할 생성 (Terraform)**

```terraform
# Vault Auto-Unseal KMS 설정
resource "aws_iam_role" "vault_kms_role" {
  name                  = "vault-kms-role"
  description           = "Vault IAM role for KMS auto-unseal"
  path                  = "/"
  assume_role_policy    = data.aws_iam_policy_document.vault_assume_role_policy.json
  force_detach_policies = true
}

data "aws_iam_policy_document" "vault_assume_role_policy" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [module.eksv2.oidc_provider_arn]
    }

    condition {
      test     = "StringEquals"
      variable = "${local.irsa_oidc_provider_url}:sub"
      values   = ["system:serviceaccount:vault:vault"]
    }
  }
}

resource "aws_iam_policy" "vault_kms_policy" {
  name   = "vault-kms-policy"
  policy = data.aws_iam_policy_document.vault_kms_policy_document.json
}

data "aws_iam_policy_document" "vault_kms_policy_document" {
  statement {
    actions = [
      "kms:Encrypt",
      "kms:Decrypt",
      "kms:DescribeKey"
    ]
    resources = [
      aws_kms_key.vault_unseal_key.arn
    ]
    effect = "Allow"
  }
}

resource "aws_iam_role_policy_attachment" "vault_kms_role_attachment" {
  policy_arn = aws_iam_policy.vault_kms_policy.arn
  role       = aws_iam_role.vault_kms_role.name
}

resource "aws_kms_key" "vault_unseal_key" {
  description             = "KMS Key for Vault Auto-Unseal"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Name = "vault-auto-unseal-key"
  }
}

resource "aws_kms_alias" "vault_unseal_key_alias" {
  name          = "alias/vault-auto-unseal-key"
  target_key_id = aws_kms_key.vault_unseal_key.key_id
}
```

2. **Vault Helm 차트 설정**

```yaml
vault:
  injector:
    enabled: true
    resources:
      requests:
        memory: 60Mi
        cpu: 80m
      limits:
        memory: 120Mi
        cpu: 120m
    tolerations:
      - effect: "NoSchedule"
        key: xquare/platform
        operator: "Equal"
        value: "true"
    affinity: null
  server:
    enabled: true
    resources:
      requests:
        memory: 200Mi
        cpu: 100m
    tolerations:
      - effect: "NoSchedule"
        key: xquare/critical_platform
        operator: "Equal"
        value: "true"
    affinity: null
    dataStorage:
      enabled: false
    volumes:
      - name: vault-storage
        persistentVolumeClaim:
          claimName: vault-pvc
    volumeMounts:
      - mountPath: "/vault/data"
        name: vault-storage

    extraEnvironmentVars:
      VAULT_SEAL_TYPE: awskms
      AWS_REGION: ap-northeast-2
      VAULT_AWSKMS_SEAL_KEY_ID: alias/vault-auto-unseal-key

    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "arn:aws:iam::786584124104:role/vault-kms-role"

  pod:
    affinity: null
    tolerations:
      - effect: "NoSchedule"
        key: xquare/platform
        operator: "Equal"
        value: "true"
  ui:
    affinity: null
    enabled: true
    serviceType: "ClusterIP"
    externalPort: 8200
```

3. **설정 적용 과정**

- Terraform을 통해 AWS 리소스(KMS 키, IAM 역할) 생성
- Helm 차트를 통해 Vault 서버에 Auto-Unseal 구성 적용
- Vault 서버 재시작 후 Auto-Unseal 동작 확인

## 단점

- AWS KMS 서비스에 대한 의존성이 생깁니다.
- KMS API 호출에 따른 약간의 비용이 발생할 수 있습니다.
- AWS KMS 서비스 장애 시 Vault의 Auto-Unseal이 실패할 수 있습니다.
- 초기 설정 복잡성이 증가합니다.

## 대안

1. **Shamir 시크릿 공유 방식**

   - 장점: 외부 서비스 의존성 없음
   - 단점: 수동 Unseal 필요, 운영 부담 증가

2. **Transit Seal (다른 Vault 사용)**

   - 장점: 중앙 집중식 키 관리 가능
   - 단점: 추가 Vault 클러스터 필요, 설정 복잡성 증가

## 필요한 인프라 ( 선택 )

- AWS KMS 서비스
- AWS IAM 서비스
- EKS 클러스터(IRSA 기능 활성화)
