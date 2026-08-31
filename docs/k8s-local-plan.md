# 로컬 쿠버네티스 전환 계획

작성 2026-08-29 · 개정 2026-08-31. 진행하며 갱신한다.

## 목표

1. 맥북에 쿠버네티스를 구축한다
2. 지금 만들고 있는 앱을 그 위에 올린다
3. **보안이 확보되면** 외부에 공개한다

3번이 1·2번과 요구사항이 다르다. 로컬 개발용 클러스터와 인터넷에 노출되는
서비스는 같은 물건이 아니다. 그래서 트랙을 나눈다.

## 트랙 구조

| 트랙 | 내용 | 성격 |
|---|---|---|
| **A. 로컬 구동** | 클러스터·CI → ArgoCD → 상태 저장소 → 서비스 메시 → 시크릿 → 앱 → 내부 검증 | 필수 경로 |
| **B. 인증 통합과 하드닝** | 게임 계정 제거, 관리자 가입으로 단일화, 비밀값 교체, 레이트 리밋 | **공개의 전제조건** |
| **C. Harbor 운영** | 프록시 캐시·복제·보존 정책·스캔 | A·D 와 독립 |
| **D. 외부 공개** | Cloudflare Tunnel, 노출 대상 최소화, WAF·Access | 마지막 |

B를 건너뛰고 D로 갈 수 없다. C는 언제 해도 된다.

## 확정된 전제

| 항목 | 결정 | 이유 |
|---|---|---|
| 배포판 | minikube (docker 드라이버) | 업스트림 쿠버네티스를 kubeadm으로 부트스트랩한다. `stop`/`start`가 일급 명령이라 데이터를 유지한 채 껐다 켤 수 있다 |
| 구성 방식 | 처음부터 GitOps | 원격 클러스터와 차트를 공유한다. 로컬은 `local.values.yaml` 로만 갈라진다 |
| 이미지 빌드 | GitHub Actions → GHCR | 저장소가 전부 public이라 표준 러너가 무료다. `ubuntu-24.04-arm` 으로 arm64를 네이티브 빌드한다 |
| 레지스트리 | Harbor (프록시 캐시 + 복제) | 클러스터의 pull 이 Harbor를 지난다. 운영 대상이자 실제 경로다 |
| self-hosted 러너 | 쓰지 않는다 | public 저장소에서는 fork PR이 러너에서 임의 코드를 실행한다 |
| 인증 | **관리자 가입으로 단일화, 승인 메커니즘 유지** | 게임 계정(uuid 로그인)을 없앤다. 승인 게이트가 공개 시 남용을 막는다 |
| 임베딩 | Ollama → Google `gemini-embedding-001` | 호스트 의존(`host.docker.internal:11434`)이 사라지고 호스트 메모리가 빈다 |
| 공개 경로 | Cloudflare Tunnel | 포트를 열지 않고 집 IP도 노출되지 않는다. WAF·레이트 리밋·Access 를 무료 티어에서 쓴다 |
| Istio·Vault | 처음부터 도입 | 나중에 붙이면 이미 굳어진 배포·시크릿 참조를 전부 다시 손대야 한다. 처음부터 넣어 그 비용을 피한다 |

## 착수 전에 고쳐야 할 것

public-infra의 GitOps 구성은 지금 그대로는 동작하지 않는다.

| # | 문제 | 위치 |
|---|---|---|
| 1 | appset이 존재하지 않는 저장소 `bbbbb0413/infra.git` 을 가리킨다. 실제 저장소는 `public-infra` 다 | `charts/appsets/values.yaml`, `appsets/infra-applicationset.yaml` |
| 2 | environment 가 `dev-test` 라 `dev-test.values.yaml` 을 찾지만, 차트에는 `dev.values.yaml` 만 있다 | `charts/appsets/values.yaml` |
| 3 | infra ApplicationSet 이 전부 주석 처리되어 MySQL·Redis 차트가 배포되지 않는다 | `charts/appsets/values.yaml` |
| 4 | 앱 차트가 `app/server` 하나뿐이고 `identity` 만 띄운다. 지금 필요한 것은 7개다 | `charts/app/` |
| 5 | Kafka·MongoDB·Qdrant 차트가 없다 | `charts/infra/` |
| 6 | `imageCredentials` 가 Harbor admin 계정을 그대로 쓴다 | `charts/app/server/dev.values.yaml` |

