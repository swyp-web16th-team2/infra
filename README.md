# infra

배포 설정(k8s 매니페스트) 저장소입니다. Argo CD가 이 저장소를 보고 홈서버의 k3s에 자동으로 맞춥니다.

## 구조

| 경로 | 내용 |
|---|---|
| `apps/backend/` | 백엔드 서버 배포 설정(Deployment, Service). 네임스페이스는 `backend` |
| `argocd/backend.yaml` | Argo CD에게 `apps/backend`를 보라고 알리는 설정. 클러스터에 한 번만 적용 |

## 새 버전 배포

backend 레포 `main`에 병합되면 CI가 이미지를 올린 뒤, 이 레포 `apps/backend/deployment.yaml`의 `image` 태그를 그 커밋 SHA로 바꿔 `main`에 바로 커밋합니다(`backend-ci`). Argo CD가 그것을 보고(기본 약 3분마다 확인) 자동으로 배포합니다. 사람이 태그를 고칠 필요는 없습니다.

- 이전 버전으로 되돌리려면 `image` 태그를 이전 커밋 SHA로 바꾸는 PR을 올립니다. 그 뒤 backend에 새 커밋이 병합되면 다시 최신 태그로 바뀝니다.
- backend 쪽 시크릿 `INFRA_DEPLOY_KEY`(이 레포의 쓰기 가능한 배포 키와 짝)가 없으면 태그 갱신 단계가 실패합니다.

브랜치·커밋·PR 규칙은 backend 레포의 `CONTRIBUTING.md`와 같습니다.
