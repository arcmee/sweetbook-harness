# SweetBook API Notes

이 문서는 SweetBook API 문서를 읽고 서비스 구현에 필요한 내용을 누적 정리한 작업 메모다.
문서 원문을 그대로 옮기기보다, 구현 시 바로 참고할 수 있게 흐름, 제약, 스키마, 주의점 위주로 정리했다.

## 1. 문서 기준

- 개요: `v1.0 (2026-03-24)`
- 빠른 시작: `v1.0.1 (2026-04-02)`
- 전체 워크플로우: `v1.0.1 (2026-04-02)`
- Books API: `v1.0.1 (2026-04-02)`
- Orders API: `v1.0.1 (2026-04-02)`
- Templates API: 버전 표기 있음, 본문 확인 완료
- BookSpecs API: `v1.0.1 (2026-04-02)`
- Credits API: `v1.0.1 (2026-04-02)`
- Webhooks API: 버전 표기 있음, 본문 확인 완료
- Errors API: 본문 확인 완료
- 웹훅 연동 가이드: `v1.0 (2026-03-24)`
- 운영 전환 가이드: `v1.0 (2026-03-24)`

## 1.1 확인 상태

### 본문 확인 완료

- 개요
- 빠른 시작
- 전체 워크플로우
- Books API
- Orders API
- Templates API
- BookSpecs API
- Credits API
- Webhooks API
- Errors API
- 충전금 운영 가이드
- 주문 상태 흐름
- 운영 전환 가이드
- gallery
- column
- special-page-rules
- idempotency
- dynamic-layout
- template-engine
- element-grouping
- text-processing

### 본문 직접 확인 실패 또는 불완전

- `api/common`
- `api/webhook-events`
- `operations/troubleshooting`
- `concepts/base-layer`

이 문서의 `미확인` 또는 `추가 확인 필요` 항목은 위 네 페이지를 아직 직접 확인하지 못한 영향이 포함된다.

## 2. 기본 정보

- Live Base URL: `https://api.sweetbook.com/v1`
- Sandbox Base URL: `https://api-sandbox.sweetbook.com/v1`
- 인증 방식: `Authorization: Bearer {API_KEY}`
- 응답 형식: JSON
- API 버전: `v1`
- 통화: `KRW`

핵심 해석:
- Sandbox와 Live는 URL, API Key, 충전금, 웹훅 설정이 분리된다.
- 프론트엔드에서 직접 호출하는 구조가 아니라, 파트너 서버를 통해 호출하는 형태가 문서 전반의 기본 전제다.

### 2.1 실제 샌드박스 응답 확인

`2026-04-03` 기준 제공받은 샌드박스 키로 실제 호출 확인 완료:

- `GET /credits`
- `GET /book-specs`
- `GET /templates`

확인 결과:
- 인증 헤더 형식 `Authorization: Bearer {API_KEY}`가 실제 동작함
- 응답 루트는 문서대로 `success`, `message`, `data`
- `GET /templates`는 `data.templates`, `data.pagination` 구조
- `GET /book-specs`는 현재 샌드박스에서 배열 형태 `data: []`로 내려옴

주의:
- 일부 `bookSpec` 레코드는 값이 비어 있는 placeholder처럼 보인다.
- 샌드박스 응답은 문서 예시와 일부 차이가 있을 수 있으므로 구현 시 null-safe 처리가 필요하다.

## 3. 표준 응답 형식

### 성공 응답

```json
{
  "success": true,
  "message": "Success",
  "data": {}
}
```

### 실패 응답

```json
{
  "success": false,
  "message": "Error message",
  "data": null,
  "errors": ["세부 에러 메시지"]
}
```

구현 메모:
- HTTP 상태코드만 보지 말고 `success`도 함께 확인한다.
- 내부 로그에는 `message`, `errors`, HTTP status, request id 성격의 값을 함께 남기는 편이 좋다.

추가 확인:
- 일부 검증 에러 응답은 `errors` 대신 `fieldErrors[]`를 반환한다.

예시:

```json
{
  "success": false,
  "message": "Validation failed",
  "fieldErrors": [
    { "field": "email", "message": "Invalid email format" }
  ]
}
```

## 4. 전체 구현 흐름

문서상 기본 순서:

1. `GET /book-specs`
2. `GET /templates`
3. `POST /books`
4. `POST /books/{bookUid}/photos`
5. `POST /books/{bookUid}/cover`
6. `POST /books/{bookUid}/contents`
7. `POST /books/{bookUid}/finalization`
8. `POST /orders/estimate`
9. `POST /orders`
10. 웹훅 수신 또는 `GET /orders/{orderUid}` 폴링

핵심 키:
- `bookUid`: 책 생성 후 이후 단계 전부의 기준 키
- `orderUid`: 주문 상태 추적 기준 키
- `templateUid`: 템플릿 선택 및 바인딩 기준 키
- `bookSpecUid`: 판형 선택 기준 키

## 5. 빠른 시작에서 확인된 제약

- 예시 판형: `SQUAREBOOK_HC`
- 해당 예시 최소 페이지 수: `24페이지`
- 표지용 이미지 2장 필요
- 내지용 이미지 15장 이상 예시
- 지원 이미지 형식: `JPG`, `PNG`, `GIF`, `BMP`, `WebP`, `HEIC`
- `SVG` 미지원
- Sandbox 테스트 가격은 `100원 이하`
- Sandbox에서는 실제 인쇄/배송이 진행되지 않음
- Sandbox 주문 상태는 `PAID`에서 멈춤
- Sandbox 주문 생성 시 `order.paid` 웹훅 발생

해석:
- 최소 페이지 규칙은 판형별로 다를 가능성이 높다.
- 서비스에서 책 생성 전 `bookSpec` 기준 최소 페이지 수 검증을 선행하는 편이 안전하다.

## 6. 권장 서비스 구조

문서가 가정하는 구조는 다음과 같다.

`사용자 앱 -> 파트너 서버 -> SweetBook API`

따라서:
- API Key는 백엔드에만 둔다.
- 템플릿 UID, 파라미터 매핑, 주문 검증 로직도 백엔드에서 관리한다.
- 판형/템플릿 목록은 캐시 대상이다.

문서가 제시한 시나리오:

### 사용자 선택형

적합한 서비스:
- 앨범 앱
- 사진 편집 앱

특징:
- 사용자가 판형, 템플릿, 사진을 선택
- 서버는 책 생성, 파일 업로드, 주문 실행 담당

### 서버 자동 생성형

적합한 서비스:
- 일기장
- 알림장

특징:
- 서버가 내부 데이터로 책을 자동 생성
- 템플릿별 `templateUid`와 `parameters` 매핑을 서버에 미리 보유

현재 프로젝트 설계 시 결정해야 할 것:
- 사용자 편집형인지
- 서버 자동 생성형인지
- 둘을 혼합한 구조인지

## 7. 주요 엔드포인트 개요

### Books

- `POST /books`
- `GET /books`
- `POST /books/{bookUid}/photos`
- `POST /books/{bookUid}/cover`
- `POST /books/{bookUid}/contents`
- `POST /books/{bookUid}/finalization`

### Orders

- `POST /orders/estimate`
- `POST /orders`
- `GET /orders`
- `GET /orders/{orderUid}`
- `POST /orders/{orderUid}/cancel`

### Templates

- `GET /templates`
- `GET /templates/{templateUid}`
- `GET /template-categories`

