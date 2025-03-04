# XEP-1001: Opentelemerty 를 활용한 관측가능성 확보


## 요약
- xquare platform에 배포되어있는 서비스들의 관측가능성을 확보하기 위해 여러 플랫폼과 통합이 가능한 Opentelemetry 를 사용하며, 내부 MongoDB에 저장하는 Exporter를 개발하여 속도와 자율성을 보장합니다.

## 동기
- 빠른 트러블슈팅을 위해서는 디버깅없이 추적이 가능해야 하며 분산 시스템에서는 심층적인 가시성을 확보하여 더 빠른 문제해결을 촉진합니다.

### 목표
- 기존 Deployment의 구조를 변경하지 않고 관측가능성을 확보합니다.
- Trace를 수집하여 Xquare Platform Web에 시각화할 수 있도록 합니다.
- 쿼리 속도를 통해 데이터 실시간으로 볼 수 있도록 합니다.
### 목표가 아닌 것
- 프론트앤드 관측가능성을 확보하지 않습니다.

## 제안
- 기존 코드를 변경하지 않아도 auto-intrumentation 이 제공되며 여러 서드파티 플랫폼과 통합이 이로운 Opentelemery 를 사용하여 관측 가능성을 측정합니다.
- Opentelemetry Protocol에 맞게 Trace exporter를 개발하여 내부 MongoDB에 데이터를 저장합니다.

### 유저 스토리 ( 선택 )

**스토리1**

**스토리2**

## 검토 설문


### 기능 활성화

어떻게 이 기능을 활성화/비활성화 하나요 <br>
(k8s-resource 레포지토리를 통해 Opentelemetry Agent가 배포되어 있다는 것을 가정합니다.)
- 관측가능성이 필요한 namespace에 Opentelemetry Instrumentation Object를 추가합니다. (k8s-resource xquare-club에 존재)
```yaml
{{- range .Values.clubs }}
{{- $clubname := .name }}
---
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation-{{ $clubname }}-prod
  namespace: {{ $clubname }}-prod
spec:
  exporter:
    endpoint: "http://$(OTEL_NODE_IP):4317"
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
---
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation-{{ $clubname }}-stag
  namespace: {{ $clubname }}-stag
spec:
  exporter:
    endpoint: "http://$(OTEL_NODE_IP):4317"
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
{{- end }}
```
- 관측가능성을 확보할 Deployment에 어노테이션을 추가합니다.
```yaml
  template:
    metadata:      
      annotations:
        sidecar.istio.io/rewriteAppHTTPProbers: "false"
        sidecar.istio.io/proxyCPU: "5m"
        sidecar.istio.io/proxyMemory: "128Mi"
        {{- if eq .Values.language "java"}}
        instrumentation.opentelemetry.io/inject-java: "true"
        {{- end }}
        {{- if eq .Values.language "nodejs" }}
        instrumentation.opentelemetry.io/inject-nodejs: "true"
        {{- end }}
```

기능을 활성화 하면 기본 동작이 변경되나요?
- Deployment에서 배포되는 Pod에 auto-inject container가 추가되며 서비스에서 남기는 Trace를 수집하고, mongodb-exporter 를 통해 MongoDB에 Trace정보가 저장됩니다. 

기능이 활성화되고 롤백할 수 있나요?
- Deployment의 annotation 을 삭제합니다.

기능 활성화/비활성화에 대한 테스트가 있나요?


### 모니터링 요구사항

운영자가 해당 기능이 사용 중인지 어떻게 확인할 수 있나요?
- auto-inject container가 추가됐는지 확인하고, MongoDB에 저장되는지, Xquare Web Trace 페이지에 조회되는지 확인합니다.

이 기능을 위한 합리적인 SLO는 무엇인가요?
- MongoDB의 가용성을 95% 이상으로 유지합니다.

운영자가 서비스 상태를 확인하기 위해 사용할 수 있는 SLI는 무엇인가요?

- [ ]  Metrics
    - Metric name: MongoDB Availability, MongoDB Available Storage
    - 수집 방법 : Prometheus
- [ ]  Others
    - Details

### 의존성

이 기능은 특정 서비스에 따라 달라지나요?

### 확장성

이 기능을 활성화/사용하면 API호출이 발생합니까?
- Service에서 Opentelemetry Agent로 데이터를 전송하는 API 호출이 발생합니다. <br>
- Opentelemetry Agent에서 mongodb-exporter로 데이터를 전송하는 grpc protocol의 API 호출이 발생합니다. <br>
이 기능을 활성화/사용하면 새로운 API 유형이 도입됩니까?

이 기능을 활성화/사용하면 클라우드(AWS) 리소스 변경사항이 존재합니까?

이 기능을 활성화/사용하면 모든 구성 요소에서 리소스 사용량(CPU, RAM, 디스크, IO 등)이 무시할 수 없을 정도로 증가합니까?
- 모든 노드에 Daemonset으로 Opentelemetry Collector가 생성됩니다. <br>

