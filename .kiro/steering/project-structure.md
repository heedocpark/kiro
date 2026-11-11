# 주로 사용하는 프로젝트

워크스페이스 `/Users/phddy/Projects/dx` 아래에는 다수의 프로젝트가 있지만, 작업은 주로 아래 프로젝트들을 대상으로 한다.

| 프로젝트 | 경로 |
|----------|------|
| `aidea-api-agent` | `/Users/phddy/Projects/dx/aidea-api-agent` |
| `aidea-api-common` | `/Users/phddy/Projects/dx/aidea-api-common` |
| `aidea-api-spinx` | `/Users/phddy/Projects/dx/aidea-api-spinx` |
| `aidea-api-workflow` | `/Users/phddy/Projects/dx/aidea-api-workflow` |
| `hexim` | `/Users/phddy/Projects/dx/hexim` |

## 작업 시 유의사항

- 각 프로젝트는 독립된 디렉토리이며, 별도의 git 저장소일 수 있다. 커밋/브랜치 작업은 해당 프로젝트 단위로 처리한다.
- 파일 접근은 위 실제 경로를 그대로 사용한다.

## 커밋 규칙

- **스티어링 파일(`/Users/phddy/Projects/dx/kiro/.kiro/steering/*.md`)을 변경하면 그 저장소(`/Users/phddy/Projects/dx/kiro`)에 바로 git 커밋한다.**
  - 내가 변경한 스티어링 파일만 지정해서 stage한다(`git add <파일>`). 사용자가 따로 수정 중인 파일은 함께 커밋하지 않는다.
  - push는 하지 않는다(요청 시에만).
- **스티어링 외 코드/문서 저장소(hexim, aidea-api-* 등)는 커밋하지 않는다.** 사용자가 명시적으로 요청할 때만 커밋한다.
