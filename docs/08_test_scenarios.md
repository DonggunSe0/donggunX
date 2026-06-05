# 08. 테스트 시나리오

## 1. 테스트 전략

- 단위 테스트: 도메인 검증, 금액 검증, 계좌번호 생성, 멱등키 requestHash 검증
- 서비스 테스트: 회원가입, 로그인, 인증, 지갑 개설, 충전, 송금 비즈니스 흐름
- 통합 테스트: Spring Security 인증, JPA 트랜잭션, MySQL/H2/Testcontainers 기반 동시성 검증
- Mock 테스트: JWT 유틸, PasswordEncoder, 외부 결제/인증 Mock 흐름
- 동시성 테스트: ExecutorService 또는 병렬 테스트로 동시에 송금 요청을 실행해 잔액 정합성 확인

## 2. Auth 테스트

| 테스트 | 테스트 목적 | Given | When | Then | 기대 결과 |
|---|---|---|---|---|---|
| 회원가입 성공 | 신규 사용자가 정상 생성되는지 확인 | 중복되지 않은 이메일/휴대폰 번호와 유효한 비밀번호가 있다. | `POST /api/auth/signup`을 호출한다. | users에 사용자가 저장되고 비밀번호는 BCrypt 해시로 저장된다. | `201 Created`, `verified=false`, `role=USER` 반환 |
| 중복 이메일 실패 | 이메일 unique 정책 확인 | 이미 가입된 이메일이 존재한다. | 같은 이메일로 회원가입을 요청한다. | 회원 생성이 거부된다. | `409 DUPLICATE_EMAIL` 반환 |
| 로그인 성공 | 올바른 인증 정보로 토큰 발급 확인 | 가입된 사용자와 올바른 비밀번호가 있다. | `POST /api/auth/login`을 호출한다. | Access Token과 Refresh Token이 발급되고 Refresh Token이 저장된다. | `200 OK`, 토큰 응답 반환 |
| 로그인 실패 | 잘못된 인증 정보 차단 | 가입된 이메일과 다른 비밀번호가 있다. | 로그인 요청을 보낸다. | 토큰이 발급되지 않는다. | `401 LOGIN_FAILED` 반환 |
| JWT 인증 실패 | 보호 API 접근 차단 | 유효하지 않거나 만료된 Access Token이 있다. | `GET /api/users/me`를 호출한다. | Security filter에서 인증 실패 처리된다. | `401 JWT_AUTHENTICATION_FAILED` 반환 |

## 3. Verification 테스트

| 테스트 | 테스트 목적 | Given | When | Then | 기대 결과 |
|---|---|---|---|---|---|
| 인증 코드 요청 성공 | Mock 인증 코드 생성 확인 | 로그인 사용자와 가입된 휴대폰 번호가 있다. | `POST /api/verifications/phone/request` 호출 | `UserVerification`이 `PENDING`으로 저장된다. | 인증 ID와 만료 시각 반환 |
| 인증 코드 검증 성공 | 올바른 코드 확인 처리 | `PENDING` 인증 요청과 올바른 코드가 있다. | `POST /api/verifications/phone/confirm` 호출 | 인증 상태가 `VERIFIED`로 변경되고 `verifiedAt`이 저장된다. | `phoneVerified=true` 반환 |
| 잘못된 인증 코드 실패 | 코드 불일치 처리 | `PENDING` 인증 요청과 다른 코드가 있다. | 인증 코드 확인 API 호출 | `attemptCount`가 증가하고 인증 성공 처리되지 않는다. | `400 VERIFICATION_CODE_MISMATCH` 반환 |
| 인증 완료 후 사용자 상태 변경 | Mock 본인 인증 성공 시 사용자 verified 변경 | 휴대폰 인증이 완료되어 있고 사용자 정보와 요청 정보가 일치한다. | `POST /api/verifications/identity` 호출 | `User.verified`가 true로 변경된다. | `verified=true` 반환 |

