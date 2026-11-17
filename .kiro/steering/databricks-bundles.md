---
inclusion: fileMatch
fileMatchPattern: 'databricks-bundles/**'
---

# DATABRICKS-BUNDLES

Databricks Job 정의를 Databricks Asset Bundles(DABs)로 Git에서 관리하는 레포. Airflow DAG 파일처럼 Job 구성만 담고, 실행 코드(노트북)는 소스 레포에 둔다(`git_source`로 실행 시점 pull). 레포 안의 Go CLI `databricks-bundle`이 번들 간 Job 참조 치환, 의존 순서 배포, Job 이름 중복 검사, 삭제된 번들 정리를 맡는다.

- **경로**: `/Users/phddy/Projects/dx/databricks-bundles`
- **모듈 경로**: `github.com/mz-team-ax/databricks-bundles`
- **Go 버전**: 1.26.3
- **원격**: `git@github.com:mz-team-ax/databricks-bundles.git`
- **워크스페이스**: `https://dii-dii.cloud.databricks.com` (단일 워크스페이스를 target이 공유, `name_prefix`로 구분)
- **databricks CLI**: v1.20.0 (Jenkinsfile `CLI_VERSION`과 동일)

> 운영 규칙과 명령 사용법의 1차 소스는 레포 `README.md`다. 이 문서는 작업할 때 알아야 할 구조, 설계 결정, 검증으로 확인한 DABs 동작을 다룬다. 공통 Go 규칙은 `go-conventions.md`를 따른다.

## 레포 구조

```
common/targets.yml     # 공통 target (local 기본값, dev/stg/prd), name_prefix, run_as, root_path, tags
bundles/tags.yaml      # bundles 하위 공통 tags/variables
bundles/<경로>/        # databricks.yml이 있는 디렉터리 = 번들. 임의 깊이 (예: bundles/a/aa/aaa/test2)
  databricks.yml
  resources/*.yml      # Job 정의
cmd/cli/               # databricks-bundle CLI
scripts/changed-bundles.sh  # 기존 변경 감지 스크립트 (Jenkinsfile이 사용, CLI changed로 대체 예정)
Jenkinsfile            # 기존 파이프라인. deployEnv() 정의 없음, CLI 연동 전
```

- 파일럿 `bundles/test`의 Job `test`가 `hello → check → link/link2/link3 → merge` 구조이고, `link*` task가 `a/aa/aaa/test2`, `a/aa/aaa/test3` 번들의 Job을 `run_job_task`로 호출한다.
- dev/stg/prd `root_path`는 `/Workspace/Users/phddy/.bundle/${bundle.name}/${bundle.target}`로 고정. local은 기본값(배포자 홈).

## 핵심 설계 결정

- **번들 간 참조 키워드**: `${bundles.<번들 경로(. 구분)>.jobs.<리소스 키>.id}`. DABs의 `${resources...}`는 같은 번들 안에서만 동작하므로 CLI가 배포 시 job_id로 치환한다.
  - 의존성은 키워드에서 자동 추출. 별도 선언 파일(`deps.yml` 등)을 만들지 않기로 했다.
  - **참조된 번들은 먼저 배포되어 있기만 하면 된다.** 참조된 번들이 바뀌어도 참조하는 번들은 재배포하지 않는다(호출은 Databricks가 job_id로 한다). `changed`는 변경 번들 + 그 번들이 참조하는 번들(전이)만 포함한다. 역방향(참조하는 쪽)은 넣지 않는다.
  - 참조 번들은 이미 배포돼 있어도 항상 함께 배포한다(멱등, 워크스페이스 조회 불필요).
  - 짧은 형식 `${bundles.<경로>}`는 참조 번들 Job이 1개일 때만 가능해서 보류(Job 추가 시 무관한 번들 배포가 깨짐).
