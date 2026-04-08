# sweetbook-harness

`sweetbook-harness`는 SweetBook 프로젝트의 하네스 엔지니어링과 통합 실행을 위한 저장소입니다.

이 저장소의 역할:

- 제품 기획, 워크플로우, 에이전트 역할 문서 관리
- backend와 frontend를 함께 실행하는 Docker 허브 제공
- 과제 제출용 통합 테스트 절차 제공

## 포함 내용

- `agents/`
- `workflow/`
- `group-based-product-flow-spec.md`
- `project-foundation.md`
- `sweetbook-api-notes.md`
- `docker-compose.yml`
- `.env.example`
- `TESTING.md`

## 실행 구조

이 저장소만으로 앱이 실행되지는 않습니다.

아래 저장소들이 같은 부모 폴더 아래에 함께 있어야 합니다.

```text
workspace/
├─ sweetbook-backend/
├─ sweetbook-frontend/
└─ sweetbook-harness/
```

현재 `docker-compose.yml`은 아래 경로를 기준으로 build합니다.

- `../sweetbook-backend`
- `../sweetbook-frontend`

## 빠른 시작

실제 Docker 실행 방법과 `.env` 설정은 아래 문서를 참고하세요.

- [TESTING.md](C:/Users/user/my-projects/sweetbook/sweetbook-harness/TESTING.md)

## 관련 저장소

- [sweetbook-backend/README.md](C:/Users/user/my-projects/sweetbook/sweetbook-backend/README.md)
- [sweetbook-frontend/README.md](C:/Users/user/my-projects/sweetbook/sweetbook-frontend/README.md)

## 현재 범위

이 저장소는 문서 저장소이면서 동시에 제출용 실행 허브 역할을 합니다.
