# Project Foundation

이 문서는 SweetBook 프로젝트의 상위 기획 문서다.
개별 구현 task의 실행 계획이 아니라, 사용자와 상호작용하며 방향을 합의하는 기준 문서로 사용한다.

## Goal

그룹 이벤트 사진 업로드, 좋아요 투표, 앨범 선정, SweetBook 연동까지 이어지는 서비스의 전체 개발 방향을 정의하고, 이후 프론트엔드/백엔드/하네스 작업의 기준을 세운다.

## Product Flow

1. 사용자가 그룹을 만든다
2. 그룹 안에서 이벤트를 만든다
3. 그룹 구성원이 이벤트에 사진을 업로드한다
4. 그룹 구성원이 사진에 좋아요를 누른다
5. 좋아요 결과와 규칙에 따라 앨범 후보 사진이 선정된다
6. 선정 결과를 바탕으로 앨범 페이지 후보가 생성된다
7. 백엔드가 이를 SweetBook 입력 구조로 변환한다
8. 검토 후 책을 최종화하고 주문한다

## Frontend / Backend Ownership

### Frontend

- 그룹/이벤트 관리 화면
- 사진 업로드 UI
- 좋아요 투표 UI
- 앨범 후보 확인 UI
- 주문 진행 UI

### Backend

- 그룹/멤버 권한 관리
- 이벤트 CRUD
- 사진 업로드 메타데이터 관리
- 좋아요 집계
- 앨범 선정 규칙 실행
- SweetBook payload 생성
- SweetBook API 호출
- 주문 상태 저장
- 웹훅 수신 및 반영

## Domain Draft

- `User`
- `Group`
- `GroupMember`
- `Event`
- `Photo`
- `PhotoLike`
- `AlbumProject`
- `AlbumPageCandidate`
- `AlbumSelectionRule`
- `SweetBookBookJob`
- `SweetBookOrder`

## Recommended Repository Strategy

현재 권장 방향:

- 코드 리포는 프론트엔드와 백엔드를 분리 운영
- 현재 workspace는 공통 문서와 task 관리 공간으로 유지

예시:

```text
sweetbook/
  docs/
  tasks/
  sweetbook-backend/
  sweetbook-frontend/
```

이유:
- 프론트와 백엔드의 변경 성격이 다르다
- 백엔드에 도메인 로직과 외부 연동 복잡도가 집중된다
- 문서/기획/하네스는 상위 workspace에서 같이 관리하는 편이 낫다

## MVP Definition

### MVP 포함

- 그룹 생성
- 이벤트 생성
- 사진 업로드
- 좋아요 집계
- 상위 사진 선정
- 앨범 후보 생성
- SweetBook 책 생성/최종화
- 견적 조회
- 샌드박스 주문 생성

### MVP 제외

- 고급 편집 UI
- 복수 이벤트 병합 앨범
- 복잡한 추천 알고리즘
- 운영자용 고급 관리도구
- 실시간 알림

## Validation Strategy

### Harness 1

- `album-selection-harness`
- 입력: 이벤트, 사진, 좋아요 fixture
- 출력: 선택된 사진 목록, 페이지 후보 구조

### Harness 2

- `sweetbook-integration-harness`
- 입력: 내부 앨범 모델
- 출력: SweetBook payload, 샌드박스 실응답 검증

## Phase Plan

### Phase 1. Planning

- 상위 계획 정리
- 도메인 규칙 확정
- task 분해

### Phase 2. Backend Domain

- 그룹/이벤트/사진/좋아요 모델
- 앨범 선정 로직
- 테스트 기반 구현

### Phase 3. Harness

- 선정 규칙 하네스
- SweetBook 연동 하네스

### Phase 4. SweetBook Integration

- 책 생성
- 표지/내지 구성
- 최종화
- 견적
- 주문
- 웹훅

### Phase 5. Frontend MVP

- 그룹/이벤트/사진 업로드
- 좋아요 화면
- 앨범 후보 확인
- 주문 흐름

## Open Decisions

- 사진 선정 기준
  - 좋아요 상위 N장인지
  - 사용자별 최소 1장 보장인지
- 동점 처리 기준
  - 업로드 시간
  - 댓글/조회수
  - 랜덤 금지 여부
- 이벤트와 앨범의 관계
  - 이벤트 1개 = 앨범 1개인지
  - 여러 이벤트를 하나의 앨범으로 묶는지
- 페이지 구성 규칙
  - 이벤트당 고정 페이지 수인지
  - 사진 수에 따라 가변인지
- 운영자 수동 편집 허용 여부

## Immediate Next Tasks

1. 앨범 선정 규칙을 구체화한다
2. 백엔드 리포의 초기 아키텍처를 정한다
3. `album-selection-harness`의 RED 단계 테스트 시나리오를 만든다
4. 프론트엔드 MVP 화면 목록을 정의한다


codex resume 019d5a86-9d24-7443-a079-6737c93e918b