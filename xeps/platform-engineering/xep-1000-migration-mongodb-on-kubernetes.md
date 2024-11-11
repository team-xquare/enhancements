# XEP-1000: MongoDB on Kubernetes 환경으로 마이그레이션


## 요약
기존 DocumentDB 에서 운영되고 있던 MongoDB를 Kubernetes 환경으로 마이그레이션 합니다.

## 동기
첫번째 이유는 비용 입니다. DocumentDB를 사용한 이유는 해당 시절에 안정적으로 운영되었어야 하는 서비스 때문에 선택했지만, 현재는 MongoDB를 Kubernetes 환경으로 마이그레이션하여 비용을 절감하고자 합니다.
두번째 이유는 확장성 입니다. DocumentDB는 확장성이 떨어지는 편이라서, MongoDB를 Kubernetes 환경으로 마이그레이션하여 확장성을 높이고자 합니다.

### 목표
- DocumentDB에서 MongoDB로의 마이그레이션
- Kubernetes 환경에서 MongoDB 운영
- 기존 데이터 손실 없이 마이그레이션
- DocumentDB를 사용하지 않는 것

### 목표가 아닌 것
- XQUARE Infra SPAN 데이터를 마이그레이션 하는 것
- 다운타임 없이 이전하는 것(MongoDB를 사용하는 서비스가 적음에 따라 마이그레이션 비용을 낮춤)

## 제안
- 무료 소프트웨어이면서, 이용이 편리하고, 자동화 기능이 많은 Percona Operator를 통해 Kubernetes에서 MongoDB를 운영합니다.

### 유저 스토리 ( 선택 )

**스토리1**

**스토리2**

## 검토 설문


### 기능 활성화

**어떻게 이 기능을 활성화/비활성화 하나요**<br>
k8s-resource 레포지토리에 Percona Helm Chart를 추가해 활성화

**기능을 활성화 하면 기본 동작이 변경되나요?**<br>
DocumentDB를 사용하지 않게 됨

**기능이 활성화되고 롤백할 수 있나요?**<br>
DocumentDB의 Snapshot을 남겨, 롤백 가능

**기능 활성화/비활성화에 대한 테스트가 있나요?**<br>
없음

### 모니터링 요구사항

**운영자가 해당 기능이 사용 중인지 어떻게 확인할 수 있나요?** <br>
k8s-resource에 Percona Operator가 존재하는지 확인

**이 기능을 위한 합리적인 SLO는 무엇인가요?**<br>
현재 MongoDB는 REPO, XQUARE INFRA가 사용되고 있으며 해당 서비스의 장애를 방지하기 위해 가용성 99.9% 이상을 목표로 함

**운영자가 서비스 상태를 확인하기 위해 사용할 수 있는 SLI는 무엇인가요?**

- [ ]  Metrics
    - Metric name: Availability ( 성공요청수 / 성공요청수 + 실패요청수 )
    - 수집 방법
      - Prometheus를 사용하여 Scraping 하여 수집
- [ ]  Others
    - Details

### 의존성

이 기능은 특정 서비스에 따라 달라지나요?<br>
아니요
### 확장성

**이 기능을 활성화/사용하면 API호출이 발생합니까?**<br>
아니요

**이 기능을 활성화/사용하면 새로운 API 유형이 도입됩니까?**<br>
아니요

**이 기능을 활성화/사용하면 클라우드(AWS) 리소스 변경사항이 존재합니까?**<br>
DocumentDB가 삭제됩니다.

**이 기능을 활성화/사용하면 모든 구성 요소에서 리소스 사용량(CPU, RAM, 디스크, IO 등)이 무시할 수 없을 정도로 증가합니까?**<br>
Kubernetes의 EC2 사용량이 대폭 증가합니다. 하지만 DocumentDB의 비용을 절감할 수 있습니다.

**이 기능을 활성화/사용하면 일부 노드 리소스의 리소스가 소진될 수 있습니까?**<br>
일부 노드의 리소스를 소진하지 않기 위해 Taint, Toleration 설정을 통해 DB 관리용 노드를 분리합니다.

### 트러블슈팅

**해당 기능이 작동하지 않을때 무엇을 확인해야 하나요?**<br>
먼저 Percona Operator가 k8s-resource 레포지토리에 존재하는지 확인하고, MongoDB Pod가 정상적으로 생성되었는지 확인합니다.

## 구현 내역
CRD를 수동으로 설치
```shell
kubectl apply --server-side -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/main/deploy/bundle.yaml
```
Percona Operator를 k8s-resource 레포지토리에 Helm으로 설치 <br>
Chart.yaml
```yaml
apiVersion: v2
name: percona-operator
description: Percona Operator
type: application
version: 1.0.0
dependencies:
   - name: psmdb-db
     version: 1.17.1
     repository: https://percona.github.io/percona-helm-charts
```
values.yaml
```yaml
psmdb-db:
  tolerations:
    - effect: "NoSchedule"
      key: xquare/platform
      operator: "Equal"
      value: "true"
```
operator는 platform 노드에서 작동하도록 하고, 실제 배포되는 DB는 database 노드에 올라가도록 한다.

실제로 배포되도록 하기 위해서 charts/xquare-application/values.yaml에 다음과 같이 추가하고 차트 버전을 올린다.
```yaml
      - name: percona-operator
        namespace: percona
        source:
          path: apps/percona-operator
          repoURL: https://github.com/team-xquare/k8s-resource.git
        syncPolicy:
          automated:
            prune: false
            selfHeal: true
```

## 단점


## 대안


## 필요한 인프라 ( 선택 )