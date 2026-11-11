# infra

배포 설정(k8s 매니페스트) 저장소입니다. Argo CD가 이 저장소를 보고 홈서버의 k3s에 자동으로 맞춥니다.

## 구조

| 경로 | 내용 |
|---|---|
| `apps/backend/` | 백엔드 서버 배포 설정(Deployment, Service). 네임스페이스는 `backend` |
| `argocd/backend.yaml` | Argo CD에게 `apps/backend`를 보라고 알리는 설정. 클러스터에 한 번만 적용 |

## 새 버전 배포

backend 레포 `main`에 병합되고 CI가 이미지를 올리면, `deploy` 작업이 이 레포 `apps/backend/deployment.yaml`의 `image` 태그를 **backend `main`에서 이미지가 있는 가장 최신 커밋 SHA**로 바꿔 `main`에 바로 커밋합니다(작성자 `backend-ci`). Argo CD가 그것을 보고(기본 설정에서 보통 3분 안팎) 자동으로 배포합니다. 사람이 태그를 고칠 필요는 없습니다.

- `deploy`는 실행 시점에 backend `main`의 최근 커밋 10개를 새 것부터 보면서 이미지가 GHCR에 있는 첫 커밋을 고릅니다. 그래서 최신 커밋의 CI가 실패했거나 이미지가 아직 올라오는 중이면 그 앞의 정상 커밋이 배포됩니다. 최근 10개 안에 이미지가 있는 커밋이 하나도 없으면 `deploy`가 실패합니다.
- 조회 직후 새 커밋이 병합될 수 있어서, 잠깐 직전 커밋 태그가 반영됐다가 새 커밋의 `deploy`가 이어서 맞추는 일이 있습니다.

**병합했다고 바로 배포된 것은 아니니, 마지막에 항상 실제 최종 상태를 직접 확인하세요.**
1. backend 레포 Actions의 **최신 실행**에서 `check`, `image`, `publish`, `deploy`가 모두 성공했는지
2. 이 레포 `deployment.yaml`의 `image` 태그가 backend `main`에서 CI가 성공한 최신 커밋의 SHA와 같은지
3. Argo CD 앱 `backend`가 이 레포 `main`의 최신 커밋을 보고 있고(Revision) Synced / Healthy인지. Synced / Healthy만 보면 이전 버전이 정상으로 돌고 있는 것을 배포 완료로 착각할 수 있습니다.

### 이전 버전으로 되돌리기
- 가장 안전한 방법은 backend에서 문제가 된 커밋을 되돌리는(revert) PR을 병합하는 것입니다. 그러면 위 자동 배포가 되돌린 버전을 배포합니다.
- 급하면 이 레포의 `image` 태그를 이전 커밋 SHA로 바꾸는 PR을 올리고 **병합**합니다. 병합해야 Argo CD가 반영합니다. 다만 두 가지를 조심하세요.
  - backend Actions에 진행 중인 실행이 있으면 그 `deploy`가 끝나면서 태그를 다시 최신으로 바꿀 수 있습니다. 먼저 진행 중인 실행이 없는지 확인하세요.
  - 롤백 뒤 backend `main`에 **어떤 커밋이든 새로 병합되어 CI가 성공하면** 문제가 있던 버전까지 포함한 최신 커밋이 다시 배포됩니다. 그 전에 backend에 되돌림(revert)을 먼저 병합하거나, 그때까지 다른 병합을 멈추세요.
- backend 쪽 시크릿 `INFRA_DEPLOY_KEY`(이 레포의 쓰기 가능한 배포 키와 짝)가 없으면 태그 갱신이 실패합니다.

브랜치·커밋·PR 규칙은 backend 레포의 `CONTRIBUTING.md`와 같습니다.