## 리소스 배정

머신은 10코어 · 32GB다. compose가 선언한 한도 합계는 8.5 vCPU · 6.7GB지만,
이 값은 상한이지 실사용량이 아니다. 쿠버네티스에서는 `requests` 가 스케줄링을
결정하므로 상한을 그대로 옮기지 않는다.

```bash
minikube start -p public-project \
  --driver docker \
  --cpus 6 --memory 12g --disk-size 40g
```

애드온을 켜지 않는다. `ingress`는 Istio ingress gateway가 대신하고(A6),
`metrics-server`는 HPA를 쓰지 않을 거라 필요 없다.

| 구성요소 | CPU requests | Memory requests | 비고 |
|---|---|---|---|
| MySQL | 100m | 512Mi | `innodb_buffer_pool_size` 를 256M로 낮춘다 |
| MongoDB | 100m | 512Mi | `mongodb/mongodb-atlas-local` 대신 표준 `mongo` 검토 |
| Kafka | 200m | 768Mi | KRaft 단일 노드, JVM 힙 512M 고정 |
| Qdrant | 100m | 256Mi | |
| Redis | 50m | 128Mi | |
| NestJS 5개 | 각 50m | 각 256Mi | 합계 250m · 1,280Mi |
| ai-service-py | 100m | 384Mi | |
| frontend | 10m | 64Mi | 정적 파일 서빙 |
| Harbor | 300m | 1,472Mi | trivy를 끈 값 |
| ArgoCD | 200m | 1,024Mi | server · repo-server · application-controller · redis |
| ArgoCD Image Updater | 20m | 64Mi | |
| istiod | 500m | 2,048Mi | ambient 모드 컨트롤 플레인. sidecar 모드로 켜면 파드마다 사이드카가 붙어 15개 안팎 기준 약 1.5 vCPU·2Gi가 추가로 든다 — 그래서 ambient를 기본으로 한다 |
| ztunnel | 100m | 128Mi | ambient 모드 노드 프록시. 단일 노드라 1개만 뜬다 |
| istio-ingressgateway | 100m | 128Mi | north-south 진입점 (A6) |
| Vault | 250m | 384Mi | 단일 노드 raft storage + Agent Injector |
| cloudflared | 10m | 64Mi | D 트랙에서 추가 |
| 쿠버네티스 시스템 | — | 약 900Mi | kubelet·CoreDNS |
| **합계** | **약 2.7 vCPU** | **약 10.3Gi** | |

기존 10GB 상한으로는 부족해 12GB로 올린다. 머신이 32GB라 이래도 macOS와
다른 앱에 20GB가 남는다. 12GB에서 약 1.7Gi가 남는데, `metrics-server`를 끈
채로도 여유가 크지 않으니 빠듯해지면 아래 순서부터 덜어낸다. `limits` 는
메모리에만 걸고 CPU에는 걸지 않는다. CPU 상한은 스로틀링을 만들어 로컬에서
원인을 찾기 어려운 지연을 만든다.

디스크는 40GB로 잡는다. 프록시 캐시가 이미지 25개 안팎을 쌓으므로 Harbor
레지스트리 PVC에만 20Gi를 준다.

더 줄여야 하면 이 순서로 덜어낸다.

1. frontend를 클러스터에서 빼고 `npm run dev` 로 호스트에서 돌린다
2. 지금 작업과 무관한 앱의 `replicas` 를 0으로 둔다

Harbor의 trivy를 켤 때는 2번으로 자리를 만든다.

---

# A 트랙 — 로컬 구동

## A0. 클러스터와 CI

