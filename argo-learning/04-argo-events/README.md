# Argo Events: The Event-Driven Dependency Manager

Argo Events는 Kubernetes를 위한 **이벤트 기반 워크플로우 자동화 프레임워크**입니다.
다양한 소스(GitHub, S3, SNS, Cron, PubSub 등)에서 이벤트를 수신하고, 이를 필터링하여 K8s 리소스(주로 Argo Workflow)를 트리거합니다.

---

## 1. 아키텍처 및 데이터 흐름

데이터는 다음과 같은 파이프라인을 타고 흐릅니다.

`Event Source` -> `EventBus` -> `Sensor` -> `Trigger`

1.  **Event Source (귀)**: 외부 이벤트를 리스닝합니다.
    *   예: "HTTP 포트 12000을 열고 기다린다." 또는 "AWS SQS 큐를 계속 쳐다본다."
2.  **Event Bus (신경망)**: 들어온 이벤트를 메시지 큐(NATS Streaming, JetStream)에 태웁니다. 센서들이 이 버스를 구독합니다.
3.  **Sensor (뇌)**: 버스에서 이벤트를 꺼내 조건을 검사(Filter)합니다.
    *   예: "Github Webhook이 왔는데, 브랜치가 `main`인 경우에만 통과시킨다."
4.  **Trigger (손/발)**: 조건이 맞으면 액션을 취합니다.
    *   예: "CI 워크플로우를 실행해라." 또는 "슬랙으로 메시지를 보내라."

---

## 2. 주요 CRD 명세 (Deep Dive)

### EventSource (`kind: EventSource`)
이벤트를 어디서 받을지 정의합니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
spec:
  # GitHub Webhook 예시
  github:
    my-repo-webhook:
      owner: "my-org"
      repository: "my-app"
      events: ["push", "pull_request"]
      secretToken:
        name: github-secret
        key: token
```

### Sensor (`kind: Sensor`)
어떤 이벤트에 반응할지, 파라미터를 어떻게 넘길지 정의합니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
spec:
  dependencies:
  - name: test-dep
    eventSourceName: webhook
    eventName: example
  triggers:
  - template:
      name: workflow-trigger
      k8s:
        operation: create
        source:
          resource:
            apiVersion: argoproj.io/v1alpha1
            kind: Workflow
            # ... (워크플로우 정의) ...
        parameters:
        - src:
            dependencyName: test-dep
            dataKey: body.head_commit.id # Webhook JSON Payload에서 커밋 해시 추출
          dest: spec.arguments.parameters.0.value # 워크플로우 파라미터로 주입
```

---

## 3. 핵심 기능

1.  **다양한 이벤트 소스 지원**: AWS(SNS, SQS, S3), GCP(PubSub), Azure(EventHub), Kafka, NATS, File, HDFS, Slack, Stripe 등 20개 이상.
2.  **복합 트리거 (Complex Dependency)**: "GitHub 푸시가 오고 **동시에** S3에 파일이 올라왔을 때만" 실행하도록 `AND` 조건을 걸 수 있습니다.
3.  **파라미터화 (Parameterization)**: 이벤트의 내용(Payload)을 동적으로 파싱해서 워크플로우의 입력값으로 넘겨줄 수 있습니다. (가장 중요!)
4.  **필터링 (Filtering)**: 불필요한 이벤트는 Sensor 단계에서 걸러내어 리소스 낭비를 막습니다.