## 4. Wallet 테스트

| 테스트 | 테스트 목적 | Given | When | Then | 기대 결과 |
|---|---|---|---|---|---|
| 인증된 사용자 계좌 개설 성공 | 본인 인증 완료 후 지갑 생성 확인 | `verified=true` 사용자에게 지갑이 없다. | `POST /api/wallets` 호출 | 계좌번호가 자동 생성되고 잔액 0원 지갑이 생성된다. | `201 Created`, `ACTIVE`, `balance=0` 반환 |
| 인증되지 않은 사용자 계좌 개설 실패 | 인증 상태 접근 제어 확인 | `verified=false` 사용자에게 지갑이 없다. | 지갑 개설 API 호출 | 지갑이 생성되지 않는다. | `403 VERIFICATION_REQUIRED` 반환 |
| 중복 계좌 개설 실패 | 사용자당 1개 지갑 제한 확인 | 이미 지갑이 있는 사용자가 있다. | 지갑 개설 API 재호출 | 추가 지갑이 생성되지 않는다. | `409 WALLET_ALREADY_EXISTS` 반환 |
| 내 잔액 조회 성공 | 본인 지갑 잔액 조회 확인 | 로그인 사용자에게 지갑과 잔액이 있다. | `GET /api/wallets/me/balance` 호출 | 본인 지갑의 잔액만 조회된다. | `200 OK`, balance 반환 |
| 타인 지갑 접근 실패 | 리소스 소유자 검증 확인 | 사용자 A와 사용자 B의 지갑이 있다. | A 토큰으로 B의 리소스를 조회하거나 관련 거래를 조회한다. | 접근이 차단된다. | `403 RESOURCE_ACCESS_DENIED` 반환 |

## 5. Charge 테스트

| 테스트 | 테스트 목적 | Given | When | Then | 기대 결과 |
|---|---|---|---|---|---|
| 충전 요청 생성 | 승인 전 Charge 생성 확인 | 인증 완료 사용자와 ACTIVE 지갑이 있다. | `POST /api/charges`에 10,000원을 요청한다. | `Charge.PENDING`이 저장되고 잔액은 변하지 않는다. | `201 Created`, `status=PENDING` 반환 |
| 충전 승인 성공 | 승인 시 잔액 증가와 거래 저장 확인 | `PENDING` Charge와 ACTIVE 지갑이 있다. | `Idempotency-Key`와 함께 승인 API 호출 | 지갑 잔액 증가, Charge SUCCESS, Transaction CHARGE 저장이 한 트랜잭션으로 처리된다. | `200 OK`, 증가 후 잔액 반환 |
| 중복 승인 방지 | 동일 승인 요청 재처리 방지 | 이미 승인 성공한 Charge와 동일 멱등키가 있다. | 같은 요청을 다시 호출한다. | 잔액이 추가 증가하지 않고 기존 응답을 반환한다. | 기존 성공 응답 반환, 잔액 1회만 증가 |
| 충전 실패 처리 | Mock 승인 실패 시 잔액 불변 확인 | `PENDING` Charge가 있다. | 실패 조건의 Mock 승인 요청을 보낸다. | Charge가 FAILED로 변경되고 잔액은 변하지 않는다. | `status=FAILED`, balance unchanged |

## 6. Transfer 테스트