- 위 명령으로 프로파일 `public-project` 를 만든다
- NestJS Dockerfile에 걸린 `platform: linux/x86_64` 강제를 걷어낸다. 현재는
  에뮬레이션으로 돌고 있어 빌드와 기동이 모두 느리다
- 저장소 세 곳에 빌드 워크플로를 추가한다

```yaml
permissions:
  contents: read
  packages: write
jobs:
  build:
    runs-on: ubuntu-24.04-arm      # arm64 네이티브. public 저장소는 무료다
    steps:
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

Harbor가 서기 전(C 트랙)까지는 `minikube image load` 로 노드에 직접 넣는다.

**완료 기준**: `kubectl get nodes` 가 `Ready` 를 반환하고, GHCR에 앱 이미지가
arm64로 올라간다.

## A1. public-infra 정합성 복구와 ArgoCD

"착수 전에 고쳐야 할 것" 1~3번을 처리하고, 클러스터 목록에 로컬 항목을 추가한다.

```yaml
clusters:
  - name: minikube
    url: https://kubernetes.default.svc   # ArgoCD가 자기 클러스터에 배포한다
    values:
      project: app
      environment: local
```

ArgoCD를 `argocd/install/install.yaml` 로 설치하고 bootstrap Application을 건다.

**완료 기준**: appset이 Application을 생성하고, 동기화 실패 사유가 "차트 없음" 이
아니라 "차트 내용" 으로 좁혀진다.

## A2. 상태 저장소 5종

- 기존 MySQL·Redis 차트에 `local.values.yaml` 을 추가한다
- Kafka·MongoDB·Qdrant 차트를 새로 쓴다. 전부 StatefulSet + PVC이고,
  스토리지 클래스는 minikube 기본 `standard`(local-path)를 쓴다

**완료 기준**: 다섯 개 모두 `Running` 이고, `kubectl port-forward` 로 각각
접속해 읽고 쓸 수 있다.

**가장 큰 미지수**: Kafka다. KRaft 단일 노드에서 `advertised.listeners` 를
잘못 잡으면 컨슈머가 오류 없이 조용히 붙지 못한다. 파드 내부용과 클러스터
서비스용 리스너를 나눠 잡고, 실제로 produce·consume을 한 번 태워 확인한다.

## A3. 서비스 메시 (Istio)

앱보다 먼저 깐다 — 이후 A5에서 앱 네임스페이스에 메시 레이블을 걸어야 하기 때문이다.

- ambient 모드로 설치한다. NestJS 5개·ai-service-py·frontend·gateway에
  상태 저장소까지 합치면 파드가 15개 안팎이라, sidecar 모드로 넣으면
  리소스가 크게 늘어난다 (위 표 참고)
- `istioctl install --set profile=ambient` 또는 helm 차트로 `istiod`·`ztunnel`을 올린다
- 앱이 속할 네임스페이스에 `istio.io/dataplane-mode=ambient` 레이블을 건다
- mTLS를 `STRICT`로 건다. 상태 저장소(MySQL·MongoDB·Kafka·Qdrant·Redis) 간
  트래픽도 포함한다
- `AuthorizationPolicy`로 서비스 간 접근을 좁히는 건 지금 범위에 넣지 않는다.
  이번엔 mTLS 활성화까지다

**완료 기준**: `istioctl proxy-status` 가 정상이고, 메시 안 파드 간 통신이
mTLS로 암호화된 채 동작한다.

## A4. 시크릿 (Vault)

API 키 4개(Groq·Anthropic·OpenAI·Google)와 DB 비밀번호가 필요하다.
**public-infra는 공개 저장소라 평문으로 커밋할 수 없다.** 주입 경로는
k8s Secret이 아니라 Vault로 통일한다 — 값이 클러스터 리소스로 한 번도
저장되지 않고 파드 안에만 파일로 들어온다.

- Vault를 단일 노드 raft storage로 설치한다
  (`helm install vault hashicorp/vault --set='server.ha.enabled=false'`)
- day-0에 손으로 `vault operator init`·`unseal`을 한 번 수행한다.
  **root 토큰과 unseal key는 git에 남기지 않는다** — 로컬 파일로만 보관한다.
  GitOps에서도 최초 시크릿 하나는 클러스터 밖에서 사람이 넣어야 하는 건
  기존 계획과 같다
- KV v2 시크릿 엔진을 켜고 `secret/app-secrets` 경로에 API 키·DB 비밀번호를 넣는다
- Vault Agent Injector를 설치하고 앱 파드에
  `vault.hashicorp.com/agent-inject: "true"` 애노테이션을 붙인다
- 앱 차트의 `vault.enabled` 를 `true` 로 바꾼다

**완료 기준**: 파드가 Vault Agent가 주입한 파일에서 API 키를 읽고, `.env` 값도
git도 k8s Secret도 평문을 갖지 않는다.

## A5. 애플리케이션

`app/server` 차트는 `command.execute` 와 `image` 가 이미 values로 빠져 있다.
NestJS 앱들은 **values 파일만 나누면 된다.** `ai-service-py` 와 `front` 는
포트·프로브·빌드 인자가 달라 차트를 따로 만든다.

compose의 비밀이 아닌 환경변수는 ConfigMap으로 옮기며 호스트명이 전부 바뀐다.
비밀값은 A4의 Vault 주입 경로를 쓴다.

| compose | 쿠버네티스 |
|---|---|
| `redis` | `infra-redis-master.infra-redis.svc.cluster.local` |
| `db` | `infra-mysql.infra-mysql.svc.cluster.local` |
| `mongo` | `infra-mongodb.infra-mongodb.svc.cluster.local` |
| `kafka:9092` | `infra-kafka.infra-kafka.svc.cluster.local:9092` |
| `qdrant:6333` | `infra-qdrant.infra-qdrant.svc.cluster.local:6333` |
| `http://ai-service-py:3004` | `http://local-app-ai-service-py.local-app.svc.cluster.local:3004` |