### BookSpecs

- `GET /book-specs`
- `GET /book-specs/{bookSpecUid}`

### Credits

- `GET /credits`
- `GET /credits/transactions`
- `POST /credits/sandbox/charge`
- `POST /credits/sandbox/deduct`

### Webhooks

- `PUT /webhooks/config`
- `GET /webhooks/config`
- `POST /webhooks/test`

## 8. Books 구현 메모

### 8.0 실제 확인된 엔드포인트

- `GET /books`
- `POST /books`
- `POST /books/{bookUid}/cover`
- `POST /books/{bookUid}/photos`
- `GET /books/{bookUid}/photos`
- `DELETE /books/{bookUid}/photos/{fileName}`
- `POST /books/{bookUid}/contents`
- `DELETE /books/{bookUid}/contents`
- `DELETE /books/{bookUid}`
- `POST /books/{bookUid}/finalization`

### 8.1 책 생성

요청 예시:

```json
{
  "title": "나의 첫 포토북",
  "bookSpecUid": "SQUAREBOOK_HC"
}
```

실제 확인된 필드:
- `title`
- `bookSpecUid`
- `specProfileUid` 선택
- `externalRef` 선택

제약:
- `title`: 1~255자
- `externalRef`: 최대 100자

동작:
- 생성 직후 상태는 `draft`
- `Idempotency-Key` 헤더 공식 지원

응답:
- `data.bookUid`

실제 샌드박스 응답 확인:

```json
{
  "success": true,
  "message": "책 생성 완료",
  "data": {
    "bookUid": "bk_4HM74Ny3E30e"
  }
}
```

활용 메모:
- `externalRef`는 파트너 내부 주문/작업 ID 연결용으로 사실상 필수에 가깝다.
- 네트워크 재시도 가능성을 고려하면 `POST /books`에는 항상 멱등 키를 붙이는 편이 안전하다.

실제 확인된 점:
- 생성 응답은 매우 얇다. 초기에 `bookUid`만 반환된다고 가정하는 편이 안전하다.
- 생성 직후 상세 정보를 더 원하면 목록 API 등 다른 조회를 써야 한다.

### 8.2 표지 추가

요청 형식:
- `multipart/form-data`

실제 확인된 필드:
- `frontPhoto`
- `backPhoto`
- `templateUid`
- `parameters`

`parameters`는 multipart 내 JSON 문자열 형태로 전달되는 것으로 보인다.

예시:

```json
{
  "title": "나의 첫 포토북",
  "author": "홍길동"
}
```

추가 규칙:
- 텍스트 파라미터를 개별 form field로 보내는 방식은 더 이상 지원되지 않음
- 동적 파라미터는 반드시 `parameters` JSON 문자열로 전달해야 함
- 이미지 필드명은 템플릿 변수명과 정확히 일치해야 함
- 최종화(`status=2`) 또는 삭제(`status=9`)된 책에는 표지 추가 불가
- EXIF 정보는 표지 업로드에서는 항상 보존됨

이미지 제공 방식:
- multipart 파일 직접 업로드
- `parameters` JSON 안에 URL 문자열 전달
- Photo Upload API로 업로드한 서버 파일명 사용
- 혼합 방식 사용 시 `$upload` 플레이스홀더 사용 가능

응답:
- 신규 표지 생성 시 `201 Created`
- 기존 표지 업데이트 시 `200 OK`
- 응답 `data.result`는 `inserted` 또는 `updated`

실제 샌드박스 응답 확인:

```json
{
  "success": true,
  "message": "Cover created successfully",
  "data": {
    "result": "inserted"
  }
}
```

실제 확인된 요청 패턴:
- `multipart/form-data`
- `templateUid`
- `parameters`는 JSON string part
- `frontPhoto`, `backPhoto`는 file part

검증에 사용한 실제 템플릿:
- `templateUid=4MY2fokVjkeY`
- 필수 파라미터: `frontPhoto`, `dateRange`, `spineTitle`
- 선택 파라미터: `backPhoto`

### 8.3 파일 업로드 공통 제한

Books 문서에서 명시된 업로드 제한:
- 파일당 최대 `50MB`
- 책당 최대 사진 수 `200장`
- 지원 포맷: `JPEG`, `PNG`, `GIF`, `BMP`, `TIFF`, `WebP`
- 업로드 Rate Limit: `200 req/min`
- 일반 API Rate Limit: `300 req/min`

추가 규칙:
- magic byte 검증 수행
- URL 다운로드 제한: `50MB`, `10초 timeout`
- `preserveExif` 파라미터 지원 언급 있음

주의:
- 파일 업로드 제한 섹션에는 `TIFF`가 보이지만, 실제 `POST /photos` 지원 포맷 표에는 `TIFF`가 없고 비지원 목록에 포함된다.
- 구현 시에는 `POST /photos` 기준을 우선 적용하는 편이 안전하다.

### 8.4 사진 업로드

워크플로우 문서에 `POST /books/{bookUid}/photos`가 등장한다.

실제 확인된 요청:
- `multipart/form-data`
- 필드명: `file`

실제 확인된 의미:
- 업로드된 사진의 `fileName`을 `contents` API에서 사용할 수 있음

실제 확인된 응답 필드:
- `fileName`
- `originalName`
- `size`
- `mimeType`
- `uploadedAt`
- `isDuplicate`
- `hash`

지원 포맷:
- `jpg`, `jpeg`
- `png`
- `gif`
- `bmp`
- `webp`
- `heic`, `heif`

비지원:
- `SVG`, `TIFF`, `PDF`, `AI`, `PSD`

서버 처리:
- GIF -> PNG
- WebP -> PNG
- BMP -> JPG
- HEIC/HEIF -> JPG
- 긴 축 기준 원본 `4000px`, 썸네일 `800px`
- EXIF는 Orientation 적용 후 제거
- MD5 해시 기반 중복 체크 수행

추가 엔드포인트:
- `GET /books/{bookUid}/photos`: 업로드 사진 목록 조회
- `DELETE /books/{bookUid}/photos/{fileName}`: 업로드 사진 삭제

실제 샌드박스 초기 응답:

```json
{
  "success": true,
  "message": "Success",
  "data": {
    "photos": [],
    "totalCount": 0
  }
}
```

삭제 규칙:
- 최종화된 책(`status=2`)의 사진은 삭제 불가
- 삭제된 `fileName`을 `contents`에서 사용하면 에러

### 8.5 내지 추가

요청 형식:
- `multipart/form-data`

실제 확인된 필드:
- `templateUid`
- `parameters`
- 동적 이미지 필드들

쿼리 파라미터:
- `breakBefore`
  - `page`: 항상 새 페이지 시작
  - `column`: 여유 컬럼 우선 배치, 없으면 다음 페이지
  - `none`: 이전 콘텐츠 바로 다음에 연속 배치

검증 규칙:
- `required=true` 파라미터 누락 시 `400`
- `required=false` 또는 미정의 파라미터 누락 시 해당 요소 제거

지원 기능:
- 동적 텍스트 높이 계산
- 지능형 요소 배치
- Gallery 레이아웃
- 컬럼 레이아웃
- 필드명 기반 이미지 매칭

추가 규칙:
- 템플릿 변수명과 업로드 필드명이 정확히 일치해야 함
- Gallery는 같은 필드명으로 여러 파일 전송 가능
- URL 방식 사용 시 다운로드 제한은 `50MB`, `10초`
- 최종화(`status=2`) 또는 삭제(`status=9`)된 책에는 콘텐츠 추가 불가
- `SQUAREBOOK_HC`는 첫 내지 페이지가 오른쪽부터 시작

