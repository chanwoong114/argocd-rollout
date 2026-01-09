# Argo Workflows: The Workflow Engine

Argo Workflows는 Kubernetes를 위한 **오픈소스 컨테이너 네이티브 워크플로우 엔진**입니다.
Airflow, Luigi, Jenkins 등의 역할을 수행할 수 있지만, 모든 단계가 **컨테이너**로 실행된다는 점이 가장 큰 특징입니다.

---

## 1. 사용 사례 (Use Cases)

1.  **CI/CD Pipelines**: 코드 체크아웃 -> 빌드 -> 테스트 -> 이미지 푸시 -> 배포. (Jenkins 대안)
2.  **Machine Learning**: 데이터 전처리 -> 모델 학습 -> 모델 평가 -> 모델 서빙. (Kubeflow Pipelines의 엔진으로 사용됨)
3.  **Data Processing**: ETL 작업, 배치 프로세싱.
4.  **Infrastructure Automation**: 테라폼 실행, 클러스터 프로비저닝.

---

## 2. 핵심 아키텍처

*   **Workflow Controller**: CRD(`Workflow`)를 감시하고, 정의된 순서대로 파드(Pod)를 생성합니다.
*   **Argo Server**: UI를 제공하고 API 요청을 처리합니다.
*   **Executor (Emissary/Docker)**: 파드 내부에서 사용자가 정의한 스크립트나 명령어를 실행하고, 결과(Output/Artifact)를 캡처합니다.

### 데이터 전달 방식 (Artifacts & Parameters)
워크플로우의 핵심은 **Step A의 결과를 Step B로 넘겨주는 것**입니다.
1.  **Parameters**: 짧은 문자열(String) 전달. (예: 도커 이미지 태그, Git 커밋 해시)
2.  **Artifacts**: 큰 파일(File) 전달. (예: 빌드된 바이너리, 컴파일된 모델 파일)
    *   Argo Workflows는 이를 위해 **S3, MinIO, GCS** 같은 외부 스토리지(Artifact Repository)를 사용합니다. A가 S3에 올리면, B가 다운로드 받는 식입니다.

---

## 3. 주요 CRD 명세 (Deep Dive)

### Workflow (`kind: Workflow`)
일회성 실행 정의서입니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ci-pipeline- # 실행 시 랜덤 이름 생성
spec:
  entrypoint: main # 시작 템플릿 지정
  templates:
  - name: main
    dag: # DAG(유향 비순환 그래프) 방식 정의
      tasks:
      - name: build
        template: build-image
      - name: test
        template: run-test
        dependencies: [build] # build가 끝나야 실행됨

  - name: build-image
    container:
      image: docker:dind
      command: [docker, build, .]

  - name: run-test
    inputs:
      parameters: # 파라미터 받기
      - name: message
    script: # 스크립트 직접 작성 가능
      image: python:alpine
      command: [python]
      source: |
        print("Testing...")
```

### WorkflowTemplate (`kind: WorkflowTemplate`)
재사용 가능한 워크플로우 "함수"입니다. 클러스터에 저장해두고 `Workflow`에서 `templateRef`로 호출해서 씁니다.

### CronWorkflow (`kind: CronWorkflow`)
`CronJob`과 같습니다. 스케줄에 맞춰 `Workflow`를 생성합니다.

---

## 4. 특징 및 장점

1.  **DAG (Directed Acyclic Graph)**: 복잡한 병렬 처리 의존성을 쉽게 정의할 수 있습니다. (A 끝나면 B, C 동시 실행 -> 둘 다 끝나면 D 실행)
2.  **Step-Level Caching**: 이미 성공한 단계는 캐싱하여(Memoization) 재실행 시 건너뛸 수 있습니다. (빌드 시간 단축)
3.  **Suspend & Resume**: 워크플로우 중간에 사람의 승인을 기다리도록(일시 정지) 할 수 있습니다.
4.  **Retry Policy**: 실패 시 재시도 정책을 세밀하게 설정할 수 있습니다. (예: 네트워크 에러일 때만 3번 재시도)