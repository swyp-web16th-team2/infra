# infra

배포 설정(k8s 매니페스트) 저장소입니다. Argo CD가 이 저장소를 보고 홈서버의 k3s에 자동으로 맞춥니다.

## 구조

| 경로 | 내용 |
|---|---|
| `apps/backend/` | 백엔드 서버 배포 설정(Deployment, Service, Ingress)과 캐시용 Redis(`redis.yaml`). 네임스페이스는 `backend` |
| `argocd/backend.yaml` | Argo CD에게 `apps/backend`를 보라고 알리는 설정. 클러스터에 한 번만 적용 |

## 외부 접속

`https://swyp16th.duckdns.org` → 공유기(443) → 홈서버 k3s의 Traefik(k3s 내장 Ingress 컨트롤러) → `apps/backend/ingress.yaml` → backend 서비스.

- **https만 열려 있습니다.** 공유기에서 80 포트는 열지 않았으므로 `http://` 주소는 밖에서 접속되지 않습니다. 의도한 상태입니다.
- 도메인은 DuckDNS(`swyp16th.duckdns.org`)로, 홈서버의 공인 IP를 가리킵니다.
- k3s는 Traefik과 servicelb(80·443을 노드에 연결)를 켠 상태로 돌아야 합니다. k3s를 다시 설치할 때 `--disable=traefik`, `--disable=servicelb`를 넣지 마세요.

### 인증서(이 레포에 없음)
인증서는 비밀 정보라 이 레포에 두지 않고 **홈서버에서** 관리합니다.
- `acme.sh`가 DuckDNS DNS 확인 방식으로 Let's Encrypt 인증서를 받습니다. 포트를 쓰지 않습니다.
- systemd 타이머 `home-server-cert-renew`가 매일 확인하고, 만료 30일 전에 갱신합니다.
- 갱신되면 `/usr/local/sbin/sync-backend-tls-secret`이 인증서를 `backend` 네임스페이스의 `backend-tls` 시크릿으로 넣고, Ingress가 그것을 씁니다.
- 수동 갱신: 홈서버에서 `sudo CERT_RENEW_FORCE=1 /usr/local/sbin/renew-duckdns-cert`.

## DB·Redis 연결

- **Redis**: `apps/backend/redis.yaml`. 캐시용이라 디스크에 저장하지 않고(재시작하면 비워짐) 메모리를 64MB로 제한합니다. 비밀번호가 없어서 NetworkPolicy로 backend 파드에서 오는 연결만 허용합니다. backend는 `REDIS_URL=redis://redis:6379`로 접속합니다.
- **Supabase Postgres**: backend는 `DATABASE_URL`(트랜잭션 풀러 주소)을 시크릿 `backend-db`에서 읽습니다. 비밀이라 이 레포에 값을 두지 않고 **홈서버에서 직접** 만듭니다. 시크릿이 없어도 backend는 뜨지만 `/health`가 `db: down`(503)을 돌려줍니다.
  ```bash
  # 홈서버에서. 주소를 화면에 보이거나 셸 기록에 남기지 않도록 read -s로 입력받습니다.
  read -rs DATABASE_URL && sudo k3s kubectl -n backend create secret generic backend-db \
    --from-literal=DATABASE_URL="$DATABASE_URL" --dry-run=client -o yaml | sudo k3s kubectl apply -f - ; unset DATABASE_URL
  sudo k3s kubectl -n backend rollout restart deploy/backend   # 환경 변수는 파드가 시작할 때만 읽습니다
  ```
- 마이그레이션용 `DIRECT_URL`(세션 풀러, IPv4 가능)은 파드가 아니라 마이그레이션을 실행하는 곳에서 씁니다. 첫 모델이 생길 때 실행 방법을 정합니다.
- 확인: `curl https://swyp16th.duckdns.org/health` → `{"status":"ok","db":"up","redis":"up"}`.

## 새 버전 배포

backend 레포 `main`에 병합되고 CI가 이미지를 올리면, `deploy` 작업이 이 레포 `apps/backend/deployment.yaml`의 `image` 태그를 **그 커밋의 SHA**로 바꿔 `main`에 바로 커밋합니다(작성자 `backend-ci`). Argo CD가 그것을 보고(기본 설정에서 보통 3분 안팎) 자동으로 배포합니다. 사람이 태그를 고칠 필요는 없습니다.

- `deploy`는 한 번에 하나씩 줄을 서서 돕니다(대기 작업은 취소되지 않고 최대 100개까지 보존). 각 실행은 **지금 배포된 커밋보다 앞선 커밋일 때만** 바꿉니다. 늦게 끝난 옛 실행이나 재실행이 새 버전을 옛 버전으로 되돌리지 못하고, 같거나 더 오래된 커밋이면 건너뜁니다.
- 최신 커밋의 CI가 실패하면 그 커밋은 배포되지 않고, 그 앞의 정상 커밋은 자기 실행에서 배포됩니다.
- 현재 태그가 커밋 SHA가 아니거나 backend에 없는 커밋이라 비교할 수 없으면 `deploy`가 실패합니다. 이때는 사람이 확인해야 합니다.

**병합했다고 바로 배포된 것은 아니니, 마지막에 항상 실제 최종 상태를 직접 확인하세요.**
1. backend 레포 Actions의 **최신 실행**에서 `check`, `image`, `publish`, `deploy`가 모두 성공했는지
2. 이 레포 `deployment.yaml`의 `image` 태그가 backend `main`에서 CI가 성공한 최신 커밋의 SHA와 같은지
3. Argo CD 앱 `backend`가 이 레포 `main`의 최신 커밋을 보고 있고(Revision) Synced / Healthy인지. Synced / Healthy만 보면 이전 버전이 정상으로 돌고 있는 것을 배포 완료로 착각할 수 있습니다.

### 이전 버전으로 되돌리기
- 가장 안전한 방법은 backend에서 문제가 된 커밋을 되돌리는(revert) PR을 병합하는 것입니다. 되돌림 커밋은 문제 커밋보다 앞선 커밋이라 위 자동 배포가 그대로 배포합니다.
- 급하면 이 레포의 `image` 태그를 이전 커밋 SHA로 바꾸는 PR을 올리고 **병합**합니다. 병합해야 Argo CD가 반영합니다. 다만 이 경우 자동 배포는 롤백한 버전보다 앞선 커밋을 다시 배포할 수 있으니 조심하세요.
  - backend Actions에 진행 중인 실행이 있으면 그 `deploy`가 끝나면서 태그를 그 커밋으로 바꿀 수 있습니다. 먼저 진행 중인 실행이 없는지 확인하세요.
  - 롤백 뒤 backend에서 이전 실행을 **다시 실행(re-run)**하거나 새 커밋이 병합되어 CI가 성공하면, 그 커밋이 롤백한 버전보다 앞서므로 문제가 있던 버전이 다시 배포될 수 있습니다. backend에 되돌림(revert)을 먼저 병합하고, 그 전에는 재실행하거나 다른 병합을 하지 마세요.
- backend 쪽 시크릿 `INFRA_DEPLOY_KEY`(이 레포의 쓰기 가능한 배포 키와 짝)가 없으면 태그 갱신이 실패합니다.

브랜치·커밋·PR 규칙은 backend 레포의 `CONTRIBUTING.md`와 같습니다.