혼합 입력 예시:
- multipart 업로드 파일
- URL
- 미리 업로드된 서버 파일명
- `$upload` 플레이스홀더

응답:
- 신규 생성 시 `201 Created`
- 기존 콘텐츠 업데이트 시 `200 OK`
- 응답에 `result`, `breakBefore`, `pageCount` 포함

실제 샌드박스 응답 확인:

첫 번째 내지 추가:

```json
{
  "success": true,
  "message": "Content created successfully",
  "data": {
    "result": "inserted",
    "breakBefore": "page",
    "pageNum": 1,
    "pageSide": "right",
    "pageCount": 0
  }
}
```

두 번째 내지 추가:

```json
{
  "success": true,
  "message": "Content created successfully",
  "data": {
    "result": "inserted",
    "breakBefore": "page",
    "pageNum": 2,
    "pageSide": "left",
    "pageCount": 2
  }
}
```

세 번째 내지 추가:

```json
{
  "success": true,
  "message": "Content updated successfully",
  "data": {
    "result": "updated",
    "breakBefore": "page",
    "pageNum": 2,
    "pageSide": "right",
    "pageCount": 2
  }
}
```

실제 검증에 사용한 템플릿:
- `templateUid=4slyauW5rkUE`
- 이 템플릿은 `rowGallery`가 선택이고 텍스트 파라미터만으로도 호출 가능

실제 관찰:
- 첫 호출 직후 `pageNum=1`, `pageSide=right`, `pageCount=0`
- 두 번째 호출에서 `pageCount=2`로 증가
- 세 번째 호출은 새 페이지 추가가 아니라 `updated`로 처리되었다

구현 메모:
- `contents`는 호출 횟수만큼 항상 페이지가 늘어난다고 단정하면 안 된다.
- 템플릿/흐름 규칙에 따라 `inserted`와 `updated`가 갈릴 수 있다.
- 페이지 수 계산은 실제 응답의 `pageCount`와 이후 책 목록의 `pageCount`를 함께 보는 편이 안전하다.

### 8.6 내지 초기화

- `DELETE /books/{bookUid}/contents`

동작:
- 모든 내지 삭제
- 표지는 유지
- 개발/테스트 용도
- 복구 불가

응답 예시 필드:
- `deletedPages`
- `message`

### 8.7 책 삭제

- `DELETE /books/{bookUid}`

동작:
- 소프트 삭제
- 삭제 상태는 `9 (deleted)`

제약:
- `DRAFT` 상태만 삭제 가능
- 책 소유자만 삭제 가능

응답 예시:
- `bookUid`
- `status`

### 8.7.1 책 목록 조회 실응답

- `GET /books`

실제 샌드박스 응답 구조:
- `data.books`
- `data.pagination`

실제 확인된 `books[]` 필드:
- `bookUid`
- `accountUid`
- `title`
- `author`
- `status`
- `pageCount`
- `bookSpecUid`
- `specProfileUid`
- `creationType`
- `createdAt`
- `updatedAt`
- `sweetbookId`
- `sb`
- `externalRef`
- `isTest`
- `ebookServerId`
- `ebookBookId`
- `pdfStatus`
- `pdfPath`
- `pdfCreatedAt`
- `pdfRequestedAt`

실제 샌드박스 예시 값:
- `status="draft"`
- `pageCount=0`
- `creationType="EBOOK_SYNC"`
- `isTest=true`

실제 후속 확인:
- 내지 추가 후 목록 조회의 `pageCount`가 `2`로 갱신되는 것을 확인
- `updatedAt`도 내지 수정 시점으로 변경됨
- `contents` 응답의 `pageCount`가 먼저 증가하고, `GET /books` 목록의 `pageCount` 반영은 약간 지연될 수 있음

구현 메모:
- `status`는 문자열로 내려온다.
- `pdf*` 계열 필드는 초안 상태에서 `null`일 수 있다.
- `ebookServerId`, `ebookBookId`는 내부 연동용으로 보이며 서비스 핵심 도메인 모델에는 선택 필드로 두는 편이 안전하다.

### 8.7.2 단건 책 조회

실제 샌드박스 확인 결과:
- `GET /books/{bookUid}` 호출 시 `405 Method Not Allowed`

해석:
- 현재 기준으로 단건 조회 API는 없거나 다른 경로일 가능성이 높다.
- 구현은 `GET /books` 목록 조회를 우선 기준으로 잡는 편이 안전하다.

### 8.8 최종화

- `POST /books/{bookUid}/finalization`
- 페이지 수 검증이 수행됨
- 최종화 후 내용 변경 불가

실제 확인된 규칙:
- `DRAFT` 상태만 최종화 가능
- 책 소유자만 가능
- 판형의 페이지 수 규칙(최소/최대/증분) 만족 필요
- 멱등 처리 지원

응답 예시 필드:
- `result`
- `pageCount`
- `finalizedAt`

추가 규칙:
- 페이지 수에 따라 표지 책등(spine) 크기가 자동 조정됨

구현 메모:
- 최종화 전 단계에서 최소 페이지 수, 필수 표지/내지 유무, 업로드 실패 건수를 내부적으로 점검한 뒤 호출하는 편이 안전하다.

실제 샌드박스 검증:

내지 1회 추가 후:

```json
{
  "success": false,
  "message": "Bad Request",
  "data": null,
  "errors": ["최소 페이지 미달: 현재 0p, 최소 24p"]
}
```

내지 3회 처리 후:

```json
{
  "success": false,
  "message": "Bad Request",
  "data": null,
  "errors": ["최소 페이지 미달: 현재 2p, 최소 24p"]
}
```

의미:
- 최종화 에러 메시지에 실제 계산된 현재 페이지 수가 들어온다.
- 페이지 수 검증은 목록 조회 `pageCount`와 동일 기준으로 보인다.

최종화 성공 실응답:

```json
{
  "success": true,
  "message": "책 최종화 완료",
  "data": {
    "result": "페이지를 추가하지 않고 완료",
    "pageCount": 24,
    "finalizedAt": "2026-04-02T18:39:56.007Z"
  }
}
```

최종화 후 목록 조회 실응답:
- `status="finalized"`
- `pageCount=24`
- `updatedAt=finalizedAt`

실제 확인된 점:
- `finalization` 호출 직전 `contents` 응답은 `pageCount=24`였지만 목록 조회는 잠시 `20`으로 보였다.
- 최종화 후 다시 조회하면 목록도 `24`로 정합성이 맞춰진다.

구현 메모:
- 직후 조회 값이 잠시 늦게 반영될 수 있으므로, 즉시 강한 정합성을 가정하지 않는 편이 안전하다.
- 최종화 직전 페이지 수 검증은 최근 `contents` 응답 기준을 함께 참고하는 편이 좋다.

## 9. 템플릿/레이아웃 개념 정리

SweetBook의 핵심은 템플릿 기반 렌더링이다. 서비스는 단순 파일 업로드만 하는 것이 아니라, 템플릿별 바인딩 규칙에 맞춰 `parameters`와 파일을 매핑해야 한다.

### 9.1 Template Engine

