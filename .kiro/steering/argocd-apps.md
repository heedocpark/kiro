---
inclusion: fileMatch
fileMatchPattern: 'argocd-apps/**'
---

# ARGOCD-APPS

팀 GitOps 저장소. ArgoCD가 바라보는 Kubernetes 매니페스트를 Kustomize base/overlay로 관리한다. AI 관련 앱(앤써/AnXer 스택)은 `aidea/` 아래에 모여 있다.

- **경로**: `/Users/phddy/Projects/dx/argocd-apps`
- **레포(values 참조용)**: `git@github.com:mz-team-ax/argocd-apps.git` (`targetRevision: master`)
- **클러스터**: `ax-cluster-2026`

## 저장소 레이아웃

- **`aidea/`** — AI 제품 앱 그룹. 앱 1개당 폴더 1개(Kustomize 프로젝트 = base + env overlay). 주 작업 대상.
- **`helm/`** — 클러스터 플랫폼 애드온(argo-rollouts, external-secrets, grafana, karpenter, cert-manager 등). 각 폴더에 `application.yaml`(upstream Helm 차트 + git values 멀티소스).
- **`readyset/`, `readyset-dev/`** — ReadySet(DB 캐시) 매니페스트, env별 분리.
- **`terraform/`** — AWS/EKS 인프라 IaC. Kustomize/ArgoCD 앱 플로우 밖.

## ArgoCD Application 연결 (중요)

- 레포 안의 `kind: Application`은 **`helm/*/application.yaml`에만** 있다. `kind: ApplicationSet`·app-of-apps 매니페스트는 **없다**.
- **`aidea/*` 앱을 가리키는 Application 매니페스트는 레포에 없다.** 이 앱들의 ArgoCD Application은 **레포 밖(ArgoCD UI)에서 관리**되며, 각 Application이 특정 overlay 경로(예: `aidea/aidea-api-spinx/dev`, `aidea/aidea-api/live`)를 직접 가리킨다.
- 컨벤션: **"앱×환경당 Application 1개, path = 해당 overlay 디렉토리"**. 그래서 overlay 디렉토리 구조/이름을 바꾸면 ArgoCD UI 쪽 설정도 따라 바꿔야 한다.

## Kustomize base/overlay 구조

표준 패턴(예: `aidea/aidea-api-spinx/`):

- **`base/`** — 순수 매니페스트 + `kustomization.yaml`. 보통 `deployment.yaml`/`service.yaml`/`ingress.yaml`/`hpa.yaml`, 필요 시 `cronjob.yaml`.
- **overlay `dev/`·`stg/`·`live/`** — 각자 `kustomization.yaml`.

**overlay 체이닝이 균일하지 않다.** 두 스타일이 공존하니 수정 전 `resources:`를 꼭 확인한다.

- **dev→base, live→base, stg→live**: spinx·aidea-api가 이 방식. `stg`는 `resources: [../live]`로 **live를 상속한 뒤** 불필요한 걸 `$patch: delete`로 지운다. → **base/live에 추가한 리소스는 stg로 자동 흘러들어간다. stg에서 원치 않으면 명시적으로 삭제해야 한다.**
- **일부 앱은 overlay가 `dev/`만** 있다(aidea-api-agent, aidea-api-workflow — stg/live 없음).

앱별 편차(수정 전 확인):

