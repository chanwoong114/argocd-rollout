# TROMBONE Full Pipeline Architecture

이 문서는 Argo Ecosystem 4가지(**Events, Workflows, CD, Rollouts**)를 모두 통합하여 구축할 수 있는 **End-to-End GitOps 파이프라인**의 청사진입니다.

---

## 1. The Big Picture (전체 흐름도)

```mermaid
graph TD
    User[Developer] -->|git push| Git[GitHub Repository]
    
    subgraph "CI & Event Phase"
        Git -->|Webhook| ES[Argo Events (Source)]
        ES -->|Event| Sensor[Argo Events (Sensor)]
        Sensor -->|Trigger| WF[Argo Workflows]
    end
    
    subgraph "Build Phase"
        WF -->|1. Test| Test[Unit Tests]
        Test -->|2. Build| Docker[Docker Build]
        Docker -->|3. Push| Reg[Container Registry]
    end
    
    subgraph "GitOps Update Phase"
        Reg -->|New Image Detected| Updater[Argo CD Image Updater]
        Updater -->|Update Tag & Commit| GitConfig[Git Config Repo]
    end
    
    subgraph "CD & Delivery Phase"
        GitConfig -->|Sync| ACD[Argo CD]
        ACD -->|Apply| K8s[Kubernetes Cluster]
        
        K8s -->|Create| Rollout[Argo Rollouts]
        Rollout -->|20% Traffic| V2[Canary Pods]
        Rollout -->|80% Traffic| V1[Stable Pods]
        
        V2 -->|Metrics| Prom[Prometheus]
        Rollout -->|Query| Analysis[AnalysisRun]
    end
    
    Analysis -->|Success| Promote[Promote to 100%]
    Analysis -->|Fail| Rollback[Rollback to V1]
```

---

## 2. 컴포넌트별 역할 정의

### ① 감지 및 트리거 (Argo Events)
*   **역할**: CI 파이프라인의 **"시작 버튼"**입니다.
*   **동작**: 개발자가 코드를 푸시하면 GitHub Webhook을 받아 유효성을 검증하고, CI 워크플로우를 실행합니다.

### ② 통합 및 빌드 (Argo Workflows)
*   **역할**: **"공장 라인"**입니다.
*   **동작**:
    1.  코드를 체크아웃 받고 테스트를 돌립니다.
    2.  도커 이미지를 빌드하여 레지스트리(ECR/DockerHub)에 푸시합니다.
    3.  이때 생성된 이미지 태그(예: `v1.0.1-sha1234`)를 다음 단계로 넘깁니다.

### ③ 상태 업데이트 (Argo CD Image Updater)
*   **역할**: **"서기(기록원)"**입니다.
*   **동작**: 레지스트리에 새 이미지가 올라온 것을 감지하면, 배포용 Git 저장소(Config Repo)의 `values.yaml`이나 `kustomization.yaml`을 자동으로 수정하여 커밋합니다.

### ④ 배포 및 동기화 (Argo CD)
*   **역할**: **"배송 기사"**입니다.
*   **동작**: Config Repo가 업데이트된 것을 감지하고, 변경된 매니페스트를 쿠버네티스 클러스터에 적용합니다.

### ⑤ 전략적 배포 (Argo Rollouts)
*   **역할**: **"안전 관리자"**입니다.
*   **동작**: 한 번에 교체하지 않고 카나리 전략을 통해 안전성을 검증하며 점진적으로 배포합니다.

---

## 3. 구축 로드맵 (Step-by-Step)

1.  **Argo CD & Rollouts 도입**: 현재 완료되었습니다. 수동으로 이미지를 바꿔가며 배포할 수 있습니다.
2.  **Argo Events & Workflows 도입**: CI 도구(Jenkins/GitHub Actions)를 대체하거나 연동하여, "코드 푸시 시 테스트 자동화"를 구현합니다.
3.  **Image Updater 연동**: CI가 끝난 후 "Git 업데이트" 과정을 자동화하여 사람의 손을 완전히 제거합니다.
4.  **Prometheus & Analysis 연동**: "배포 후 검증" 과정을 자동화하여, 장애 발생 시 새벽에 깨지 않고 자동 롤백되도록 만듭니다.