ArgoCD Image Updater를 설치해 새 태그를 감시하고 `local.values.yaml` 을 갱신하게
한다. 이것으로 사람이 태그를 고치는 단계가 사라진다.

**Image Updater는 ArgoCD 내장 기능이 아니라 별도 컨트롤러다.** ArgoCD API에
접근하기 편하도록 같은 `argocd` 네임스페이스에 자기 Deployment로 띄운다.
설정은 두 군데로 나뉜다.

- **컨트롤러 자신**: `argocd-image-updater-config` ConfigMap에 Harbor 레지스트리와
  로봇 계정 크리덴셜을 등록한다. Application을 갱신할 ArgoCD API 토큰
  (`argocd account generate-token`)과, write-back 방식이 `git`이라 `public-infra`에
  커밋할 git 크리덴셜도 필요하다. 폴링 주기는 `--interval` 로 준다
- **앱별 대상**: 각 앱의 `Application` 리소스에 애노테이션으로 건다. 지금
  구조에서는 `charts/appsets/` 가 생성하는 `Application` 이므로 appset
  템플릿(`appsets/infra-applicationset.yaml` 계열)에 앱마다 하나씩 넣는다

  ```yaml
  metadata:
    annotations:
      argocd-image-updater.argoproj.io/image-list: gateway=infra-harbor.test.com/app/gateway
      argocd-image-updater.argoproj.io/gateway.update-strategy: latest
      argocd-image-updater.argoproj.io/write-back-method: git
      argocd-image-updater.argoproj.io/git-branch: main
  ```

**Image Updater는 GHCR이 아니라 Harbor 레지스트리(로봇 계정)를 보게 설정한다.**
실제 pull 경로가 Harbor라서다. GHCR을 보게 하면 Harbor로 복제되기 전 태그를
`local.values.yaml` 에 먼저 써버려 `ImagePullBackOff` 가 뜬다. Harbor를 보면
Image Updater가 태그를 인식하는 시점 자체가 "Harbor에 이미 있는 시점"이라 이
경합이 애초에 생기지 않는다 (C 트랙과 연결).