문서에서 확인한 핵심:
- 템플릿은 파라미터 정의와 레이아웃 요소 정의를 가진다.
- `parameters.definitions`로 입력값 스키마를 선언하는 구조가 보인다.
- 텍스트, 파일 배열, 날짜 등 여러 타입을 바인딩한다.

서비스에 필요한 대응:
- 템플릿 상세 조회 결과를 내부적으로 파싱 가능한 형태로 저장
- 템플릿별 필수 파라미터 검증기 작성
- UI가 있다면 템플릿 정의를 기반으로 입력 폼을 자동 생성할 여지도 있음

### 9.2 Gallery Templates

문서에서 확인한 갤러리 타입:
- `collageGallery`
- `rowGallery`

특징:
- `collageGallery`: 1~9장
- `rowGallery`: 최소 1장, 사진 수 제한 없음
- 동일 필드명으로 여러 파일을 반복 전송

예시 필드명:
- `collagePhotos`
- `rowPhotos`

자동 분할:
- 사진 수/레이아웃 크기에 따라 서버가 여러 그룹 또는 여러 페이지로 분할
- 결과 그룹명 예시:
  - `collageGallery_tag_part0`
  - `rowGallery_tag_page0`

선택 기준:
- 적은 수 사진을 미적으로 배치: `collageGallery`
- 많은 사진을 체계적으로 나열: `rowGallery`

### 9.3 Column Templates

문서에서 확인한 컬럼 규칙:
- 1컬럼, 2컬럼, 3컬럼 템플릿 예시 존재
- 같은 컬럼 수의 템플릿끼리는 같은 페이지 면에 배치 가능
- 다른 컬럼 수의 템플릿은 같은 면에 배치 불가

예시:
- 2컬럼 템플릿 적용 시 콘텐츠 영역이 좌우 슬롯으로 나뉨
- 3컬럼에서는 세 개 슬롯 생성

서비스 영향:
- 템플릿을 섞어 배치할 때 컬럼 수 호환성 때문에 예상 페이지 수가 달라질 수 있다.
- 편집 UI를 만든다면 "같은 면에 들어갈 수 있는 조합" 규칙을 알아야 한다.

### 9.4 Special Page Rules

문서에서 확인한 `template_kind`:
- `cover`
- `content`
- `divider`
- `publish`

규칙:
- `content`는 일반 흐름에 따라 이어짐
- `divider`, `publish`는 독립 특수 페이지
- `divider/publish`에는 `breakBefore=page`만 허용되고 `none`, `column`은 에러
- `content`에서는 `pageSide` 사용 불가라는 설명이 문서에 있음

서비스 영향:
- 간지/발행면 기능을 쓸 경우 일반 내지와 별도 모델로 다루는 편이 좋다.

### 9.5 Text Processing

문서 전체 예시를 통해 확인된 텍스트 특성:
- 텍스트 내 줄바꿈 `\n`을 허용
- 날짜 문자열 포맷을 템플릿 파라미터로 넘기는 예시 다수 존재

추정:
- 템플릿 엔진 차원에서 텍스트 줄바꿈, 정렬, 포맷터가 작동

서비스 영향:
- 서버 또는 프론트에서 줄바꿈 정규화가 필요할 수 있다.
- 사용자 입력을 그대로 넘길지, sanitize 후 넘길지 기준을 정해야 한다.

## 10. Orders 구현 메모

### 10.0 주문 상태 코드

- `20 PAID`
- `25 PDF_READY`
- `30 CONFIRMED`
- `40 IN_PRODUCTION`
- `45 COMPLETED`
- `50 PRODUCTION_COMPLETE`
- `60 SHIPPED`
- `70 DELIVERED`
- `80 CANCELLED`
- `81 CANCELLED_REFUND`
- `90 ERROR`

주요 상태 전이:
- `PAID -> PDF_READY`
- `PDF_READY -> CONFIRMED`
- `CONFIRMED -> IN_PRODUCTION`
- `IN_PRODUCTION -> COMPLETED`
- `COMPLETED -> PRODUCTION_COMPLETE`
- `PRODUCTION_COMPLETE -> SHIPPED`
- `SHIPPED -> DELIVERED`
- `PAID -> CANCELLED_REFUND`
- `PDF_READY -> CANCELLED_REFUND`

### 10.1 견적 조회

요청 예시:

```json
{
  "items": [
    { "bookUid": "{bookUid}", "quantity": 1 }
  ]
}
```

용도:
- 주문 전 예상 비용 확인
- 잔액 부족 여부 선확인

실제 샌드박스 응답 예시:

```json
{
  "success": true,
  "message": "성공",
  "data": {
    "items": [
      {
        "bookUid": "bk_4HM74Ny3E30e",
        "bookSpecUid": "SQUAREBOOK_HC",
        "pageCount": 24,
        "quantity": 1,
        "unitPrice": 100.00,
        "itemAmount": 100.00,
        "packagingFee": 0
      }
    ],
    "productAmount": 100.00,
    "shippingFee": 3000,
    "packagingFee": 0,
    "totalAmount": 3100.00,
    "paidCreditAmount": 3410,
    "creditBalance": 1000.00,
    "creditSufficient": false,
    "currency": "KRW"
  }
}
```

실제 확인된 점:
- `totalAmount`와 `paidCreditAmount`가 다르다.
- 샌드박스 주문 생성 시 실제 차감 기준은 `paidCreditAmount`로 보인다.
- `creditSufficient`로 잔액 부족 여부를 바로 판단할 수 있다.

### 10.2 주문 생성

요청 예시:

```json
{
  "items": [
    { "bookUid": "{bookUid}", "quantity": 1 }
  ],
  "shipping": {
    "recipientName": "홍길동",
    "recipientPhone": "010-1234-5678",
    "postalCode": "06236",
    "address1": "서울특별시 강남구 테헤란로 123",
    "address2": "4층"
  }
}
```

현재까지 확인된 필드:
- `items[].bookUid`
- `items[].quantity`
- `shipping.recipientName`
- `shipping.recipientPhone`
- `shipping.postalCode`
- `shipping.address1`
- `shipping.address2`
- `shipping.memo`
- `externalRef`

제약:
- `items` 최소 1개
- `items[].quantity` 범위 `1~100`
- `items[].bookUid`는 `FINALIZED` 상태여야 함
- `shipping.recipientName` 최대 100자
- `shipping.recipientPhone` 최대 20자
- `shipping.postalCode` 최대 10자
- `shipping.address1`, `shipping.address2` 최대 200자
- `shipping.memo` 최대 200자
- `externalRef` 최대 100자

주문 생성 시 동작:
- 충전금 즉시 차감
- `order.paid` 이벤트 발생

처리 로직:
1. `bookUid` 유효성 검증
2. 파트너 소유 여부 검증
3. `FINALIZED` 상태 검증
4. `bookSpecUid` 기준 가격 계산
5. 배송비 `3,500원` + 포장비 `500원/수량` + 상품금액 합산
6. 충전금 확인 및 차감
7. 주문/항목 레코드 생성

중요:
- `POST /orders`는 `Idempotency-Key`를 반드시 포함하라고 문서에 명시되어 있다.

실제 샌드박스 잔액 부족 응답:

```json
{
  "success": false,
  "message": "Insufficient Credit",
  "data": {
    "required": 3410,
    "balance": 1000.00,
    "currency": "KRW"
  },
  "errors": ["잔액이 부족합니다. 필요: 3410, 잔액: 1000.00"]
}
```