- **치환은 임시 복사본에서**: 키워드가 있는 번들만 레포 전체(`.git`, `.databricks`, `.idea` 제외)를 임시 디렉터리에 복사해 치환 후 배포한다. 원본 작업 트리는 건드리지 않으므로 치환 결과를 커밋하거나 되돌릴 일이 없다. job_id는 target마다 다르고 Job 재생성 시 바뀌므로 Git에는 키워드만 둔다.
  - 한계: 복사본에 `.git`이 없어 배포 메타데이터의 git 정보(`bundle_root_path`, commit, branch)가 비어 `orphans --destroy`로 정리할 수 없다. `git worktree` 등으로 개선하는 것이 남은 과제.
- **번들 식별자**: CLI 인자와 내부 키는 `bundles/` 기준 경로(`a/aa/aaa/test2`). 키워드에서만 `.` 구분. 경로 세그먼트는 `[A-Za-z0-9_-]`만 허용(`.` 불가).
- **`bundle.name` 유일성**: 배포 경로가 `<root>/<bundle.name>/<target>`이라 같은 이름이면 state를 덮어쓴다. CLI가 로딩 시 중복을 에러로 막는다.
- **삭제된 번들 탐지는 워크스페이스 기준**: 별도 트리/레지스트리 파일을 두지 않고 `<root>/` 바로 아래 디렉터리 목록(=배포된 `bundle.name`)과 레포를 비교한다. Databricks가 관리하는 state가 실제와 항상 일치하기 때문. 원래 경로와 job_id는 `<root>/<이름>/<target>/state/metadata.json`(`config.bundle.git.bundle_root_path`, `config.resources.jobs.*.id`)에서 얻는다.
- **삭제는 명시적 플래그로만**: `orphans`는 기본이 조회, `--destroy`일 때만 삭제(y/N 확인, `--auto-approve`로 생략). 체크아웃 기준 판단이라 오래된 브랜치에서 실행하면 정상 번들도 고아로 보이기 때문. 자동 삭제는 main 빌드에서만.
- **이름 중복 검사**: 배포 전 레포 전체 번들의 최종 이름(`name_prefix` 적용)을 `bundle summary`로 만들고, 레포 내 중복과 워크스페이스(`jobs list` 1회) 중복이 하나라도 있으면 아무것도 배포하지 않는다. **이미 배포되어 있고 이름이 그대로인 Job은 검사에서 제외**(이번 배포로 새 중복이 생기지 않음). 같은 번들이 배포한 Job은 `settings.deployment.metadata_file_path == <state_path>/metadata.json`으로 구분한다.

## CLI 구조 (`cmd/cli`, package main)

| 파일 | 역할 |
|---|---|
| `main.go` | cobra 명령 정의. 루트 `databricks-bundle`, 하위 `changed`/`graph`/`render`/`deploy`/`orphans` |
| `cli.go` | `databricksCli{v *viper.Viper, repo *repo}`. `setup`(PersistentPreRunE)에서 플래그 바인딩 + 레포 로드. `changed`/`graph`/`render`/`deploy` 핸들러 |
| `repo.go` | `loadRepo`(번들 탐색, 키워드 추출, 검증), `plan`(DFS 위상 정렬 + 순환 검출), `substitute`, `ownerOf` |
| `changed.go` | git diff 변경 감지. `bundlesAt(base)`로 기준 커밋 시점 번들 트리를 읽어 삭제된 번들 파일을 매핑 |
| `stage.go` | 임시 복사본(`ensure`, 지연 생성)과 번들별 치환 파일 작성(`dirFor`) |
| `names.go` | `checkJobNames`, `plannedJobs`(summary 4개 병렬) |
| `orphans.go` | `findOrphans`, `destroyOrphan`(files/ 복원 → `bundle destroy`), `confirm` |
| `databricks.go` | `databricks` CLI `os/exec` 래퍼(`deploy`, `summary`, `listJobs`, `listWorkspace`, `exportWorkspace`, `exportDir`, `destroy`) |