**폴링 주기**: 기본값(Image Updater 2분, ArgoCD의 git 폴링 3분)을 그대로 두면
Harbor 복제 이후에도 최악 5분 가까이 더 걸릴 수 있다. 내 레지스트리라 rate
limit 걱정이 없으니 `--interval`을 30초~1분으로 낮춘다. 당장 반영을 보고
싶으면 기다리지 말고 `argocd app sync <app>` 으로 바로 당긴다 — Harbor 웹의
수동 Replicate 버튼과 같은 성격의 탈출구다.

**태그는 손으로 건드리지 않는다.** `local.values.yaml` 의 이미지 태그는
Image Updater만 쓴다. 사람이 Harbor에 아직 없는 sha를 직접 적으면
`ImagePullBackOff` 가 뜬다 (Harbor가 따라잡으면 kubelet 재시도로 저절로
복구되긴 한다) — 그래도 원칙은 "손대지 않는다"다.

**완료 기준**: 파드가 모두 `Ready` 이고 gateway가 각 백엔드에 gRPC·HTTP로 닿는다.
main에 push하면 배포까지 자동으로 이어진다.

## A6. 내부 진입점과 검증

Istio ingress gateway로 `gateway` 와 `frontend` 를 노출한다. nginx `ingress`
애드온은 쓰지 않는다 — north-south 트래픽도 같은 메시 안에서 처리한다
(A3). 아직 외부 공개는 아니다.

- Istio `Gateway`·`VirtualService` 리소스로 `public.local` 호스트를 라우팅한다
- `istio-ingressgateway` 서비스를 NodePort로 열거나 `minikube tunnel` 로
  LoadBalancer IP를 받는다
- `/etc/hosts` 에 게이트웨이 IP를 `public.local argocd.local` 로 매핑한다
- `public.local/` → frontend, `public.local/api` → gateway

임베딩을 Google로 바꾸며 `EMBEDDING_DIMENSION` 을 2560에서 3072으로 올린다.
`gemini-embedding-001` 의 기본 출력 차원이다. **Qdrant 컬렉션을 다시 만들고
문서를 재색인해야 한다** — 기존 벡터는 차원이 달라 그대로 쓸 수 없다.

아래 흐름을 실제로 태워 compose와 동등한지 확인한다.

- 문서 업로드 → 인제스트 완료 SSE 수신
- RAG 질의 → 토큰 스트리밍 → 근거·신뢰도 표시
- 답변 생성 중단
- 실시간 채팅 송수신
- 결제 생성·조회

**완료 기준**: 다섯 흐름이 모두 통과하고, docker-compose를 내린 상태에서도 동작한다.

---

# B 트랙 — 인증 통합과 하드닝

공개하려면 반드시 끝나야 하는 트랙이다. 지금 상태로 노출하면 다음이 성립한다.

- `POST /auth/login` 이 `{ uuid }` 문자열 하나만 받는다. 비밀번호도 검증도 없다
- `ACCESS_TOKEN_SECRET: 'personal project'` 가 public-infra에 커밋되어 있어
  토큰을 직접 위조할 수 있다
- 레이트 리밋이 없고 회원가입이 `nickName` 하나로 무제한이다. RAG 질의마다
  LLM API 요금이 나가므로 요금이 무제한으로 열린다

## B1. 게임 계정 제거와 인증 단일화

**관리자 회원가입을 유일한 가입 경로로 삼고 승인 메커니즘을 유지한다.**

이미 갖춰져 있는 것을 쓴다.

```ts
// apps/admin-server/src/user/user.service.ts
async signIn(email, password) {
  if (!(await user.checkPassword(password))) throw ...   // bcrypt 비교
  if (!user.activatedAt) throw '아직 활성화되지 않은 계정입니다'   // 승인 게이트
}
```

`Password` 값 객체(bcrypt)와 `joi-password-complexity` 복잡도 검증이 이미 있다.
새로 짤 암호화 로직이 없다.