이 기능을 활성화/사용하면 일부 노드 리소스의 리소스가 소진될 수 있습니까?

### 트러블슈팅

해당 기능이 작동하지 않을때 무엇을 확인해야 하나요?
- OpenTelemetry Agent가 정상적으로 올라가 있으며, 로그를 확인하여 올바르게 데이터를 수집하고 있는지 확인합니다.

## 구현 내역
- Opentelemetry 배포
```yaml
apiVersion: v2
name: opentelemetry
type: application
version: 1.0.0
dependencies:
  - name: "opentelemetry-operator"
    version: 0.62.0
    repository: "https://open-telemetry.github.io/opentelemetry-helm-charts"
  - name: "opentelemetry-collector"
    version: 0.93.3
    repository: "https://open-telemetry.github.io/opentelemetry-helm-charts"
```
```yaml
opentelemetry-operator:
  manager:
    collectorImage:
      repository: "otel/opentelemetry-collector-contrib"
  tolerations:
    - effect: "NoSchedule"
      key: xquare/platform
      operator: "Equal"
      value: "true"

opentelemetry-collector:
  ports:
    metrics:
      # The metrics port is disabled by default. However you need to enable the port
      # in order to use the ServiceMonitor (serviceMonitor.enabled) or PodMonitor (podMonitor.enabled).
      enabled: true
      containerPort: 8889
      servicePort: 8889
      protocol: TCP
  tolerations:
    - operator: "Exists"
  image:
    repository: "otel/opentelemetry-collector-contrib"
  mode: "daemonset"
  config:
    exporters:
      otlp/go:
        endpoint: grpc-nginx-service.xquare-prod.svc.cluster.local:80
        tls:
          insecure: true
        compression: none
      prometheus:
        endpoint: "0.0.0.0:8889"
    extensions:
      health_check: {}
      memory_ballast: {}
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
      batch:
        timeout: 1s
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: ${env:MY_POD_IP}:4317
          http:
            endpoint: ${env:MY_POD_IP}:4318
    service:
      extensions:
        - health_check
        - memory_ballast
      pipelines:
        traces:
          exporters:
            - otlp/go
          processors:
            - memory_limiter
            - batch
          receivers:
            - otlp
        metrics:
          receivers:
            - otlp
            - prometheus
          processors:
            - memory_limiter
            - batch
          exporters:
            - prometheus
```

- gRPC 서버 배포 (k8s Service가 gRPC Protocol 을 지원하지 않는 문제를 해결하는 nginx pod는 생략합니다.)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: otel-trace-reciever-be-prod
    argocd.argoproj.io/instance: otel-trace-reciever-be-prod
    environment: prod
    project: otel-trace-reciever
    type: be
  name: otel-trace-reciever-be-prod
  namespace: xquare-prod
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app: otel-trace-reciever-be-prod
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      annotations:
        sidecar.istio.io/proxyCPU: 5m
        sidecar.istio.io/proxyMemory: 128Mi
        sidecar.istio.io/rewriteAppHTTPProbers: 'false'
      labels:
        app: otel-trace-reciever-be-prod
        project: otel-trace-reciever
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - otel-trace-reciever-be-prod
              topologyKey: kubernetes.io/hostname
      containers:
        - image: >-
            786584124104.dkr.ecr.ap-northeast-2.amazonaws.com/otel-trace-reciever-be-prod:prod-52117a2820946e016de22a172bd247dda2724ba2
          env:
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongo-uri
          imagePullPolicy: Always
          name: otel-trace-reciever-be-prod
          ports:
            - containerPort: 4317
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            initialDelaySeconds: 20
            periodSeconds: 10
            successThreshold: 3
            tcpSocket:
              port: 4317
            timeoutSeconds: 1
          resources:
            limits:
              memory: 2000Mi
            requests:
              cpu: 20m
              ephemeral-storage: 20Mi
              memory: 700Mi
      dnsPolicy: ClusterFirst
      nodeSelector:
        Karpenter: enabled
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 120
      tolerations:
        - effect: NoSchedule
          key: xquare/server
          operator: Equal
          value: 'true'
```
- club namespace에 Instrumentation 을 추가합니다.
```yaml
{{- range .Values.clubs }}
{{- $clubname := .name }}
---
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation-{{ $clubname }}-prod
  namespace: {{ $clubname }}-prod
spec:
  exporter:
    endpoint: "http://$(OTEL_NODE_IP):4317"
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
---
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation-{{ $clubname }}-stag
  namespace: {{ $clubname }}-stag
spec:
  exporter:
    endpoint: "http://$(OTEL_NODE_IP):4317"
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
{{- end }}
```
## 단점
- MongoDB on Kubernetes를 사용하고 있어 Storage 확장의 어려움이 있습니다.
- 현재 Xquare Platform Web의 Trace 조회 API가 비약하여 조회 속도에 한계가 있습니다.

## 대안


## 필요한 인프라 ( 선택 )
