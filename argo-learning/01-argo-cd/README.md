# Argo CD: The GitOps Engine

Argo CD는 Kubernetes를 위한 **선언적 GitOps 지속적 배포(CD) 도구**입니다.
"애플리케이션의 정의, 설정, 환경은 버전 관리되어야 하며 선언적이어야 한다"는 원칙을 따릅니다.

---

## 1. 아키텍처 및 동작 원리

Argo CD는 크게 세 가지 주요 컴포넌트로 구성됩니다.

1.  **API Server**: gRPC/REST API를 노출하며, Web UI와 CLI의 요청을 처리합니다.
2.  **Repository Server**: Git 저장소(GitHub, GitLab 등)의 로컬 캐시를 유지하고, 매니페스트(YAML)를 생성(Generate)합니다. (Helm 템플릿 처리, Kustomize 빌드 등이 여기서 일어납니다.)
3.  **Application Controller**: 현재 클러스터 상태(Live State)와 Git에 정의된 상태(Desired State)를 비교하고, 차이(Diff)가 발생하면 동기화(Sync)를 수행합니다.

### GitOps 루프
1.  **Code Change**: 개발자가 Git에 배포 설정을 푸시합니다.
2.  **Detect**: Argo CD는 3분마다(기본값) Git을 폴링하여 변경을 감지합니다. (Webhook 설정 시 즉시 감지)
3.  **Diff**: 변경된 Git 내용과 현재 K8s 상태를 비교하여 `OutOfSync` 상태로 표시합니다.
4.  **Sync**: 자동(Auto) 또는 수동(Manual)으로 클러스터에 변경 사항을 적용(Apply)합니다.

---

## 2. 핵심 기능 상세

### 2.1. 동기화 정책 (Sync Policy)
`Application` 리소스에서 가장 중요한 설정입니다.

*   **Manual (기본값)**: `OutOfSync`가 떠도 자동으로 배포하지 않습니다. 사람이 버튼을 눌러야 합니다.
*   **Automated**: 변경 감지 시 즉시 배포합니다.
    *   `prune: true`: Git에서 파일이 삭제되면, K8s 리소스도 삭제합니다. (이게 꺼져 있으면 Git에서 지워도 서버엔 남습니다.)
    *   `selfHeal: true`: 누군가 `kubectl`로 서버 설정을 몰래 바꿔도, Argo CD가 즉시 Git 상태로 되돌립니다. (Config Drift 방지)

### 2.2. 멀티 클러스터 관리
하나의 Argo CD 인스턴스로 **수백 개의 외부 클러스터**를 관리할 수 있습니다.
*   Argo CD가 설치된 관리용 클러스터(Control Plane)가 있고, 배포 대상이 되는 타겟 클러스터(Remote Cluster)들의 API 접근 권한(Secret)만 등록하면 됩니다.

### 2.3. 다양한 배포 도구 지원
단순 YAML뿐만 아니라 템플릿 엔진을 내장하고 있습니다.
*   **Helm Charts**: `values.yaml`을 덮어쓰거나 특정 파라미터만 오버라이딩 가능.
*   **Kustomize**: `kustomization.yaml` 빌드 지원.
*   **Jsonnet**: 구글이 만든 데이터 템플릿 언어 지원.

---

## 3. 주요 CRD 명세 (Deep Dive)

### Application (`kind: Application`)
가장 많이 다루게 될 리소스입니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  # 1. 소스 (Git)
  source:
    repoURL: https://github.com/my-org/my-repo.git
    targetRevision: main # HEAD, v1.0.0, sha1 등 가능
    path: k8s/overlays/prod
    # Helm 전용 설정 예시
    helm:
      valueFiles: [values-prod.yaml]
      parameters:
      - name: replicaCount
        value: "3"

  # 2. 목적지 (K8s)
  destination:
    server: https://kubernetes.default.svc # 로컬 클러스터
    namespace: my-app-prod

  # 3. 동기화 옵션
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true # 네임스페이스가 없으면 생성
    - ApplyOutOfSyncOnly=true # 변경된 리소스만 적용 (성능 최적화)
```

---

## 4. 운영 팁 (Best Practices)

1.  **App of Apps 패턴**: 애플리케이션이 100개라면 100개의 YAML을 `kubectl apply` 하지 마세요. "애플리케이션들을 관리하는 애플리케이션" 하나만 만들어서 Git의 폴더를 바라보게 하세요.
2.  **Secret 관리**: Git에는 비밀번호를 평문으로 올리면 안 됩니다. **Sealed Secrets**나 **External Secrets Operator**와 함께 사용하여 암호화된 상태로 Git에 올리세요.
3.  **리소스 정리 (Finalizer)**: Application 삭제 시 배포된 파드들도 같이 지우고 싶다면 `metadata.finalizers`에 `resources-finalizer.argocd.argoproj.io`를 추가해야 합니다. (기본적으로는 앱만 지워지고 파드는 남습니다.)