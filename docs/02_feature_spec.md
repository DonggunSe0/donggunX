# 02. 기능 명세

## 1. 공통 정책

- 모든 금액은 0원 초과 정수 원화 금액만 허용한다.
- 금액 저장 타입은 `Long`을 기본으로 하며 단위는 KRW 원이다.
- 인증이 필요한 API는 `Authorization: Bearer {accessToken}` 헤더를 사용한다.
- 충전 승인과 송금 생성처럼 중복 처리 위험이 있는 API는 `Idempotency-Key` 헤더를 필수로 사용한다.
- 공통 성공 응답은 `{ success, message, data }` 형식을 따른다.
- 공통 실패 응답은 `{ success, errorCode, message }` 형식을 따른다.
- 사용자는 본인의 리소스만 조회/변경할 수 있다.

---

## 2. Auth 기능

### 2.1 회원가입

- API: `POST /api/auth/signup`
- 인증: 불필요
- 입력값: 이메일, 비밀번호, 이름, 휴대폰 번호, 생년월일
- 처리 규칙:
  - 이메일은 전체 사용자 기준으로 유일해야 한다.
  - 비밀번호는 BCrypt로 암호화하여 저장한다.
  - 회원가입 직후 `verified=false`, `role=USER`로 생성한다.
  - 휴대폰 번호는 향후 본인 인증에 사용하므로 중복 여부 정책을 정해야 한다. MVP에서는 휴대폰 번호도 unique로 관리하는 것을 권장한다.
- 성공 결과:
  - 사용자 ID, 이메일, 이름, 인증 상태를 반환한다.
- 실패:
  - 중복 이메일: `409 CONFLICT / DUPLICATE_EMAIL`
  - 잘못된 입력값: `400 BAD_REQUEST / INVALID_INPUT_VALUE`

### 2.2 로그인

- API: `POST /api/auth/login`
- 인증: 불필요
- 입력값: 이메일, 비밀번호
- 처리 규칙:
  - 이메일로 사용자를 조회한다.
  - BCrypt로 비밀번호를 검증한다.
  - 인증 성공 시 Access Token과 Refresh Token을 발급한다.
  - Refresh Token은 DB에 저장한다.
  - 기존 Refresh Token을 어떻게 처리할지는 정책이 필요하다. MVP에서는 사용자당 활성 Refresh Token 1개를 권장하며, 재로그인 시 기존 토큰을 삭제 또는 갱신한다.
- 성공 결과:
  - `accessToken`, `refreshToken`, `tokenType`, `accessTokenExpiresIn` 반환
- 실패:
  - 이메일 또는 비밀번호 불일치: `401 UNAUTHORIZED / LOGIN_FAILED`

### 2.3 JWT Access Token 발급

- 발급 시점:
  - 로그인 성공 시
  - Refresh Token으로 재발급 성공 시
- 포함 클레임 예시:
  - `sub`: 사용자 ID
  - `email`: 사용자 이메일
  - `role`: 사용자 권한
  - `iat`: 발급 시각
  - `exp`: 만료 시각
- 권장 만료 시간:
  - MVP: 30분
  - 운영 확장 시: 보안 정책에 따라 5~30분 조정
- 서버 저장 여부:
  - Access Token은 Stateless하게 검증하며 서버 DB에 저장하지 않는다.

### 2.4 Refresh Token 발급

- 발급 시점: 로그인 성공 시
- 저장 위치: MySQL `refresh_tokens` 테이블
- 저장 필드:
  - 사용자 ID
  - Refresh Token 원문 또는 해시값
  - 만료 시각
  - 생성 시각
- 권장 만료 시간:
  - MVP: 14일
- 보안 고려:
  - 구현 단계에서는 원문 저장보다 해시 저장을 우선 고려한다.
  - 로그아웃 시 DB에서 삭제하거나 만료 처리한다.

### 2.5 Access Token 재발급

- API: `POST /api/auth/reissue`
- 인증: Access Token 불필요, Refresh Token 필요
- 입력값: Refresh Token
- 처리 규칙:
  - Refresh Token이 DB에 존재하는지 확인한다.
  - 만료 여부를 확인한다.
  - 해당 사용자 상태가 유효한지 확인한다.
  - 유효하면 새로운 Access Token을 발급한다.
  - Refresh Token Rotation 정책을 적용하는 경우 새 Refresh Token도 함께 발급하고 기존 Refresh Token을 폐기한다.
- MVP 정책:
  - 보안 강화를 위해 Refresh Token Rotation을 권장한다.
  - 재발급 성공 시 새 Access Token과 새 Refresh Token을 함께 반환한다.