- **cronjob이 base에 있는 앱은 `aidea-api-spinx`뿐**(`base/cronjob.yaml`, CronJob 3개).
- **cronjob이 live overlay에만 있는 경우**: `aidea-api`(`live/cronjob.yaml` → `aidea-crawler-trigger`), `aidea-api-thread`(`live/cronjob.yaml` → `aidea-api-thread-delete-deploy`). overlay `resources: [./cronjob.yaml]`로 추가.
- **ingress 없음**: `aidea-crawler`, `aidea-calculator`(Spark, `daily/`+`monthly/` 레이아웃).
- **hpa 없음(base)**: aidea-api-agent, aidea-api-workflow.
- **aidea-api 주의**: `base/ingress.yaml`이 디스크엔 있으나 `base/kustomization.yaml` `resources`에 안 올라가 **사실상 미사용**. dev/stg host 패치도 주석 처리됨.
- **`aidea-common`** — Deployment 없는 공유 앱. `account.yaml`(SA), `role.yaml`, `externalsecret.yaml`(공유 시크릿), overlay의 `volume.yaml`(PV/PVC). 다른 앱이 참조하는 클러스터 공유 객체를 제공.
- **`aidea-mcp`** — 하위 앱 3개(`admin/`·`kiwi/`·`search/`) 각자 base+overlay.
- **`aidea-logger`, `aidea-tools/browserless`** — base/overlay 분리 없는 평면 kustomization.

## 환경별 값

| | dev | stg | live |
|---|---|---|---|
| namespace | `aidea-dev` | `aidea-stg` | `aidea` |
| nameSuffix | `-dev` | `-stg` | (없음) |
| ENV literal | `ENV=dev` | (live 상속, MIN_REPLICAS만 override) | `ENV=live` |
| volumeMount subPath | `dev` | `stg` | `live` |
| image newName | `.../aidea/<app>/dev` | (live 이미지 상속) | `.../aidea/<app>/live` |
| ingress host | `api.dev.anxer.ai` | `api.stg.anxer.ai` | `api.anxer.ai` |

- **이미지 핀**: `images: [{name: <base-image>, newName: ax-harbor.cjmezzomedia.com/aidea/<app>/<env>, newTag: "<build#>"}]`. **stg는 `images`를 재선언하지 않고 live 이미지/태그를 상속**한다(stg가 `../live` 위에 빌드되므로).
- **ingress host**: base는 항상 **운영 호스트**. overlay가 `/spec/rules/0/host`를 패치. live=bare(`api.anxer.ai`/`anxer.ai`/`mcp.anxer.ai`), stg=`*.stg.anxer.ai`, dev=`*.dev.anxer.ai`/`*-dev.anxer.ai`. 모두 AWS ALB(`ingressClassName: alb`, `group.name: aidea-alb-2026`, internal, ACM, health `/health`).
- **JSON6902 patch 대상 kind**: dev는 `Deployment`, stg는 `Rollout`(live의 rollout 변환을 상속하므로), live는 파일 패치 `transform_to_rollout.json`(Deployment→Rollout canary)·`replace_hpa.json`(HPA scaleTargetRef→Rollout) 적용.

## ConfigMap / Secret 컨벤션

- **`generatorOptions: disableNameSuffixHash: true`**를 거의 모든 kustomization에 둔다. ConfigMap 이름에 content hash suffix가 안 붙어야 cronjob/deployment가 **고정 이름**으로 참조 가능하다. (빼면 참조가 깨진다.)
- 앱별 config map 이름: **`<app>-config-map`**(예: `aidea-api-spinx-config-map`). base가 전체 literal 선언 → overlay는 `configMapGenerator: [{behavior: merge, literals: [...], name: <동일 이름>}]`로 env 값 override/추가. **merge 키는 동일한 map 이름**.
- **공유 config map `aidea-config-map`**: `aidea/aidea-common/base/kustomization.yaml`이 생성(`ENV=`, `AWS_DEFAULT_REGION=ap-northeast-2`, `LOG_BUCKET=aidea-log`, `LOG_PATH=/log/aidea/`). 앱은 `envFrom: configMapRef: name: aidea-config-map`으로 소비.
- **secretGenerator는 안 쓴다.** 시크릿은 `aidea/aidea-common/base/externalsecret.yaml`의 `ExternalSecret`에서 온다: `aidea-api-secret`(ClusterSecretStore `aws-clustersecretstore`, key `live/aidea-api`; dev overlay가 `dev/aidea-api`로 패치), `harbor-secret`(dockerconfigjson pull secret). 앱은 `secretRef: name: aidea-api-secret`으로 참조.