**승인 게이트가 공개 시 남용을 막는 핵심 장치다.** 가입은 누구나 할 수 있지만
승인 전에는 로그인이 거부되므로, 낯선 사람이 RAG를 호출해 LLM 요금을 태울 수 없다.

작업 목록.

| 저장소 | 내용 |
|---|---|
| public-server | identity의 `Login`·`Register`·`GetGameAccount` gRPC 제거. gateway의 `/auth/*` 제거 |
| public-server | `game_account` 테이블과 account 모듈 제거 + 마이그레이션 |
| public-server | identity는 `SendMail` 만 남겨 메일 전용 서비스로 축소한다 |
| public-front | `AuthContext` 를 없애고 `AdminAuthContext` 로 단일화. 로그인·회원가입 화면 정리 |

**세션 매핑은 이미 된다.** `GatewayAuthGuard:67-72` 가
`uuid: user.uuid ?? String(user.id)`, `nickName: user.nickName ?? user.name` 으로
관리자 JWT도 `Session` 으로 정규화한다. 채팅 소켓(SPEC-017)도 같은 모양을 쓴다.
다만 관리자 계정의 `uuid` 가 `String(user.id)` 가 되므로 **값이 안정적인지
확인해야 한다** — RAG 세션·개인 프롬프트·답변 평가가 전부 이 값을 소유자 키로 쓴다.

**기존 데이터는 버려진다.** 옛 게임 계정 uuid로 저장된 대화 세션·프롬프트·평가는
새 계정에서 보이지 않는다. 로컬 환경이라 수용 가능하지만 명시해 둔다.

**나중에 할 것**: identity가 메일만 담당하게 되면 admin-server가 그 기능을 흡수해
서비스 하나를 줄일 수 있다. 이번 범위에 넣지 않는다.

## B2. 비밀값 교체

- `ACCESS_TOKEN_SECRET`·`REFRESH_TOKEN_SECRET`·`CHALLENGE_PRIVATE_KEY` 를
  `dev.values.yaml`·`staging.values.yaml` 에서 제거하고 Vault로 옮긴다 (A4와
  같은 경로). **값도 새로 만든다** — 공개 저장소에 노출된 값이다
- `harborAdminPassword: "Harbor12345"` 를 바꾼다
- `imageCredentials` 를 admin 계정에서 pull 전용 로봇 계정으로 바꾼다 (C 트랙과 연결)

## B3. 그 밖의 하드닝

| 항목 | 지금 | 바꿀 것 |
|---|---|---|
| TypeORM `synchronize` | `GAME_DB_SYNCHRONIZE: true`, `PERSONAL_DB_SYNCHRONIZE: true` | `false`. 엔티티에 맞춰 스키마를 자동 변경하며 컬럼을 지우는 방향으로도 동작한다 |
| 레이트 리밋 | 없음 | 가입·로그인·RAG 질의에 앱 레벨 제한을 건다 |
| 사용량 상한 | 없음 | 승인된 사용자에게도 일일 RAG 호출 한도를 둔다 |
| NetworkPolicy | 기본 허용 | 기본 차단 후 필요한 경로만 연다 |
| 관측 | 없음 | `charts/infra` 의 prometheus·grafana를 올린다. 공개하면 장애와 남용을 알아야 한다 |

**완료 기준**: 비밀번호 없이 로그인할 수 있는 경로가 없고, 커밋된 서명 키로
발급한 토큰이 거부되며, 승인되지 않은 계정이 RAG를 호출할 수 없다.

---

# C 트랙 — Harbor 운영

레지스트리 운영이 목적이다. A·D 와 독립이라 순서는 자유롭다.

## 이미지 흐름

```
GitHub Actions (ubuntu-24.04-arm)
  │  빌드 → push
  ▼
ghcr.io/bbbbb0413/<app>:<sha>
  │  Harbor 복제 규칙이 주기적으로 끌어옴
  ▼
Harbor  infra-harbor.test.com
  ├── app/            내 앱 이미지 (GHCR 복제본)
  ├── proxy-docker/   docker.io 프록시 캐시
  └── proxy-quay/     quay.io 프록시 캐시
  │
  ▼  클러스터의 pull 이 여기를 지난다
minikube
```