- 실패:
  - 토큰 없음/불일치: `401 UNAUTHORIZED / INVALID_REFRESH_TOKEN`
  - 만료: `401 UNAUTHORIZED / REFRESH_TOKEN_EXPIRED`

### 2.6 로그아웃

- API: `POST /api/auth/logout`
- 인증: 필요
- 처리 규칙:
  - 현재 사용자 ID 기준으로 저장된 Refresh Token을 삭제한다.
  - Access Token은 Stateless 구조이므로 즉시 무효화하려면 블랙리스트 저장소가 필요하다.
  - MVP에서는 Refresh Token 삭제를 로그아웃 기준으로 한다.
  - Redis를 사용하지 않는 MVP에서는 Access Token 만료 시간 짧게 유지로 위험을 줄인다.
- 성공 결과:
  - 로그아웃 성공 메시지 반환

### 2.7 내 정보 조회

- API: `GET /api/users/me`
- 인증: 필요
- 처리 규칙:
  - Access Token에서 사용자 ID를 추출한다.
  - DB에서 사용자 정보를 조회한다.
  - 비밀번호, Refresh Token 등 민감 정보는 반환하지 않는다.
- 반환 정보:
  - 사용자 ID, 이메일, 이름, 휴대폰 번호 마스킹 값, 생년월일, 인증 상태, 권한, 가입일

---

## 3. Mock 본인 인증 기능

### 3.1 휴대폰 인증 코드 요청

- API: `POST /api/verifications/phone/request`
- 인증: 필요
- 입력값: 휴대폰 번호
- 처리 규칙:
  - 로그인 사용자의 휴대폰 번호와 요청 휴대폰 번호가 일치하는지 확인한다.
  - 6자리 숫자 Mock 인증 코드를 생성한다.
  - `UserVerification`을 `PENDING` 상태로 저장한다.
  - 만료 시간은 생성 시각 기준 5분을 권장한다.
  - 실제 SMS 발송은 하지 않는다.
  - 포트폴리오 편의를 위해 개발 프로필에서는 응답에 Mock 인증 코드를 포함할 수 있으나, 운영 프로필에서는 제외하는 설계가 바람직하다.
- 실패:
  - 휴대폰 번호 불일치: `400 INVALID_PHONE_NUMBER`
  - 너무 잦은 요청: `429 TOO_MANY_REQUESTS` 또는 MVP에서는 `400 VERIFICATION_REQUEST_LIMIT_EXCEEDED`

### 3.2 휴대폰 인증 코드 확인

- API: `POST /api/verifications/phone/confirm`
- 인증: 필요
- 입력값: 휴대폰 번호, 인증 코드
- 처리 규칙:
  - 최신 `PENDING` 인증 요청을 조회한다.
  - 만료 여부를 확인한다.
  - 인증 코드 일치 여부를 확인한다.
  - 실패 시 `attemptCount`를 증가한다.
  - 최대 실패 횟수는 5회로 제한한다.
  - 성공 시 `status=VERIFIED`, `verifiedAt=now`로 변경한다.
- 실패:
  - 코드 불일치: `400 VERIFICATION_CODE_MISMATCH`
  - 코드 만료: `400 VERIFICATION_CODE_EXPIRED`
  - 실패 횟수 초과: `429 VERIFICATION_ATTEMPT_EXCEEDED`

### 3.3 이름, 생년월일, 휴대폰 번호 기반 본인 인증

- API: `POST /api/verifications/identity`
- 인증: 필요
- 입력값: 이름, 생년월일, 휴대폰 번호
- 처리 규칙:
  - 입력값이 회원가입 정보와 일치해야 한다.
  - 휴대폰 인증 코드 확인이 완료된 이력이 있어야 한다.
  - 조건을 만족하면 `User.verified=true`로 변경한다.
  - 본인 인증 완료 시각은 별도 컬럼 또는 verification record로 추적한다.
- 실패:
  - 입력 정보 불일치: `400 IDENTITY_VERIFICATION_FAILED`
  - 휴대폰 코드 인증 미완료: `400 PHONE_VERIFICATION_REQUIRED`

### 3.4 인증 상태 기반 접근 제어

- 본인 인증 완료 전 허용:
  - 회원가입
  - 로그인
  - 토큰 재발급
  - 로그아웃
  - 내 정보 조회
  - 휴대폰 인증 요청/확인
  - 본인 인증 요청
- 본인 인증 완료 후 허용:
  - 지갑 개설
  - 충전 요청/승인
  - 송금
  - 거래내역 조회

---

## 4. 계좌 / 지갑 기능

### 4.1 지갑 개설

- API: `POST /api/wallets`
- 인증: 필요
- 전제 조건:
  - 사용자 `verified=true`
  - 사용자에게 기존 지갑이 없어야 함
