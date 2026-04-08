# SweetBook 통합 실행 가이드

이 문서는 `sweetbook-harness`를 기준으로 SweetBook 전체 프로젝트를 Docker로 실행하고 검증하는 방법을 설명합니다.

## 준비할 저장소

아래 세 저장소를 같은 부모 디렉토리에 둡니다.

```text
workspace/
├─ sweetbook-backend/
├─ sweetbook-frontend/
└─ sweetbook-harness/
```

중요:

- `sweetbook-harness/docker-compose.yml`은 위 구조를 전제로 합니다
- backend와 frontend가 다른 위치에 있으면 `build` 경로를 수정해야 합니다

## 환경 변수 설정

`sweetbook-harness` 디렉토리에서 `.env.example`를 복사해 `.env`를 만듭니다.

예시:

```powershell
cd sweetbook-harness
Copy-Item .env.example .env
```

`.env` 예시:

```env
SWEETBOOK_ENV=sandbox
SWEETBOOK_API_KEY=your-sandbox-key
```

설명:

- `SWEETBOOK_API_KEY`가 없으면 일반 앱 흐름은 확인할 수 있어도 SweetBook estimate / submit 은 실패할 수 있습니다
- `.env`는 로컬 전용 파일이므로 Git에 올리지 않습니다

## Docker 실행

`sweetbook-harness`에서 아래 명령을 실행합니다.

```powershell
docker compose up --build
```

백그라운드 실행:

```powershell
docker compose up --build -d
```

상태 확인:

```powershell
docker compose ps
```

## 접속 주소

- frontend: `http://localhost:5173`
- backend health: `http://localhost:3000/health`
- postgres: `localhost:5432`

## 기본 계정

기본 시드 계정:

- 아이디: `demo`
- 비밀번호: `sweetbook123!`

회원가입도 가능하므로 새 계정을 만들어 테스트할 수도 있습니다.

## 권장 수동 테스트 순서

1. 랜딩 접속
   - `http://localhost:5173`
2. 회원가입 또는 로그인
3. 그룹 생성
4. 이벤트 생성
5. 이벤트에서 사진 업로드
6. 좋아요 확인
7. 책에 넣을 사진 선택
8. 스프레드 구성
9. 주문 단계에서 SweetBook 견적 확인

## 컨테이너 구성

### `postgres`

- 이미지: `postgres:16-alpine`
- healthcheck 포함
- volume 사용

### `backend`

- `../sweetbook-backend` 기준으로 build
- postgres 준비 후 시작
- 업로드 파일용 volume 포함

### `frontend`

- `../sweetbook-frontend` 기준으로 build
- backend 준비 후 시작

## 문제 해결

### SweetBook 견적이 실패할 때

먼저 `.env`에 아래 값이 있는지 확인합니다.

```env
SWEETBOOK_ENV=sandbox
SWEETBOOK_API_KEY=your-sandbox-key
```

설정 후 다시 빌드합니다.

```powershell
docker compose up --build -d
```

### 코드 수정이 반영되지 않을 때

컨테이너 이미지는 빌드 시점의 파일을 사용하므로, 코드 수정 후에는 다시 빌드해야 합니다.

```powershell
docker compose up --build -d
```

### 전체 종료

```powershell
docker compose down
```