**GitHub 호스팅 러너는 로컬 Harbor에 닿을 수 없다.** 러너는 GitHub 클라우드에 있고
Harbor는 로컬 네트워크에 있다. GHCR이 양쪽 모두 닿는 중간 지점이 되고, Harbor가
밖으로 나가서 끌어오는 방향이라 Harbor를 인터넷에 노출하지 않아도 된다. 로컬 빌드
스크립트를 두지 않는다.

**왜 GHCR만 쓰지 않는가.** 클러스터가 GHCR에서 바로 받으면 Harbor는 장식이 된다.
프록시 캐시로 세울 때만 Harbor가 실제 경로에 선다. 부수 효과로 Docker Hub 익명
pull 제한(IP당 6시간 100회)도 피한다 — 클러스터를 하루에 몇 번 다시 만들면
실제로 걸린다.

## 구축

- `charts/infra/harbor` 에 `local.values.yaml` 을 추가한다. `trivy.enabled: false`,
  레지스트리 PVC 20Gi, 나머지 1Gi
- `harbor-cert/` 의 self-signed 인증서를 `infra-harbor-tls` Secret으로 넣는다

**신뢰 설정** — 여기서 막히는 경우가 가장 많다

| 대상 | 필요한 것 |
|---|---|
| minikube 노드(containerd) | `ca.crt` 를 신뢰해야 pull 이 된다. 없으면 `x509: certificate signed by unknown authority` 로 떨어진다 |
| 이름 풀이 | 호스트 `/etc/hosts` 와 **노드 안** 양쪽에 매핑해야 한다. 파드가 보는 DNS는 호스트와 별개다 |

CA는 **레지스트리 호스트 범위로만** 넣는다.
`/etc/containerd/certs.d/infra-harbor.test.com/` 같은 경로는 그 레지스트리에만
적용되지만, 시스템 신뢰 저장소에 넣으면 모든 도메인에 적용된다.
`harbor-cert/ca.crt` 에는 Name Constraints가 없고 `CA:TRUE` 라 도메인 제한이 없다.

## 운영 항목

- 프로젝트를 나눈다: `app`(내 이미지), `proxy-docker`·`proxy-quay`(프록시 캐시)
- **프록시 캐시 엔드포인트**를 등록하고 프로젝트를 프록시 모드로 만든다. 이후
  이미지 참조를 `infra-harbor.test.com/proxy-docker/library/mysql:8.0.23` 형태로 바꾼다
- **복제 규칙**을 만든다. `ghcr.io` 를 원본으로 하는 pull 기반 규칙을 cron
  `* * * * *` (1분 주기)로 건다. pull 기반 복제는 이벤트로 즉시 트리거되지
  않고 스케줄 또는 수동뿐이다 — **CI(GitHub 클라우드)가 Harbor(로컬)를 향해
  복제를 걸 수는 없다.** 앞서 "GitHub 호스팅 러너는 로컬 Harbor에 닿을 수
  없다"와 같은 이유로, 반대 방향(CI → Harbor)도 막혀 있다. 그래서 Harbor가
  스스로 자주 당겨오는 것으로 지연을 줄인다 — worst case가 5분에서 1분으로
  준다. GHCR 쪽 원본 등록에는 `read:packages` 권한의 PAT을 쓴다. 이 주기의
  API 호출량은 GitHub 인증 요청 기준 시간당 5,000회 한도에 한참 못 미쳐
  실질적인 제한에 걸리지 않는다 — Docker Hub 익명 pull처럼 IP당 정해진
  한도가 있는 게 아니라 토큰 단위라서 `proxy-docker` 캐시와는 다른 문제다
- **로봇 계정**으로 클러스터에 pull 전용 권한만 준다
- 보존 정책(태그 N개 유지), 가비지 컬렉션 주기, 프로젝트 쿼터