- 처리 규칙:
  - 서버가 계좌번호를 자동 생성한다.
  - 계좌번호는 unique 해야 한다.
  - 초기 잔액은 0원이다.
  - 지갑 상태는 `ACTIVE`로 생성한다.
  - 사용자당 지갑/계좌는 1개만 생성 가능하다.
- 계좌번호 생성 예시:
  - prefix `900`
  - 날짜 또는 랜덤 숫자 조합
  - 중복 발생 시 재생성
  - 예: `900-123456-78901`
- 실패:
  - 본인 인증 미완료: `403 VERIFICATION_REQUIRED`
  - 이미 지갑 존재: `409 WALLET_ALREADY_EXISTS`

### 4.2 내 지갑 조회

- API: `GET /api/wallets/me`
- 인증: 필요
- 처리 규칙:
  - 로그인 사용자 ID 기준으로 지갑을 조회한다.
  - 다른 사용자의 지갑은 조회할 수 없다.
- 반환 정보:
  - 지갑 ID, 계좌번호, 잔액, 상태, 생성일, 수정일

### 4.3 잔액 조회

- API: `GET /api/wallets/me/balance`
- 인증: 필요
- 처리 규칙:
  - 로그인 사용자의 지갑만 조회한다.
  - 잔액은 `balance` 값만 별도 반환한다.
- 반환 정보:
  - 지갑 ID, 계좌번호, 잔액, 조회 시각

### 4.4 지갑 잠금

- API: `PATCH /api/wallets/me/lock`
- 인증: 필요
- 처리 규칙:
  - 본인의 지갑 상태를 `LOCKED`로 변경한다.
  - 잠긴 지갑은 송금 보내기/받기 및 충전 승인 대상에서 제외한다.
  - 이미 `LOCKED` 상태면 성공으로 볼지 예외로 볼지 정책이 필요하다. MVP에서는 멱등적 상태 변경으로 성공 응답을 권장한다.

### 4.5 지갑 잠금 해제

- API: `PATCH /api/wallets/me/unlock`
- 인증: 필요
- 처리 규칙:
  - 본인의 지갑 상태를 `ACTIVE`로 변경한다.
  - `CLOSED` 지갑은 해제할 수 없다.
- 실패:
  - 닫힌 지갑: `400 WALLET_CLOSED`

### 4.6 지갑 상태

| 상태 | 설명 | 충전 | 송금 보내기 | 송금 받기 | 조회 |
|---|---|---:|---:|---:|---:|
| `ACTIVE` | 정상 사용 가능 | 가능 | 가능 | 가능 | 가능 |
| `LOCKED` | 사용자 또는 운영 정책으로 잠금 | 불가 | 불가 | 불가 | 가능 |
| `CLOSED` | 해지된 지갑 | 불가 | 불가 | 불가 | 제한적 가능 |

---

## 5. Mock 충전 기능

### 5.1 충전 요청 생성

- API: `POST /api/charges`
- 인증: 필요
- 입력값: 금액
- 처리 규칙:
  - 사용자는 인증 완료 상태여야 한다.
  - 지갑이 존재하고 `ACTIVE` 상태여야 한다.
  - 금액은 0원 초과여야 한다.
  - `Charge`를 `PENDING` 상태로 생성한다.
  - 이 단계에서는 잔액을 증가시키지 않는다.
- 반환 정보:
  - `chargeId`, 금액, 상태, 생성일

### 5.2 충전 승인

- API: `POST /api/charges/{chargeId}/approve`
- 인증: 필요
- 필수 헤더: `Idempotency-Key`
- 입력값: `mockPaymentKey`
- 처리 규칙:
  - 요청 사용자가 해당 Charge의 소유자인지 확인한다.
  - Charge 상태가 `PENDING`인지 확인한다.
  - Idempotency-Key 중복 여부와 requestHash 충돌 여부를 확인한다.
  - 하나의 DB 트랜잭션 안에서 다음을 처리한다.
    1. 지갑 조회 및 잠금
    2. 지갑 상태 검증
    3. 지갑 잔액 증가
    4. Charge 상태를 `SUCCESS`로 변경
    5. `Transaction`을 `CHARGE/SUCCESS`로 저장
    6. 멱등키 응답 저장
- 실패:
  - 이미 승인된 Charge: `409 CHARGE_ALREADY_PROCESSED`
  - 지갑 잠금: `423 WALLET_LOCKED`
  - 멱등키 충돌: `409 IDEMPOTENCY_KEY_CONFLICT`

### 5.3 충전 실패 처리

- 실패 발생 위치:
  - Mock 결제 승인 실패
  - Charge 상태 불일치
  - 지갑 상태 불일치
  - 멱등키 검증 실패
