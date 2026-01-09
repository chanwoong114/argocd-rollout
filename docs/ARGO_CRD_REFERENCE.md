# Argo Ecosystem CRD Reference Guide

**Project:** TROMBONE  
**Date:** 2026-01-08  
**Author:** ArgoCD CLI Agent  
**Context:** Kubernetes Progressive Delivery & GitOps

---

## 1. 개요 (Overview)

본 문서는 트롬본 프로젝트의 배포 파이프라인을 구성하는 **Argo Rollouts**와 **Argo CD**의 핵심 **CRD(Custom Resource Definition)**를 기술적으로 분석하고 정리한 문서입니다.

각 CRD가 기존 Kubernetes Native Resource(Deployment 등)와 어떻게 다르며, 내부적으로 어떤 역할을 수행하는지, 그리고 실무에서 어떻게 활용되는지를 중점적으로 다룹니다.

---

## 2. Argo Rollouts CRD (Progressive Delivery)

Argo Rollouts는 Kubernetes의 기본 컨트롤러가 제공하지 못하는 **고급 배포 전략(Blue/Green, Canary)**을 수행하기 위해 자체 컨트롤러를 사용합니다.

### 2.1. Rollout (`kind: Rollout`)

가장 핵심이 되는 리소스로, Kubernetes의 **`Deployment` 리소스를 대체**합니다.

*   **Wrapper Target**: `Deployment` + `ReplicaSet` 관리 로직 + **Traffic Management**
*   **기술적 특징**:
    *   기존 `Deployment`와 `spec`이 거의 100% 호환됩니다 (`replicas`, `selector`, `template` 등).
    *   **Custom Controller Loop**: `Deployment` 컨트롤러는 단순히 파드 개수를 맞추는 데 집중하지만, `Rollout` 컨트롤러는 `spec.strategy`에 정의된 단계(Steps)에 따라 `ReplicaSet`의 스케일링과 서비스(Service)/인그레스(Ingress)의 가중치를 조절합니다.
*   **핵심 필드**:
    ```yaml
    spec:
      strategy:
        canary: # 또는 blueGreen
          steps:
          - setWeight: 20
          - pause: {duration: 1h}
          - analysis: {templates: [{templateName: success-rate}]}
    ```

### 2.2. AnalysisTemplate (`kind: AnalysisTemplate`)

배포의 품질을 판단하는 **"검증 로직(Metrics/Tests)"을 캡슐화**한 템플릿입니다.

*   **역할**: 배포의 성공/실패 여부를 결정하는 쿼리나 스크립트 정의.
*   **Providers**:
    *   **Prometheus/Datadog/CloudWatch**: 쿼리 결과(예: 에러율 5% 미만)를 평가.
    *   **Job**: Kubernetes Job을 실행하여 Exit Code(0=성공, 1=실패)로 평가.
    *   **Web**: 외부 API를 호출하여 응답값(HTTP 200)으로 평가.
*   **Scope**: 기본적으로 네임스페이스(Namespace) 레벨입니다.

### 2.3. ClusterAnalysisTemplate (`kind: ClusterAnalysisTemplate`)

`AnalysisTemplate`과 동일하지만, **클러스터 전역(Cluster-wide)**에서 사용할 수 있는 리소스입니다.

*   **Use Case**: 전사적으로 공통된 모니터링 기준(예: "모든 서비스는 CPU Throttle이 발생하면 안 된다")을 정의해두고, 개별 팀의 Rollout이 이를 참조(`clusterScope: true`)하여 사용하도록 강제할 때 유용합니다.

### 2.4. AnalysisRun (`kind: AnalysisRun`)

`AnalysisTemplate`의 **실행 인스턴스(Instance)**입니다.

*   **작동 방식**:
    *   `Rollout`이 배포 단계(`Step`) 중 `analysis` 단계를 만나면, `AnalysisTemplate`을 복제하여 `AnalysisRun` 객체를 생성합니다.
    *   이 객체는 생성되는 순간부터 측정(Metric Query or Job Execution)을 시작합니다.
    *   결과는 `Successful`, `Failed`, `Inconclusive`, `Error` 상태로 기록됩니다.
*   **수명 관리(GC)**: `Rollout` 설정(`successfulRunsHistoryLimit`, `failedRunsHistoryLimit`)에 따라 일정 개수만 유지되고 삭제됩니다.

### 2.5. Experiment (`kind: Experiment`)

단순 교체(Replacement)가 아닌, **동시에 여러 버전을 띄워 데이터를 수집**하는 A/B 테스팅 전용 리소스입니다.