- Databricks 연동은 Go SDK가 아니라 **`databricks` CLI 호출**(번들 deploy/summary/destroy는 SDK 미지원). `-p` 프로필은 모든 호출에 전달.
- 출력 규칙: 목록/JSON은 stdout, 상태 로그(`logf`)와 databricks CLI 출력은 stderr. `changed` 결과를 파이프로 쓰기 위함.
- viper 환경변수 접두사 `DBB_`(예: `DBB_TARGET`). 원래 스캐폴드의 접두사 없는 `AutomaticEnv`는 `TARGET` 같은 일반 이름과 충돌해 바꿨다.
- `zap`은 쓰지 않는다(`zap.L()` 기본값이 no-op이라 에러가 안 보였음). 에러는 `main`에서 stderr로 출력.
- 로딩 시 검증(`loadRepo`)은 모든 명령에 적용된다: 없는 번들/Job 키 참조, 자기 참조, 순환, `bundle.name` 중복, Job `name` 원문 중복, 경로 문자.
- 빌드 확인: `go vet ./... && go build -o /tmp/databricks-bundle ./cmd/cli`. 레포 루트에 바이너리(`cli`, `databricks-bundle`)를 만들면 커밋하지 않는다.

## 검증으로 확인한 DABs 동작

- **병합 우선순위**: `databricks.yml` 본문 < include 파일(적힌 순서, 뒤가 우선). 번들 본문의 최상위 `presets`/`variables`는 공통 파일의 같은 키를 못 덮는다.
  - 예: 본문 `presets.tags.team: ABC` → 결과 `AX`. 본문 `variables.git_branch.default: main` → `bundles/tags.yaml`의 `master`가 적용.
  - 번들 단위 덮어쓰기는 리소스 `tags`, include 맨 뒤 파일, 또는 `targets.<t>.presets`로 한다.
- 리소스 `tags`는 `presets.tags`보다 우선.
- target에서 `name_prefix: ""`는 미지정으로 처리되어 최상위 값(`[DEV <사용자>] `)이 적용된다.
- 미치환 키워드(`${bundles...}`)가 있어도 `bundle validate`/`summary`는 통과한다(문자열로 남음). 그래서 이름 검사와 `orphans`는 원본 경로에서 summary를 돌린다. 키워드 번들을 CLI 없이 deploy하면 안 된다.
- `bundle summary -o json`: `resources.jobs.<키>.{id,name}`(name은 prefix 적용), `workspace.state_path`. 미배포면 id가 비어 있다.
- `jobs list -o json`: `settings.deployment.metadata_file_path`로 배포한 번들 state를 알 수 있다. UI 생성 Job은 `deployment`가 없다. 워크스페이스 Job 약 300개, 전체 조회 약 2.5초.
- `.databricks/`(로컬 캐시)를 다른 번들에서 복사하면 원본 번들의 state(`resources.json`의 `__id__`)를 가리켜 원본 Job을 덮어쓸 수 있다. 번들 복사 시 제외.

## 작업 시 유의사항

- **실제 deploy/destroy는 사용자 확인 후에만.** 워크스페이스에 Job이 생기거나 실행 이력과 함께 삭제된다. 검증은 `--dry-run`, `render`, `graph`, `validate`, `summary`로 하고, 시나리오 테스트는 `/tmp`에 레포를 복사해 `--root`로 돌린다(원본 작업 트리 수정 금지).
- 사용자가 `bundles/` 아래 테스트 번들(test2/test3/test5 등)을 직접 만들고 지우며 실험 중이다. 내가 만들지 않은 번들 파일은 문제(include 경로, `bundle.name` 중복, 복사된 `.databricks`)를 발견하면 알리고, 필요한 최소 수정만 한다.
- 현재 남은 테스트 값: stg/prd `name_prefix`의 `1`, `bundles/tags.yaml`의 `abc: 123`, `git_branch: master`.
- 남은 과제: Jenkinsfile CLI 연동(`deploy --base $GIT_PREVIOUS_SUCCESSFUL_COMMIT`, main에서만 `orphans --destroy --auto-approve`), 키워드 번들 git 정보 기록, `orphans --destroy` 실제 삭제 검증.