- 처리 규칙:
  - Mock 결제 승인 실패는 Charge를 `FAILED`로 변경할 수 있다.
  - 서버 검증 실패는 잔액 변경 없이 예외 응답한다.
  - 트랜잭션 내부 오류 발생 시 모든 잔액/상태 변경은 롤백한다.

### 5.4 충전 조회

- API: `GET /api/charges/{chargeId}`
- 인증: 필요
- 처리 규칙:
  - 로그인 사용자 본인의 Charge만 조회 가능하다.

---

## 6. 송금 기능

### 6.1 송금 생성

- API: `POST /api/transfers`
- 인증: 필요
- 필수 헤더: `Idempotency-Key`
- 입력값:
  - 받는 사람 계좌번호 또는 지갑 식별자
  - 금액
  - 메모
- 처리 규칙:
  - 송금자는 본인 인증 완료 상태여야 한다.
  - 송금자 지갑이 존재하고 `ACTIVE`여야 한다.
  - 받는 사람 지갑이 존재하고 `ACTIVE`여야 한다.
  - 자기 자신에게 송금할 수 없다.
  - 금액은 0원 초과여야 한다.
  - 송금자 잔액보다 큰 금액은 송금할 수 없다.
  - 동일 `Idempotency-Key`와 동일 요청 본문이면 저장된 응답을 반환한다.
  - 동일 `Idempotency-Key`이나 요청 본문이 다르면 `IDEMPOTENCY_KEY_CONFLICT`로 차단한다.

### 6.2 받는 사람 조회 방식

MVP에서는 다음 중 하나를 받는 사람 식별자로 사용한다.

| 방식 | 설명 | 권장 여부 |
|---|---|---|
| 계좌번호 | 사용자에게 노출 가능한 식별자 | 권장 |
| walletId | 내부 DB 식별자 | API 내부/테스트용으로만 제한 권장 |

기본 송금 요청은 `receiverAccountNumber`를 사용한다.

### 6.3 송금 트랜잭션 처리

송금 성공 시 하나의 트랜잭션에서 다음 작업을 모두 수행한다.

1. 멱등키 레코드 생성 또는 검증
2. 송금자/수신자 지갑 락 획득
3. 지갑 상태 검증
4. 송금자 잔액 검증
5. 송금자 잔액 차감
6. 수신자 잔액 증가
7. 송금자 거래내역 `TRANSFER_SEND/SUCCESS` 저장
8. 수신자 거래내역 `TRANSFER_RECEIVE/SUCCESS` 저장
9. 멱등키 응답 저장

중간 단계 실패 시 잔액 변경과 거래내역 저장은 롤백한다.

### 6.4 송금 조회

- API: `GET /api/transfers/{transactionId}`
- 인증: 필요
- 처리 규칙:
  - 해당 transaction이 로그인 사용자의 지갑과 관련된 경우에만 조회 가능하다.
  - `TRANSFER_SEND` 또는 `TRANSFER_RECEIVE` 거래내역을 반환한다.

---

## 7. 거래 내역 기능

### 7.1 내 거래내역 조회

- API: `GET /api/transactions/me`
- 인증: 필요
- 처리 규칙:
  - 로그인 사용자 지갑과 관련된 거래만 조회한다.
  - 기본 정렬은 `createdAt DESC`이다.
  - 기본 페이지는 `page=0`, `size=20`이다.

### 7.2 필터링 조회

- API: `GET /api/transactions/me?type=&status=&startDate=&endDate=&page=&size=`
- 필터:
  - `type`: 거래 유형
  - `status`: 거래 상태
  - `startDate`: 시작 일시 또는 날짜
  - `endDate`: 종료 일시 또는 날짜
  - `page`: 0부터 시작
  - `size`: 페이지 크기

### 7.3 거래 유형

| 값 | 설명 |
|---|---|
| `CHARGE` | 충전 |
| `TRANSFER_SEND` | 송금 보내기 |
| `TRANSFER_RECEIVE` | 송금 받기 |

### 7.4 거래 상태

| 값 | 설명 |
|---|---|
| `PENDING` | 처리 대기 |
| `SUCCESS` | 성공 |
| `FAILED` | 실패 |
| `CANCELED` | 취소 |

### 7.5 거래내역 저장 원칙

- 거래내역은 append-only 성격으로 설계한다.
- 성공 거래내역은 원칙적으로 수정하지 않는다.
- 상태 변경이 필요한 경우 별도 상태 컬럼만 제한적으로 변경하거나 보정 거래를 추가하는 방식으로 확장한다.
- 송금은 보내는 사람과 받는 사람 각각의 관점에서 조회 가능해야 하므로 양측 거래내역을 저장한다.