| 테스트 | 테스트 목적 | Given | When | Then | 기대 결과 |
|---|---|---|---|---|---|
| 송금 성공 | 잔액 이동과 양측 거래내역 저장 확인 | A 잔액 50,000원, B ACTIVE 지갑이 있다. | A가 B에게 10,000원 송금 | A 잔액 40,000원, B 잔액 10,000원이 되고 양측 거래내역이 저장된다. | `201 Created`, `TRANSFER_SEND`, `TRANSFER_RECEIVE` 저장 |
| 잔액 부족 실패 | 초과 송금 차단 | A 잔액 5,000원, 송금 요청 10,000원 | 송금 API 호출 | 잔액 변경 없이 실패한다. | `409 INSUFFICIENT_BALANCE` 반환 |
| 자기 자신에게 송금 실패 | 자기 송금 차단 | A의 계좌번호가 있다. | A가 A 계좌번호로 송금 요청 | 잔액 변경 없이 실패한다. | `400 CANNOT_TRANSFER_TO_SELF` 반환 |
| 존재하지 않는 계좌 송금 실패 | 수신자 검증 확인 | 존재하지 않는 receiverAccountNumber가 있다. | 송금 API 호출 | 송금 처리되지 않는다. | `404 RECEIVER_WALLET_NOT_FOUND` 반환 |
| 잠긴 계좌 송금 실패 | 지갑 상태 검증 확인 | A 또는 B 지갑 상태가 `LOCKED`이다. | 송금 API 호출 | 잔액 변경 없이 실패한다. | `423 WALLET_LOCKED` 반환 |
| 중복 송금 요청 방지 | 멱등키로 중복 차감 방지 | 동일 Idempotency-Key와 동일 요청 본문이 있다. | 같은 송금 API를 2회 호출 | 첫 요청만 처리되고 두 번째는 기존 응답을 반환한다. | 잔액 1회만 차감 |
| 동시 송금 시 잔액 정합성 유지 | 비관적 락 기반 lost update 방지 | A 잔액 10,000원, 8,000원 송금 요청 2건이 동시에 준비되어 있다. | 두 요청을 병렬 실행한다. | 하나만 성공하고 하나는 잔액 부족으로 실패한다. | 최종 A 잔액 2,000원, B 총 증가 8,000원 |

## 7. Transaction 테스트

| 테스트 | 테스트 목적 | Given | When | Then | 기대 결과 |
|---|---|---|---|---|---|
| 거래 내역 조회 | 내 거래 전체 조회 확인 | 사용자 지갑에 여러 거래내역이 있다. | `GET /api/transactions/me` 호출 | 로그인 사용자 관련 거래만 최신순 조회된다. | `200 OK`, `createdAt DESC` |
| 거래 유형별 조회 | type 필터 확인 | CHARGE, TRANSFER_SEND, TRANSFER_RECEIVE 거래가 있다. | `type=CHARGE`로 조회 | 충전 거래만 반환된다. | 모든 content.type이 `CHARGE` |
| 거래 상태별 조회 | status 필터 확인 | SUCCESS, FAILED 거래가 있다. | `status=SUCCESS`로 조회 | 성공 거래만 반환된다. | 모든 content.status가 `SUCCESS` |
| 페이징 조회 | page/size 동작 확인 | 25개 거래내역이 있다. | `page=0&size=10` 조회 | 첫 페이지 10개와 페이지 메타데이터가 반환된다. | `size=10`, `totalElements=25`, `totalPages=3` |

## 8. 추가 권장 테스트

| 테스트 | 목적 | 기대 결과 |
|---|---|---|
| Refresh Token 재발급 성공 | Refresh Token DB 검증과 rotation 확인 | 새 Access/Refresh Token 발급, 기존 Refresh Token 폐기 |
| 로그아웃 후 재발급 실패 | 로그아웃 시 Refresh Token 삭제 확인 | `INVALID_REFRESH_TOKEN` |
| 멱등키 충돌 실패 | 같은 key로 다른 본문 요청 차단 | `409 IDEMPOTENCY_KEY_CONFLICT` |
| 데드락 방지 테스트 | A→B와 B→A 동시 송금 안정성 확인 | 데드락 없이 둘 다 순차 처리 또는 잔액 조건에 따라 실패 |
| 잠긴 지갑 충전 승인 실패 | 충전 승인 시 지갑 상태 검증 | `423 WALLET_LOCKED`, 잔액 불변 |