## 공통 리소스 컨벤션

- **이미지 레지스트리**: 앱 `ax-harbor.cjmezzomedia.com/aidea/<app>/<env>`, 유틸 `ax-harbor.cjmezzomedia.com/etc/...`(예: `etc/alpine:...`).
- **`imagePullSecrets: [{name: harbor-secret}]`** — 모든 pod spec(deployment·cronjob).
- **`serviceAccountName: aws-access-sa`** — IRSA(`arn:aws:iam::897722678530:role/eks-aws-services-role`, `aidea-common/base/account.yaml` 정의). HPA용 `aidea-hpa-sa`도 있음.
- **tz-config 볼륨(Asia/Seoul)**: `hostPath: /usr/share/zoneinfo/Asia/Seoul` → `/etc/localtime`. 모든 deployment·cronjob에 존재.
- **app-log PVC**: `claimName: app-log-pvc` → `/log/aidea` 마운트, **`subPath`를 env별로 패치**(dev/stg/live). PVC/PV는 `aidea-common/<env>/volume.yaml`(`local-storage`, RWX, hostPath `/log/aidea`).
- **HPA minReplicas 와이어링**: config map의 `MIN_REPLICAS` literal을 `replacements`로 `HorizontalPodAutoscaler.spec.minReplicas`에 복사. HPA 있는 모든 앱·overlay 공통.
- spinx: `securityContext` uid/gid/fsGroup `1000`, `topologySpreadConstraints`(hostname+zone), `preStop sleep 30` + `terminationGracePeriodSeconds: 300`.
- **PDB**: live overlay가 `pdb.yaml`(`minAvailable: 1`) 추가, stg는 `$patch: delete`로 제거.

## 새 CronJob 추가 시 체크리스트 (중요 — 과거 누락 사례 있음)

CronJob은 현재 `aidea-api-spinx/base/cronjob.yaml`에 3개:
`aidea-api-spinx-daily-cronjob`(alpine curl, `10 15 * * *`), `aidea-api-spinx-mcp-cronjob`(`/app/mcp-checker`, `*/5 * * * *`), `aidea-api-spinx-mcp-reviewer-cronjob`(`/app/mcp-reviewer`, `2,32 8-17 * * 1-5` + `timeZone: Asia/Seoul`).

base에 CronJob을 추가하면 모든 env로 상속되므로, overlay의 선택적 처리를 **직접 미러링**해야 한다:

1. **dev overlay — app-log subPath 패치.** `app-log`를 마운트하면 dev `kustomization.yaml`의 CronJob subPath 패치 대상에 새 cronjob을 포함시킨다. **patch target `name`은 정규식**이라 여러 cronjob을 `|`로 묶어 한 패치로 처리한다(현재 선례):
   ```yaml
   - patch: |
       - op: replace
         path: /spec/jobTemplate/spec/template/spec/containers/0/volumeMounts/0/subPath
         value: dev
     target:
       kind: CronJob
       name: aidea-api-spinx-mcp-cronjob|aidea-api-spinx-mcp-reviewer-cronjob
   ```
   **경로 주의**: Deployment는 `/spec/template/...`, CronJob은 `/spec/jobTemplate/spec/template/...`.
   **주의**: 새 cronjob이 `app-log`를 마운트하는데 이 정규식에 안 넣으면 dev에서 subPath가 안 바뀌어 로그가 env 분리 없이 쌓인다. `daily-cronjob`은 alpine curl이라 app-log를 안 쓰므로 패치 대상에서 빠져 있는 게 정상.