실제 샌드박스 주문 생성 성공 응답:

```json
{
  "success": true,
  "message": "주문이 생성되었습니다",
  "data": {
    "orderUid": "or_245QqefA1vDP",
    "orderStatus": 20,
    "orderStatusDisplay": "결제완료",
    "paidCreditAmount": 3410,
    "creditBalanceAfter": 2590.00,
    "items": [
      {
        "itemUid": "oi_4udZWtOJO48I",
        "bookUid": "bk_4HM74Ny3E30e",
        "pageCount": 24,
        "itemStatus": 20,
        "itemStatusDisplay": "결제완료"
      }
    ]
  }
}
```

실제 확인된 점:
- 생성 응답은 주문 상세에 가까운 두꺼운 payload를 반환한다.
- `orderStatus`, `itemStatus`는 숫자 코드로 내려온다.
- `recipientName`, `address1`, `address2`는 응답에서 마스킹 또는 인코딩 이슈처럼 보이는 값으로 보일 수 있다.
- `shipping.memo` 요청 필드는 실제 응답에서 `shippingMemo`로 내려온다.

### 10.3 주문 취소

문서상 존재:
- `POST /orders/{orderUid}/cancel`

확인된 동작:
- 주문 취소 시 결제 금액이 충전금으로 즉시 환불
- 웹훅 `order.cancelled` 발생
- 취소 후 상태는 `CANCELLED_REFUND`
- 요청 바디에 `cancelReason` 전달 가능

취소 가능 조건:
- `PAID` 또는 `PDF_READY`
- `NORMAL` 유형 주문만 가능
- `CONFIRMED` 이후 취소 불가

응답:
- `200 OK`
- 변경된 주문 상세 정보 반환

### 10.4 주문 상태 흐름

문서상 상태 전이:

1. `PAID (20)`
2. `PDF_READY (25)`
3. `CONFIRMED (30)`
4. `IN_PRODUCTION (40)`
5. `PRODUCTION_COMPLETE (50)`
6. `SHIPPED (60)`
7. `DELIVERED (70)`

Sandbox 차이:
- `PAID`에서 멈춤

구현 메모:
- 상태 코드를 enum으로 직접 정의하고, 웹훅과 조회 API 둘 다 같은 상태 머신으로 처리하는 게 좋다.

### 10.5 주문 목록 조회

- `GET /orders`

쿼리 파라미터:
- `limit` 기본 `20`, 최대 `100`
- `offset` 기본 `0`
- `status`
- `from` ISO 8601
- `to` ISO 8601

응답 예시 필드:
- `total`
- `limit`
- `offset`
- `hasNext`
- `items[]`
  - `orderUid`
  - `orderStatus`
  - `orderStatusDisplay`
  - `totalAmount`
  - `orderedAt`

실제 샌드박스 응답 구조:
- `data.orders`
- `data.pagination`

실제 확인된 `orders[]` 필드:
- `orderUid`
- `accountUid`
- `accountName`
- `accountOrganizationName`
- `orderType`
- `externalRef`
- `orderStatus`
- `orderStatusDisplay`
- `totalAmount`
- `paidCreditAmount`
- `paymentMethod`
- `itemCount`
- `recipientName`
- `isTest`
- `orderedAt`
- `createdAt`

### 10.6 주문 상세 조회

- `GET /orders/{orderUid}`

응답 예시 필드:
- `orderUid`
- `orderType`
- `orderStatus`
- `orderStatusDisplay`
- `externalRef`
- `totalProductAmount`
- `totalShippingFee`
- `totalPackagingFee`
- `totalAmount`
- `paidCreditAmount`
- `recipientName`
- `recipientPhone`
- `postalCode`
- `address1`
- `address2`
- `orderedAt`
- `items[]`
  - `itemUid`
  - `bookUid`
  - `bookTitle`
  - `quantity`
  - `unitPrice`
  - `itemAmount`
  - `itemStatus`
  - `itemStatusDisplay`

실제 샌드박스 응답 확인:
- `GET /orders/or_245QqefA1vDP` 성공
- 생성 응답과 거의 같은 수준의 상세 payload 반환

실제 확인된 추가 필드:
- `accountUid`
- `accountName`
- `accountOrganizationName`
- `originalOrderUid`
- `isTest`
- `paymentMethod`
- `batchUid`
- `creditBalanceAfter`
- `endUserAmount`
- `endUserShippingFee`
- `endUserDiscount`
- `endUserPaidAmount`
- `externalUserId`
- `trackingNumber`
- `trackingCarrier`
- `cancelReason`
- `refundAmount`
- `printDay`
- `memo`
- `paidAt`
- `confirmedAt`
- `cancelledAt`
- `shippedAt`
- `deliveredAt`
- `createdAt`

실제 확인된 `items[]` 추가 필드:
- `bookSpecUid`
- `bookSpecName`
- `pageCount`
- `endUserUnitPrice`
- `endUserItemAmount`
- `printIndex`
- `completedDay`
- `completedIndex`
- `trackingNumber`
- `trackingCarrier`
- `shippedAt`
- `originalItemUid`
- `createdAt`
- `ebookServerId`
- `ebookBookId`

### 10.7 배송지 변경

- `PATCH /orders/{orderUid}/shipping`

가능 상태:
- `PAID`
- `PDF_READY`
- `CONFIRMED`

불가 시점:
- `SHIPPED` 이후

부분 업데이트 지원 필드:
- `recipientName`
- `recipientPhone`
- `postalCode`
- `address1`
- `address2`
- `shippingMemo`

## 11. Credits 구현 메모

### 11.1 잔액 조회

- `GET /credits`

응답 필드:
- `balance`
- `currency`
- `env`

`env` 값:
- `"test"`
- `"live"`

실제 샌드박스 응답 예시:

```json
{
  "success": true,
  "message": "성공",
  "data": {
    "accountUid": "u_adb13baf12554c1ba0996b90af",
    "balance": 1000.00,
    "currency": "KRW",
    "createdAt": "2026-04-02T18:15:42.000Z",
    "updatedAt": "2026-04-02T18:15:42.000Z",
    "env": "test"
  }
}
```

### 11.2 거래 내역 조회

- `GET /credits/transactions`

쿼리 파라미터:
- `limit` 기본 `20`, 최대 `100`
- `offset` 기본 `0`
- `createdFrom` ISO 8601
- `createdTo` ISO 8601

응답 필드:
- `transactionUid`
- `type`
- `amount`
- `balanceAfter`
- `description`
- `createdAt`

거래 유형 예시:
- `charge`
- `deduct`
- `refund`

실제 샌드박스 응답 구조:

```json
{
  "success": true,
  "message": "성공",
  "data": {
    "transactions": [
      {
        "transactionId": 102,
        "accountUid": "u_adb13baf12554c1ba0996b90af",
        "reasonCode": 9,
        "reasonDisplay": "샌드박스 충전",
        "direction": "+",
        "currency": "KRW",
        "amount": 1000.00,
        "balanceAfter": 1000.00,
        "memo": "샌드박스 크레딧 충전",
        "createdAt": "2026-04-02T18:15:42.000Z",
        "isTest": true
      }
    ]
  }
}
```

실제 확인된 점:
- 현재 샌드박스 응답은 `transactionUid`가 아니라 `transactionId`
- 문서 요약 때 적었던 `type`, `description` 대신 실제 응답은 `reasonCode`, `reasonDisplay`, `memo`
- `direction`, `isTest` 필드 존재
- 현재 응답에는 pagination 정보가 없다

