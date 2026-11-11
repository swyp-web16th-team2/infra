# infra

배포 설정(k8s 매니페스트) 저장소입니다. Argo CD가 이 저장소를 보고 홈서버의 k3s에 자동으로 맞춥니다.

## 구조

| 경로 | 내용 |
|---|---|
| `apps/backend/` | 백엔드 서버 배포 설정(Deployment, Service). 네임스페이스는 `backend` |
| `argocd/backend.yaml` | Argo CD에게 `apps/backend`를 보라고 알리는 설정. 클러스터에 한 번만 적용 |

## 새 버전 배포

backend 레포 `main`에 병합되고 CI가 성공하면, `deploy` 작업이 이 레포 `apps/backend/deployment.yaml`의 `image` 태그를 그 커밋 SHA로 바꿔 `main`에 바로 커밋합니다(작성자 `backend-ci`). Argo CD가 그것을 보고(기본 설정에서 보통 3분 안팎) 자동으로 배포합니다. 사람이 태그를 고칠 필요는 없습니다.

병합했다고 바로 배포된 것은 아닙니다. 아래를 확인하세요.
- backend 레포 Actions에서 `check`, `image`, `publish`, `deploy`가 모두 성공했는지. (CI가 실패하면 태그는 바뀌지 않습니다. 병합 직후 더 새 커밋이 들어오면 앞선 커밋은 태그 갱신을 건너뛰고, 더 새 커밋이 배포합니다.)
- Argo CD 앱 `backend`가 Synced / Healthy인지.

### 이전 버전으로 되돌리기
- 가장 안전한 방법은 backend에서 문제가 된 커밋을 되돌리는(revert) PR을 병합하는 것입니다. 그러면 위 자동 배포가 되돌린 버전을 배포합니다.
- 급하면 이 레포의 `image` 태그를 이전 커밋 SHA로 바꾸는 PR을 올리고 **병합**합니다. 병합해야 Argo CD가 반영합니다. 단, backend Actions에 진행 중인 실행이 있으면 그 `deploy`가 끝나면서 태그를 다시 최신으로 바꿀 수 있으니, 먼저 진행 중인 실행이 없는지 확인하고 backend에도 되돌림을 반영하세요.
- backend 쪽 시크릿 `INFRA_DEPLOY_KEY`(이 레포의 쓰기 가능한 배포 키와 짝)가 없으면 태그 갱신이 실패합니다.

브랜치·커밋·PR 규칙은 backend 레포의 `CONTRIBUTING.md`와 같습니다.