2. **stg overlay — CronJob은 돌리지 않는다(정책).** **stg는 live의 최종 테스트 환경이고 DB도 live와 동일한 걸 쓴다.** 따라서 stg에서 배치가 돌면 live DB를 건드리게 되므로 **stg에서는 모든 CronJob을 돌리지 않는다**(mcp-checker/mcp-reviewer 포함). stg는 `../live`를 상속해 base/live의 cronjob이 자동 포함되므로, stg에서 안 돌리려면 `$patch: delete`로 명시 제거해야 한다. 새 cronjob을 base/live에 추가하면 stg에서도 삭제 블록을 반드시 추가한다.
   ```yaml
   - patch: |
       $patch: delete
       apiVersion: batch/v1
       kind: CronJob
       metadata:
         name: <new-cronjob-name>
     target:
       kind: CronJob
       name: <new-cronjob-name>
   ```
   (선례: stg가 `aidea-api-spinx-daily-cronjob`을 이렇게 삭제.)
3. **컨벤션 맞추기**: `serviceAccountName: aws-access-sa`, `imagePullSecrets: [harbor-secret]`, `tz-config` 마운트, `envFrom`에 올바른 `<app>-config-map` + `aidea-config-map` + `secretRef aidea-api-secret`. 로그 볼륨 쓰면 `app-log-pvc` + env별 `subPath`.
4. **스케줄 타임존**: CronJob 스케줄은 kube-controller-manager(UTC) 기준으로 평가된다. KST 기준으로 돌리려면 `spec.timeZone: "Asia/Seoul"`를 넣는다(클러스터 1.25+). 컨테이너의 tz-config 마운트는 컨테이너 내부 시계일 뿐 스케줄 평가와 무관.

## 작업 시 유의사항

- base 수정은 전 환경에 퍼진다. overlay 체이닝(특히 stg→live)을 먼저 확인하고, 환경별로 원하는 결과가 나오는지 역으로 추적한다.
- 매니페스트 검증은 `kustomize build <overlay-dir>`(또는 `kubectl kustomize`)로 렌더 결과를 확인한다. ArgoCD가 실제 적용하는 산출물이 이것이다.
- live 변경은 운영 영향이 크다. 이미지 태그 범프·replica·cronjob 스케줄 변경은 의도와 블라스트 반경을 확인하고 진행한다.

## 함정: patch target `name`은 글롭이 아니라 정규식(RE2)

여러 리소스를 한 패치로 묶을 때 자주 틀리는 포인트. **매칭 방식이 patch 형태에 따라 다르다.**

- **inline patch**(= `patch:`만 있고 `target:` 없음): patch 본문의 `metadata.name`이 base 리소스와 **정확히 문자열 일치**해야 한다. 정규식/글롭 안 먹는다. `$patch: delete`도 마찬가지 — `name`에 `A|B`를 적으면 "`A|B`라는 이름" 리소스를 찾으므로 아무것도 안 지워지고(대상 못 찾아) **build가 실패**한다.
- **`target:` 블록 사용**: `target.name`이 **정규식(RE2)**으로 해석된다. 여기서만 `A|B` alternation, `.*` prefix 등이 동작한다.

그래서 **여러 cronjob을 묶어 삭제/패치하려면 반드시 `target:` 블록을 써야 한다.**

```yaml
# 여러 CronJob 삭제 (stg에서 mcp 배치 둘 다 제거) — target 블록 필수
- patch: |-
    $patch: delete
    apiVersion: batch/v1
    kind: CronJob
    metadata:
      name: ignored        # target 사용 시 이 name은 무시됨(아무 값)
  target:
    kind: CronJob
    name: aidea-api-spinx-mcp-checker-cronjob|aidea-api-spinx-mcp-reviewer-cronjob
```

- `name: foo-mcp-*` 같은 **글롭은 동작하지 않는다**(정규식에서 `*`는 앞 토큰 반복). prefix 매칭이 필요하면 `foo-mcp-.*`를 쓰되, 범위가 넓어져 의도치 않은 리소스까지 걸릴 수 있다. JSON6902 `replace`는 대상에 해당 path가 없으면 에러나므로, **명시적 alternation(`A|B`)이 가장 안전**하다.