구현 메모:
- Credits transactions는 문서 예시보다 실제 응답을 우선 기준으로 DTO를 잡는 편이 안전하다.
- 운영/샌드박스 간 차이 가능성을 고려해 유연한 매핑이 필요하다.

### 11.3 Sandbox 테스트용 충전/차감

- `POST /credits/sandbox/charge`
- `POST /credits/sandbox/deduct`

요청 바디:
- `amount`
- `description`

중요:
- Sandbox에서만 사용 가능
- `Idempotency-Key` 헤더 지원

실제 샌드박스 충전 응답 예시:

```json
{
  "success": true,
  "message": "샌드박스 크레딧 충전 성공",
  "data": {
    "accountUid": "u_adb13baf12554c1ba0996b90af",
    "balance": 1000,
    "currency": "KRW",
    "createdAt": "2026-04-02T18:15:41.777Z",
    "updatedAt": "2026-04-02T18:15:41.779Z",
    "env": "test"
  }
}
```

구현 메모:
- 테스트 자동화 시 잔액 조작용으로 유용하다.
- 운영 코드에는 절대 연결되지 않도록 환경별 방어가 필요하다.

### 11.4 잔액 부족 에러

`402 Payment Required` 예시 응답:

```json
{
  "success": false,
  "message": "Insufficient Credit",
  "data": {
    "required": 64400.00,
    "balance": 10000.00,
    "currency": "KRW"
  },
  "errors": ["잔액이 부족합니다. 필요: 64400.00, 잔액: 10000.00"]
}
```

서비스 대응:
- 주문 전에 `estimate`와 `credits` 조회를 함께 사용
- 부족액을 사용자에게 안내할 수 있는 별도 상태 정의

## 12. 웹훅 구현 메모

### 12.1 설정

웹훅 등록:
- `PUT /webhooks/config`

요청 필드:
- `webhookUrl`
- `events[]`
- `description`

예시 이벤트:
- `order.paid`
- `order.confirmed`
- `order.status_changed`
- `order.shipped`
- `order.cancelled`

설정 조회:
- `GET /webhooks/config`

추가 규칙:
- 최초 등록 시 `secretKey` 자동 생성
- 최초 등록 시에만 `secretKey` 전체 값 반환
- 이후 조회에서는 앞 8자만 노출
- 수정 시 기존 `secretKey` 유지, 수정 응답에는 전체 `secretKey` 포함
- `DELETE /webhooks/config`로 비활성화 가능
- 해제는 소프트 삭제이며 전송 이력은 유지

테스트 이벤트:
- `POST /webhooks/test`

테스트 규칙:
- 샘플 데이터 전송
- 실제 주문 데이터는 포함되지 않음
- 재시도되지 않음
- 실패 시 `responseStatus`, `responseBody`로 진단

### 12.2 서명 검증

문서에서 확인된 헤더:
- `X-Webhook-Event`
- `X-Webhook-Delivery`
- `X-Webhook-Timestamp`
- `X-Webhook-Signature`

서명 방식:
- `{timestamp}.{payload}` 문자열을 시크릿 키로 HMAC-SHA256 해싱
- 헤더 서명 형식은 `sha256=...`

구현 메모:
- raw body 기준 검증이 필요하다.
- JSON 파싱 전에 먼저 서명 검증해야 한다.
- Express라면 `express.raw({ type: 'application/json' })` 패턴이 적절하다.

### 12.3 공통 페이로드 구조

```json
{
  "event_uid": "evt_abc123def456",
  "event_type": "order.paid",
  "created_at": "2025-10-21T03:00:00Z",
  "data": {}
}
```

공통 필드:
- `event_uid`
- `event_type`
- `created_at`
- `data`

추가 필드:
- Sandbox/Live 공통으로 웹훅이 발생하며, `isTest` 필드로 Sandbox 이벤트 여부 구분 가능하다고 Webhooks API 문서에 명시됨

### 12.4 이벤트별 요약

`order.paid`
- 주문 생성 완료
- 충전금 차감 완료
- `order_uid`, `order_status`, `total_amount`, `item_count`, `ordered_at`

`order.confirmed`
- 제작 확정
- `print_day`, `confirmed_at`

`order.status_changed`
- 상태 전이 통합 이벤트
- `previous_status`, `new_status`, `changed_at`

`order.shipped`
- 발송 완료
- `tracking_number`, `tracking_carrier`, `shipped_at`

`order.cancelled`
- 주문 취소 및 환불
- `cancel_reason`, `refund_amount`, `cancelled_at`

### 12.5 재시도 정책

- 1차 재시도: 1분 후
- 2차 재시도: 5분 후
- 3차 재시도: 30분 후
- 이후 중단

폴링 대안:
- `GET /orders/{orderUid}` 사용 가능
- 권장 간격 예시: 5~10분

## 13. 멱등성 정리

문서와 개별 API 예시에서 확인된 내용:
- 적어도 Credits Sandbox charge/deduct는 `Idempotency-Key` 지원
- `POST /books` 지원
- `POST /orders` 지원
- 멱등성 문서에는 재시도 시 같은 키를 유지하라는 방향의 설명이 있음
- 5xx는 재시도, 일반 4xx는 재시도하지 않는 예시가 있음

서비스 원칙 제안:
- 외부 결제/차감/주문 생성 계열 요청에는 내부 멱등 키를 반드시 부여
- 네트워크 실패, 타임아웃, 5xx에서만 자동 재시도
- 사용자 중복 클릭 방지도 같은 키 전략으로 통일

후속 확인 필요:
- `POST /books/{bookUid}/finalization`
- `POST /webhooks/config`

위 두 엔드포인트가 공식적으로 `Idempotency-Key`를 지원하는지는 추가 확인이 필요하다.

## 14. Templates 구현 메모

### 14.1 기본 분류

- `templateKind=cover`
- `templateKind=content`

표지 템플릿은 `POST /books/{bookUid}/cover`에만 사용한다.
내지 템플릿은 `POST /books/{bookUid}/contents`에만 사용한다.

### 14.2 카테고리

- `diary`
- `notice`
- `album`
- `yearbook`
- `wedding`
- `baby`
- `travel`
- `etc`

### 14.3 목록 조회

- `GET /templates`

현재까지 확인된 응답 예시 필드:
- `templateUid`
- `templateName`
- `templateKind`
- `category`
- `theme`
- `bookSpecUid`
- `isPublic`
- `status`
- `thumbnails.layout`
- `createdAt`
- `updatedAt`
- `pagination.total`
- `pagination.limit`
- `pagination.offset`
- `pagination.hasNext`

실제 샌드박스 응답 구조:

```json
{
  "success": true,
  "message": "성공",
  "data": {
    "templates": [],
    "pagination": {
      "total": 148,
      "limit": 3,
      "offset": 0,
      "hasNext": true
    }
  }
}
```

실제 확인된 템플릿 목록 필드:
- `templateUid`
- `accountUid`
- `templateName`
- `description`
- `templateKind`
- `category`
- `theme`
- `bookSpecUid`
- `bookSpecName`
- `specProfileUid`
- `isPublic`
- `status`
- `createdAt`
- `updatedAt`
- `thumbnails.layout` 선택적 포함

해석:
- 목록 응답은 배열 루트가 아니라 `data.templates` 래핑 구조다.
- `category`, `theme`, `description`, `specProfileUid`, `thumbnails`는 null 가능성이 있다.