*   **Rollout(Canary)과의 차이점**:
    *   **Canary**: 구버전 -> 신버전으로의 "안전한 전환"이 목적.
    *   **Experiment**: 버전 A vs 버전 B의 "비즈니스 지표 비교"가 목적. 실험 종료 후 어떤 버전도 선택되지 않을 수 있음.
*   **기술적 특징**:
    *   일시적으로 `ReplicaSet`을 생성하여 트래픽을 흘려보낸 뒤, `AnalysisRun`을 통해 결과를 도출하고 종료합니다.

---

## 3. Argo CD CRD (GitOps & Sync)

Argo CD는 Git 저장소의 상태를 Kubernetes 클러스터에 동기화(Reconcile)하는 역할을 수행합니다.

### 3.1. Application (`kind: Application`)

Argo CD의 가장 기본적인 배포 단위입니다.

*   **정의 내용**:
    *   **Source**: Git 저장소 URL, 리비전(Branch/Tag/Commit), 경로(Path).
    *   **Destination**: 배포될 Kubernetes 클러스터 URL 및 네임스페이스.
    *   **Project**: 해당 앱이 속한 논리적 그룹(`AppProject`).
    *   **SyncPolicy**: 자동화 여부(`automated`), 가지치기(`prune`), 자가치유(`selfHeal`).
*   **기술적 특징**: Argo CD 컨트롤러는 이 리소스를 주기적으로 감시하며 Git과 클러스터 간의 Diff(차이)를 계산하고 리포팅합니다.

### 3.2. AppProject (`kind: AppProject`)

멀티 테넌시(Multi-tenancy) 환경을 위한 **보안 및 권한 관리 격리벽**입니다.

*   **주요 기능**:
    *   **Source Repositories Whitelist**: 이 프로젝트는 특정 Git 주소에서만 배포 가능.
    *   **Destination Whitelist**: 이 프로젝트는 특정 클러스터/네임스페이스에만 배포 가능.
    *   **Resource Whitelist**: 이 프로젝트는 `Secret`, `RoleBinding` 등 민감한 리소스 생성을 차단 가능.
*   **활용**: "결제 팀" 프로젝트는 "결제-prod" 네임스페이스에만 배포 가능하도록 제한.

### 3.3. ApplicationSet (`kind: ApplicationSet`)

하나의 템플릿으로 **여러 개의 `Application`을 동적으로 생성**하는 "Application Factory"입니다.

*   **Generators (생성기)**:
    *   **List Generator**: 하드코딩된 리스트 기반 생성.
    *   **Git Generator**: Git 디렉토리 구조를 스캔하여 폴더마다 App 생성 (Mono-repo 패턴에 최적).
    *   **Cluster Generator**: 등록된 모든 클러스터에 동일한 App 배포 (Add-on 관리에 최적).
*   **Use Case**: "마이크로서비스가 50개인데 `Application` YAML을 50번 작성하기 싫다" -> `ApplicationSet` 1개로 해결.

---

## 4. 리소스 상호작용 다이어그램 (Interaction)

```text
[Git Repository]
      |
      | (Source Code Change)
      v
[Argo CD Controller] <--- (Watch) --- [Application CRD]
      |
      | (Sync / Apply)
      v
[Kubernetes Cluster]
      +-----------------------------------------+
      | [Rollout Controller] <--- [Rollout CRD] |
      |          |                              |
      |          +---> (Creates) [ReplicaSet]   |
      |          |                              |
      |          +---> (Creates) [AnalysisRun]  |
      |                               |         |
      |          [AnalysisTemplate] <-+         |
      |                                         |
      +-----------------------------------------+
```

## 5. 요약 비교 (Summary)

| 영역 | CRD | 역할 | 비유 |
| :--- | :--- | :--- | :--- |
| **Delivery** | **Rollout** | Deployment 대체, 배포 전략 실행 | 현장 감독관 |
| | **AnalysisTemplate** | 메트릭 검증 로직 정의 | 품질 검사 기준표 |
| | **AnalysisRun** | 검증 실행 및 결과 저장 | 품질 검사 성적서 |
| **GitOps** | **Application** | 1:1 배포 매핑 (Git -> K8s) | 작업 지시서 |
| | **AppProject** | 권한 제어 및 격리 | 부서별 보안 구역 |
| | **ApplicationSet** | 1:N 배포 자동화 (템플릿) | 대량 생산 라인 |

---
**Note:** 트롬본 프로젝트의 초기 단계에서는 `Application`, `Rollout`, `AnalysisTemplate` 세 가지 리소스의 작성법을 숙지하는 것이 가장 중요합니다.
