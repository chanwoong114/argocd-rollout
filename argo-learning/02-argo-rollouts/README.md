# Argo Rollouts: Advanced Deployment Strategies

Argo Rollouts는 Kubernetes의 표준 `Deployment` 리소스가 제공하지 못하는 **Blue/Green, Canary, Analysis(자동 검증)** 기능을 제공하는 **Progressive Delivery 컨트롤러**입니다.

---

## 1. 왜 필요한가? (Deployment의 한계)
Kubernetes 기본 `Deployment`의 `RollingUpdate` 전략은 다음과 같은 문제가 있습니다.
1.  **제어 불가**: 배포 속도를 조절할 수 없습니다. (그냥 `maxSurge` 설정에 따라 우르르 교체됨)
2.  **검증 불가**: "새 버전이 500 에러를 뿜는지" 확인하지 않고 그냥 배포합니다.
3.  **트래픽 제어 불가**: "전체 유저의 1%에게만 보여주고 싶다"는 요구사항을 처리하기 어렵습니다.

Argo Rollouts는 이를 해결하기 위해 **네트워크 레벨(Service Mesh, Ingress)의 트래픽 제어**와 **단계적 배포(Steps)**를 통합했습니다.

---

## 2. 아키텍처 및 동작 원리

Argo Rollouts 컨트롤러는 `Rollout` 리소스를 감시하며 다음 작업을 수행합니다.

1.  **ReplicaSet 관리**: `Deployment`와 마찬가지로 `ReplicaSet`을 생성하여 파드 개수를 조절합니다.
    *   `Stable ReplicaSet`: 현재 서비스 중인 구 버전.
    *   `Canary ReplicaSet`: 새로 배포된 신 버전.
2.  **Networking 조작**: 설정된 트래픽 라우터(Ingress Nginx, AWS ALB, Istio 등)의 설정을 동적으로 변경하여 트래픽 비율을 조절합니다. (예: `weight: 20` 설정)
3.  **Analysis 실행**: 배포 단계 중간에 `AnalysisRun`을 생성하여 외부 메트릭(Prometheus 등)을 질의하고, 결과에 따라 진행(Promote)할지 롤백(Abort)할지 결정합니다.

---

## 3. 핵심 기능 상세

### 3.1. 트래픽 관리 (Traffic Management)
단순 파드 개수 조절이 아니라, 실제 네트워크 트래픽을 나눕니다.
*   **Basic (Service 레벨)**: 파드 개수 비율로 조정. (가장 단순, 정밀도 낮음)
*   **Ingress Nginx**: Ingress Annotation을 조작하여 정밀하게 나눔.
*   **AWS ALB**: TargetGroup 가중치를 조절.
*   **Service Mesh (Istio, Linkerd)**: VirtualService 등을 조작.

### 3.2. Blue/Green 전략
*   **개념**: 구 버전(Active)과 신 버전(Preview)을 동시에 완벽하게 띄워두고 스위칭.
*   **특징**: `previewService`를 별도로 지정하여, 운영자는 미리 새 버전을 테스트해볼 수 있습니다.
*   **설정 예시**: `autoPromotionEnabled: false`로 설정하면 운영자가 승인하기 전까지 트래픽을 전환하지 않습니다.

### 3.3. Canary 전략
*   **개념**: 트래픽을 조금씩 이동.
*   **Steps**:
    *   `setWeight`: 트래픽 비율 설정.
    *   `pause`: 대기 (시간 지정 또는 무기한 대기).
    *   `analysis`: 검증 실행.

---

## 4. 주요 CRD 명세 (Deep Dive)

### Rollout (`kind: Rollout`)
`Deployment`와 99% 유사하지만 `strategy` 필드가 다릅니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  replicas: 5
  strategy:
    canary:
      # 트래픽 제어 대상 서비스 (안정 버전용)
      stableService: my-app-stable
      # 트래픽 제어 대상 서비스 (신규 버전용)
      canaryService: my-app-canary
      trafficRouting:
        nginx: # Nginx Ingress를 사용하여 트래픽 제어
          stableIngress: my-app-ingress
      steps:
      - setWeight: 20
      - pause: {duration: 1h} # 1시간 동안 20% 유지하며 모니터링
      - analysis:
          templates:
          - templateName: success-rate # 에러율 검사
      - setWeight: 50
      - pause: {} # 운영자가 수동 승인할 때까지 대기
```

### AnalysisTemplate (`kind: AnalysisTemplate`)
재사용 가능한 검증 로직입니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
spec:
  metrics:
  - name: success-rate
    interval: 5m # 5분마다 체크
    failureLimit: 3 # 3번 실패하면 롤백
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(irate(http_requests_total{status!~"5.*"}[1m])) 
          / 
          sum(irate(http_requests_total[1m]))
        successCondition: result[0] >= 0.95 # 성공률 95% 이상이어야 통과
```

---

## 5. 운영 팁

1.  **HPA 호환성**: `Rollout`은 K8s 기본 HPA(Horizontal Pod Autoscaler)와 호환됩니다. `scaleTargetRef`를 `Rollout`으로 지정하면 됩니다.
2.  **Ephemeral Metadata**: 카나리 배포 중에만 파드에 특정 라벨을 붙이고 싶다면 `ephemeralMetadata` 기능을 사용하세요. (로그 수집이나 모니터링 시 버전 구분 용이)
3.  **Anti-Affinity**: 카나리 배포 시 구 버전 파드와 신 버전 파드가 한 노드에 몰리지 않도록 `antiAffinity` 설정을 꼼꼼히 하세요.