실제 필터 동작 확인:
- `GET /templates?bookSpecUid=SQUAREBOOK_HC&limit=5` 호출 성공
- 반환된 5개 템플릿 모두 `bookSpecUid="SQUAREBOOK_HC"`
- `templateKind` 값으로 `content`, `publish`가 실제 목록에 함께 존재함

구현 메모:
- `bookSpecUid` 필터는 실제 동작한다.
- `templateKind`는 `cover`, `content`뿐 아니라 `publish`도 고려해야 한다.

### 14.4 상세 조회

- `GET /templates/{templateUid}`

확인된 상세 응답 구성:
- `parameters`
- `layout`
- `layoutRules`
- `baseLayer`
- `thumbnails`

의미:
- `parameters`: 텍스트/이미지/갤러리 바인딩 정보
- `layout`: 요소 배치 및 크기
- `layoutRules`: 여백, 흐름, 컬럼 규칙
- `baseLayer`: 홀수/짝수 페이지 배경 요소
- `thumbnails`: `layout`, `baseLayerOdd`, `baseLayerEven`

실제 샌드박스 응답 확인:
- `GET /templates/75HruEK3EnG5` 성공 확인
- 응답 루트는 `data` 객체
- 표지 템플릿 상세에는 `templateUid`, `accountUid`, `templateName`, `description`, `templateKind`, `category`, `theme`, `bookSpecUid`, `specProfileUid`, `isPublic`, `status`, `parameters`, `layout`, `layoutRules`, `createdAt`, `updatedAt`, `thumbnails`가 포함됨
- 실제 응답에는 `baseLayer`가 없는 템플릿도 존재함

구현 메모:
- `baseLayer`는 항상 있다고 가정하면 안 된다.
- `thumbnails`도 부분 필드만 올 수 있다.

### 14.5 파라미터 바인딩

확인된 바인딩 예시:
- `text`: 문자열
- `file`: 이미지 URL 또는 업로드 파일
- `rowGallery`: 사진 파일명 배열

중요:
- 갤러리 바인딩에 사용할 사진은 먼저 `POST /books/{bookUid}/photos`로 업로드하고, 반환된 `fileName`을 사용한다.

실제 샌드박스 응답의 `parameters.definitions` 예시:

```json
{
  "childName": {
    "binding": "text",
    "type": "string",
    "required": true,
    "description": "아이 이름"
  },
  "coverPhoto": {
    "binding": "file",
    "type": "string",
    "required": true,
    "description": "표지 사진"
  }
}
```

실제 확인된 점:
- `definitions`는 객체 맵 구조다.
- 각 항목에 `binding`, `type`, `required`, `description`이 포함된다.
- 실제 API 응답에는 `required`가 존재한다.

주의:
- 개념 문서에서 최신 스펙 설명과 실제 응답이 다를 수 있으므로, 구현은 실제 API 응답을 우선 기준으로 한다.

### 14.6 레이아웃 상세 구조

실제 샌드박스 템플릿 상세에서 확인된 `layout` 필드:
- `backgroundColor`
- `elements[]`

`elements[]` 예시 타입:
- `rectangle`
- `text`
- `graphic`
- `photo`

요소 공통 성격:
- `element_id`
- `type`
- `position.x`
- `position.y`
- `width`
- `height`

텍스트 요소 예시 속성:
- `text`
- `fontFamily`
- `fontSize`
- `textBold`
- `textBrush`
- `backgroundColor`
- `textAlignment`
- `verticalAlignment`
- `textOffset`
- `isDynamic`
- `splittable`

사진 요소 예시 속성:
- `fileName`
- `fit`
- `borderBrush`
- `borderThickness`
- `cornerRadius`

그래픽 요소 예시 속성:
- `imageSource`
- `opacity`
- `graphicType`

직접 확인된 바인딩 문법:
- 텍스트/파일 치환은 `$$paramName$$` 형식 사용

## 15. BookSpecs 구현 메모

### 15.1 판형 예시

`PHOTOBOOK_A4_SC`
- 210 x 297 mm
- Softcover
- PUR
- 24 ~ 130 페이지
- 2페이지 단위 증가

`PHOTOBOOK_A5_SC`
- 148 x 210 mm
- Softcover
- PUR
- 50 ~ 200 페이지
- 2페이지 단위 증가

`SQUAREBOOK_HC`
- 243 x 248 mm
- Hardcover
- PUR
- 24 ~ 130 페이지
- 2페이지 단위 증가

### 15.2 조회

- `GET /book-specs`
- `GET /book-specs/{bookSpecUid}`

문서상 포함 정보:
- 가격 정보
- 페이지 제한(최소/최대/단위)
- 크기 정보

실제 샌드박스 응답 확인 결과:
- `GET /book-specs?limit=3` 호출에서도 실제로는 6개가 반환되었다.
- 따라서 `limit` 쿼리 파라미터 동작이 문서와 다를 수 있다.
- 응답 루트는 `data` 배열 형태였다.

실제 확인된 필드:
- `bookSpecUid`
- `bookSpecId`
- `ebookProductId`
- `name`
- `innerTrimWidthMm`
- `innerTrimHeightMm`
- `pageMin`
- `pageMax`
- `pageDefault`
- `pageIncrement`
- `coverType`
- `bindingType`
- `bindingEdge`
- `priceCurrency`
- `priceBase`
- `pricePerIncrement`
- `sandboxPriceBase`
- `sandboxPricePerIncrement`
- `paper`
- `production`
- `bleed`
- `pdfSize`
- `layoutSize`
- `spineRules`
- `visibility`

실제 확인된 판형 값:
- `PHOTOBOOK_A4_SC`
  - `sandboxPriceBase=100`
  - `sandboxPricePerIncrement=10`
  - `pdfSize.inner=216x303`
  - `pdfSize.cover=455x303`
- `PHOTOBOOK_A5_SC`
  - `sandboxPriceBase=100`
  - `sandboxPricePerIncrement=10`
- `SQUAREBOOK_HC`
  - `sandboxPriceBase=100`
  - `sandboxPricePerIncrement=10`
  - `spineRules.rules[]` 구조 사용

해석:
- 책등 규칙은 판형마다 구조가 다르다.
  - 소프트커버: `baseWidth`, `widthPerPage`
  - 하드커버: `rules[]`
- 코드에서 `spineRules`를 단일 구조로 가정하면 안 된다.

### 15.3 상세 조회 실응답

`GET /book-specs/SQUAREBOOK_HC` 실응답 확인 결과:
- 응답 루트는 `data` 객체
- 목록 API에 보이던 필드와 동일 계열 구조를 상세에서도 사용
- `pdfSize`는 `null`일 수 있음
- `layoutSize`는 존재
- `spineRules.rules[]` 각 항목에 `minPage`, `maxPage`, `spineThickness`, `spineWidth` 포함

구현 메모:
- `pdfSize`, `bleed`, `paper`, `production`은 null 가능성을 열어둔다.
- 가격 계산은 현재 실응답 기준으로 `sandboxPriceBase`, `sandboxPricePerIncrement`를 우선 사용 가능하다.

추가 규칙:
- `accountUid`를 전달하지 않으면 인증된 계정 UID 자동 적용
- 계정별 커스텀 가격이 자동 반영될 수 있음
- 타인의 `accountUid` 전달 시 `403 Forbidden`

