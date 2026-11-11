# infra

배포 설정(k8s 매니페스트) 저장소입니다. Argo CD가 이 저장소를 보고 홈서버의 k3s에 자동으로 맞춥니다.

## 구조

| 경로 | 내용 |
|---|---|
| `apps/backend/` | 백엔드 서버 배포 설정(Deployment, Service). 네임스페이스는 `backend` |
| `argocd/backend.yaml` | Argo CD에게 `apps/backend`를 보라고 알리는 설정. 클러스터에 한 번만 적용 |

## 새 버전 배포

`apps/backend/deployment.yaml`의 `image` 태그를 backend 레포 `main`의 커밋 SHA로 바꾸는 PR을 올리고 병합하면 Argo CD가 자동으로 배포합니다. (이미지는 `main` 병합 때 자동으로 올라갑니다.)

브랜치·커밋·PR 규칙은 backend 레포의 `CONTRIBUTING.md`와 같습니다.