**부팅 순서** — Harbor 자신의 이미지는 Harbor를 통과할 수 없다. Harbor·ArgoCD·
Istio(istiod·ztunnel·ingressgateway)·Vault 이미지는 업스트림에서 직접 받는다.
앱 이미지에 `imagePullPolicy: IfNotPresent` 를 두어 Harbor가 죽어도 이미 받아 둔
이미지로 재기동되게 한다.

**완료 기준**: 클러스터가 로봇 계정으로 `infra-harbor.test.com/app/<앱>:<sha>` 를
pull 해 파드를 띄우고, `proxy-docker` 를 통해 mysql 이미지가 캐시된다.

---

# D 트랙 — 외부 공개

**B 트랙이 끝나기 전에는 시작하지 않는다.**

## 왜 Cloudflare Tunnel인가

```
사용자 브라우저
   │  https://<도메인>
   ▼
Cloudflare 엣지        ← TLS 종료, WAF, 레이트 리밋, Access
   │  ↑ cloudflared 가 밖으로 걸어 둔 연결
   ▼
cloudflared 파드      ← minikube 안
   │
   ▼
istio-ingressgateway → gateway / frontend
```

앱은 맥북에 그대로 있고 Cloudflare는 입구 역할만 한다. 클러스터가 **밖으로 나가는
연결만** 만들므로 공유기에 포트를 열지 않고 집 IP도 노출되지 않는다. 통신사 CGNAT
환경에서도 동작한다.

Tailscale Funnel은 도메인 없이 무료지만 WAF·레이트 리밋·Access 가 없어 방어를
전부 애플리케이션이 져야 한다. B 트랙에서 확인한 위험을 감안해 Cloudflare로 간다.

## 단계

1. **Quick Tunnel 로 구조 검증** — `cloudflared tunnel --url http://...` 하나로
   `*.trycloudflare.com` 주소가 즉시 나온다. 계정도 도메인도 필요 없다. 재시작마다
   주소가 바뀌므로 검증용으로만 쓴다
2. **도메인 구입** — Cloudflare Registrar는 도매 원가로 팔아 갱신 시 값이 뛰지
   않는다. 저렴한 TLD면 연 1~2만원 수준이다
3. **named tunnel 로 전환** — 터널 자격증명을 Secret으로 넣고, 어느 호스트명을
   어느 서비스로 보낼지 설정에 적는다
4. **노출 대상을 최소화한다** — frontend와 gateway **둘만** 적는다. ArgoCD·Harbor·
   DB·Kafka는 적지 않는다. 적지 않은 것은 외부에서 닿을 경로가 없다
5. **Access·WAF·레이트 리밋** — 관리 UI를 열어야 하면 Access로 SSO·OTP를 앞에 세운다.
   가입·로그인·RAG 경로에 엣지 레이트 리밋을 건다

## 알아 둘 것

- **맥북이 잠들면 서비스가 끊긴다.** 포트폴리오·데모 수준이면 감수할 만하지만
  상시 가용성을 기대할 수 없다
- **엣지 레이트 리밋은 보조 수단이다.** 무료 플랜은 규칙 1개다. 앱 레벨 제한(B3)이
  주 방어선이고 엣지는 그 앞에 덧대는 것이다
- **호스트명은 숨겨지지 않는다.** TLS 인증서 발급 기록이 인증서 투명성(CT) 로그에
  공개된다. "주소를 안 알려주면 된다"는 통하지 않는다

**완료 기준**: 외부 네트워크에서 도메인으로 접속해 가입·승인·로그인·RAG 질의가
동작하고, ArgoCD·Harbor·DB는 외부에서 닿지 않는다.

---

## 이후 (범위 밖)

- identity를 admin-server가 흡수해 서비스 하나 줄이기
- Harbor 취약점 스캔(trivy) 켜기와 스캔 차단 정책
- Istio `AuthorizationPolicy`로 서비스 간 접근 좁히기
- Harbor 복제 규칙으로 원격 클러스터 레지스트리와 동기화
- 원격 클러스터(`10.10.100.111:6443`)와 values 공유 검증