## 16. 공통 에러 메모

확인된 공통 HTTP 상태:
- `200`
- `201`
- `400`
- `401`
- `402`
- `403`
- `404`
- `409`
- `429`
- `500`

추가 규칙:
- `409`: 멱등성 키 충돌 또는 중복 요청
- `429`: `Retry-After` 헤더 확인
- `500`: 재시도 권장

검증 에러 특징:
- 일부 API는 `errors[]` 대신 `fieldErrors[]` 반환

## 17. 개념 문서 구현 포인트

### 17.1 Dynamic Layout

- Contents API는 페이지, 컬럼, 좌/우 면을 자동 관리
- PUR 제본은 첫 내지(`pageNum=1`)가 오른쪽 시작
- 기타 제본은 왼쪽 시작
- 오른쪽 면의 X 좌표는 `pageWidth` 오프셋 추가
- `pageMargin`이 정의되면 값이 0이어도 신규 레이아웃이 적용
- 1컬럼 템플릿에서는 `columnGap` 무시

### 17.2 Template Engine

- 최신 스펙 기준 `required` 프로퍼티 제거
- `parameters`에 정의된 항목은 기본적으로 필수
- `visible` 바인딩 지원
- `shiftUpOnHide` 지원
- 기본 `binding`은 `text`, `file`
- 타입은 `string`, `int`, `boolean`, `double` 명확화

`layoutRules` 주요 필드:
- `margin.pageMargin`
- `margin.mirrorEvenPages`
- `flow.columns`
- `flow.columnGap`
- `flow.itemSpacing.size`
- `shiftUpOnHide`
- `lanes`

주의:
- 실제 템플릿 상세 응답에는 `required: true`가 포함되어 있다.
- 따라서 개념 문서 설명보다 실제 API 응답을 우선 기준으로 구현한다.

### 17.3 Element Grouping

- 같은 `groupName`이라도 중간에 다른 그룹 요소가 끼면 별도 그룹으로 처리
- 요소는 Y 좌표 순으로 정렬되어 처리
- Y 순서를 잘못 주면 그룹이 의도와 다르게 분리될 수 있음

### 17.4 Text Processing

문서상 확인된 유틸:
- `GetDispartText()`: 높이 제한 내 텍스트 분할
- `BadTextRemover()`: HTML 엔티티 및 특수문자 제거

개발/디버깅용 API:
- `POST /books/{bookUid}/text-height`
- 응답에 `pageNumber`, `calculatedHeights` 포함

## 18. 운영 전환 가이드 요약

Live 전환 사전 조건:
- 사업 협의 완료
- Business 계정 전환 완료

전환 체크리스트:
- Live API Key 발급
- Base URL 변경
- 실제 충전금 충전
- 웹훅 URL 운영 주소로 변경
- IP 화이트리스트 설정 권장
- 에러 핸들링/재시도 로직 점검
- 충전금 잔액 모니터링 설정
- 첫 Live 주문 테스트

문서상 명시:
- Sandbox와 Live는 API 인터페이스가 동일
- 코드 변경은 실질적으로 `BASE_URL`, `API_KEY` 두 가지가 핵심

운영 SLA:
- 제작: 영업일 기준 3~4일
- 배송: 1~2일
- 택배사 예시: 한진택배

## 19. 트러블슈팅/에러 메모

문서에서 직접 확인된 자주 겪는 문제:
- `401 Unauthorized`
- `402 Payment Required`
- 최소 페이지 수 부족
- `Template not found`
- 이미지 업로드 실패

주요 원인:
- 잘못된 API Key
- Sandbox Key로 Live URL 호출
- 충전금 부족
- 잘못된 `templateUid`
- 미지원 이미지 형식

서비스 레벨 대응:
- 인증 오류는 즉시 내부 경고
- 충전금 부족은 사용자 복구 가능 상태
- 템플릿 UID 오류는 운영 설정 오류 가능성이 높음
- 최소 페이지 오류는 사전 검증 누락 신호

## 20. 아직 미확인인 항목

현재 문서 링크를 통해 큰 흐름은 확보했지만, 아래는 추가 확인이 필요하다.

- 실제 `GET /orders/{orderUid}` 샌드박스 응답 예시
- `api/common` 문서의 공통 헤더, pagination, rate limit, request id 정책
- `api/webhook-events`의 이벤트별 전체 스키마
- `operations/troubleshooting`의 운영 문제 해결 패턴
- `concepts/base-layer`의 base layer 상세 규칙
- `POST /books/{bookUid}/finalization`의 `Idempotency-Key` 지원 여부
- `PUT /webhooks/config`의 멱등성/재시도 정책
- `GET /credits`와 운영 가이드의 `/credits/balance` 경로 차이 정합성

특히 마지막 항목은 문서 간 경로가 다르게 보인다.
- Credits API 문서: `GET /credits`
- 운영 가이드 문서: `GET /credits/balance`

구현 전에 실제 기준 경로를 확정해야 한다.

실제 응답 기준으로 추가 확인이 필요한 점:
- `GET /book-specs`에서 placeholder처럼 보이는 레코드(`name=""`, `pageMin=0`)가 왜 포함되는지
- `GET /book-specs`의 `limit` 파라미터가 무시되는지
- `GET /templates`의 필터 파라미터 중 `templateKind`, `category`, `theme` 실제 동작
- `GET /orders`는 실제로 `data.orders`, `data.pagination` 구조임이 확인되었고, 상세 조회와 상태 필드 전체는 주문 생성 후 추가 확인 필요
- `GET /credits/transactions`는 문서 요약과 실제 응답 필드 차이가 있어 Live 환경 또는 공식 common/errors 문서와 정합성 재확인 필요
- `contents` API의 `inserted`와 `updated` 판정 규칙

## 21. 구현 우선순위 제안

1. 백엔드에 SweetBook API client 계층 생성
2. 환경 분리: `SANDBOX/LIVE` base url, key, webhook url
3. `book-specs`, `templates` 캐시 조회 구현
4. `books -> cover/contents -> finalization` 흐름 구현
5. `orders/estimate`, `orders` 구현
6. `credits` 조회 및 Sandbox 테스트 헬퍼 구현
7. 웹훅 수신 및 서명 검증 구현
8. 상태 머신과 멱등성 정책 적용

## 22. 원문 링크

- 개요: `https://api.sweetbook.com/docs/`
- 빠른 시작: `https://api.sweetbook.com/docs/quickstart/`
- 전체 워크플로우: `https://api.sweetbook.com/docs/guides/workflow/`
- 웹훅 연동 가이드: `https://api.sweetbook.com/docs/guides/webhooks/`
- 운영 전환 가이드: `https://api.sweetbook.com/docs/guides/go-live/`
- Credits API: `https://api.sweetbook.com/docs/api/credits/`
- 주문 상태 흐름: `https://api.sweetbook.com/docs/operations/order-status/`
- 갤러리 개념: `https://api.sweetbook.com/docs/concepts/gallery/`
- 컬럼 개념: `https://api.sweetbook.com/docs/concepts/column/`
- 특수 페이지 규칙: `https://api.sweetbook.com/docs/concepts/special-page-rules/`
- 멱등성 개념: `https://api.sweetbook.com/docs/concepts/idempotency/`

codex resume 019d4f10-21ce-7a33-8331-3b5da7b22a7f
codex resume 019d4f10-21ce-7a33-8331-3b5da7b22a7f