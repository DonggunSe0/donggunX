# 04. ERD 및 테이블 설계

## 1. 설계 개요

본 설계는 MySQL과 Spring Data JPA 기반 구현을 전제로 한다. Redis는 초기 필수 구성에서 제외하며, Refresh Token, 인증 코드, 멱등키도 우선 MySQL 테이블로 관리한다.

## 2. 핵심 관계 요약

```text
User 1 ── 1 Wallet
User 1 ── N UserVerification
User 1 ── N Charge
User 1 ── N RefreshToken
User 1 ── N IdempotencyKey
Wallet 1 ── N Transaction(senderWallet)
Wallet 1 ── N Transaction(receiverWallet)
Wallet 1 ── N Charge
```

설계 판단:

- 사용자와 지갑은 1:1 관계이다.
- 지갑과 거래내역은 1:N 관계이다.
- 송금은 보내는 지갑과 받는 지갑이 모두 필요하다.
- 거래내역은 append-only 성격으로 설계한다.
- 잔액 정합성을 위해 `wallets.version` 컬럼 또는 DB 락 전략을 고려한다.
- MVP 송금에서는 동시성 충돌 빈도와 잔액 정합성 중요도를 고려해 비관적 락을 우선 선택한다.

## 3. 금액 타입 결정

금액 타입은 `Long`을 선택한다.

| 후보 | 장점 | 단점 | 판단 |
|---|---|---|---|
| `Long` | 정수 원화 금액 처리 단순, 연산 빠름, 소수 오차 없음 | 소수 통화/환율 처리에 부적합 | MVP 원화 지갑에 적합 |
| `BigDecimal` | 소수 통화, 이자, 환율, 수수료 계산에 적합 | 코드 복잡도 증가, 비교/스케일 관리 필요 | 향후 다중 통화/수수료 확장 시 고려 |

MVP는 KRW 원 단위만 처리하므로 `BIGINT`/`Long`을 사용한다. 모든 금액은 0 이상 정수로 저장하며 요청 금액은 0 초과만 허용한다.

## 4. Enum 정의

### 4.1 UserRole

| 값 | 설명 |
|---|---|
| `USER` | 일반 사용자 |
| `ADMIN` | 향후 관리자 확장용 |

### 4.2 VerificationType

| 값 | 설명 |
|---|---|
| `PHONE` | 휴대폰 인증 |
| `IDENTITY` | 이름/생년월일/휴대폰 기반 Mock 본인 인증 |

### 4.3 VerificationStatus

| 값 | 설명 |
|---|---|
| `PENDING` | 인증 대기 |
| `VERIFIED` | 인증 성공 |
| `FAILED` | 인증 실패 |
| `EXPIRED` | 인증 만료 |

### 4.4 WalletStatus

| 값 | 설명 |
|---|---|
| `ACTIVE` | 정상 |
| `LOCKED` | 잠금 |
| `CLOSED` | 해지 |

### 4.5 TransactionType

| 값 | 설명 |
|---|---|
| `CHARGE` | 충전 |
| `TRANSFER_SEND` | 송금 보내기 |
| `TRANSFER_RECEIVE` | 송금 받기 |

### 4.6 TransactionStatus

| 값 | 설명 |
|---|---|
| `PENDING` | 처리 대기 |
| `SUCCESS` | 성공 |
| `FAILED` | 실패 |
| `CANCELED` | 취소 |

### 4.7 ChargeStatus

| 값 | 설명 |
|---|---|
| `PENDING` | 승인 대기 |
| `SUCCESS` | 승인 성공 |
| `FAILED` | 승인 실패 |
| `CANCELED` | 취소 |

### 4.8 IdempotencyStatus

| 값 | 설명 |
|---|---|
| `PROCESSING` | 처리 중 |
| `SUCCESS` | 처리 성공, 응답 저장됨 |
| `FAILED` | 처리 실패 |

## 5. users

테이블명: `users`

| 컬럼 | 타입 | Nullable | Unique | 인덱스 | 설명 |
|---|---|---:|---:|---|---|
| `id` | `BIGINT` | N | PK | PK | 사용자 ID |
| `email` | `VARCHAR(255)` | N | Y | `uk_users_email` | 로그인 이메일 |
| `password` | `VARCHAR(255)` | N | N | - | BCrypt 해시 비밀번호 |
| `name` | `VARCHAR(100)` | N | N | - | 사용자 이름 |
| `phone_number` | `VARCHAR(30)` | N | Y | `uk_users_phone_number` | 휴대폰 번호 |
| `birth_date` | `DATE` | N | N | - | 생년월일 |
| `verified` | `BOOLEAN` | N | N | `idx_users_verified` | 본인 인증 완료 여부 |
| `role` | `VARCHAR(20)` | N | N | `idx_users_role` | `USER`, `ADMIN` |
| `created_at` | `DATETIME(6)` | N | N | - | 생성일 |
| `updated_at` | `DATETIME(6)` | N | N | - | 수정일 |

연관관계:

- `User 1:1 Wallet`
- `User 1:N UserVerification`
- `User 1:N Charge`
- `User 1:N RefreshToken`
- `User 1:N IdempotencyKey`

설계 이유:

- 이메일은 로그인 식별자이므로 unique로 관리한다.
- 휴대폰 번호는 Mock 본인 인증에 사용되므로 MVP에서는 unique로 관리한다.
- `verified` 컬럼으로 금융 기능 접근 여부를 빠르게 판단한다.
- `role` 컬럼은 관리자 기능 확장을 위한 기반이다.

## 6. user_verifications

테이블명: `user_verifications`

| 컬럼 | 타입 | Nullable | Unique | 인덱스 | 설명 |
|---|---|---:|---:|---|---|
| `id` | `BIGINT` | N | PK | PK | 인증 요청 ID |
| `user_id` | `BIGINT` | N | N | `idx_user_verifications_user_id` | 사용자 FK |
| `verification_type` | `VARCHAR(30)` | N | N | `idx_user_verifications_type` | `PHONE`, `IDENTITY` |
| `verification_code` | `VARCHAR(20)` | Y | N | - | Mock 휴대폰 인증 코드 |
| `status` | `VARCHAR(30)` | N | N | `idx_user_verifications_status` | 인증 상태 |
| `expires_at` | `DATETIME(6)` | Y | N | `idx_user_verifications_expires_at` | 코드 만료 시각 |
| `verified_at` | `DATETIME(6)` | Y | N | - | 인증 완료 시각 |
| `attempt_count` | `INT` | N | N | - | 인증 시도 횟수 |
| `created_at` | `DATETIME(6)` | N | N | - | 생성일 |
| `updated_at` | `DATETIME(6)` | N | N | - | 수정일 |

연관관계:

- `user_verifications.user_id` → `users.id`

설계 이유:

- 인증 요청 이력을 남겨 실패 횟수, 만료, 성공 시각을 추적한다.
- 실제 SMS 또는 실명 인증 API 없이도 인증 흐름을 구현할 수 있다.
- `verification_code`는 실제 운영에서는 해시 저장 또는 Redis TTL 저장을 고려한다.

## 7. wallets

테이블명: `wallets`

| 컬럼 | 타입 | Nullable | Unique | 인덱스 | 설명 |
|---|---|---:|---:|---|---|
| `id` | `BIGINT` | N | PK | PK | 지갑 ID |
| `user_id` | `BIGINT` | N | Y | `uk_wallets_user_id` | 사용자 FK, 사용자당 1개 지갑 보장 |
| `account_number` | `VARCHAR(30)` | N | Y | `uk_wallets_account_number` | 서버 자동 생성 계좌번호 |
| `balance` | `BIGINT` | N | N | - | 잔액, KRW 원 단위 |
| `status` | `VARCHAR(20)` | N | N | `idx_wallets_status` | `ACTIVE`, `LOCKED`, `CLOSED` |
| `version` | `BIGINT` | N | N | - | 낙관적 락 또는 변경 버전 관리 |
| `created_at` | `DATETIME(6)` | N | N | - | 생성일 |
| `updated_at` | `DATETIME(6)` | N | N | - | 수정일 |

연관관계:

- `wallets.user_id` → `users.id`
- `Wallet 1:N Transaction` as sender
- `Wallet 1:N Transaction` as receiver
- `Wallet 1:N Charge`

설계 이유:

- `user_id` unique로 사용자당 지갑 1개 정책을 DB 레벨에서 보장한다.
- `account_number` unique로 송금 수신자 조회를 안전하게 처리한다.
- `balance`는 트랜잭션과 락으로 변경한다.
- `version`은 낙관적 락 확장 또는 변경 추적에 사용한다.

## 8. transactions

테이블명: `transactions`

| 컬럼 | 타입 | Nullable | Unique | 인덱스 | 설명 |
|---|---|---:|---:|---|---|
| `id` | `BIGINT` | N | PK | PK | 거래 ID |
| `transaction_type` | `VARCHAR(30)` | N | N | `idx_transactions_type` | 거래 유형 |
| `transaction_status` | `VARCHAR(30)` | N | N | `idx_transactions_status` | 거래 상태 |
| `sender_wallet_id` | `BIGINT` | Y | N | `idx_transactions_sender_wallet_id` | 보내는 지갑 FK |
| `receiver_wallet_id` | `BIGINT` | Y | N | `idx_transactions_receiver_wallet_id` | 받는 지갑 FK |
| `amount` | `BIGINT` | N | N | - | 거래 금액 |
| `balance_after_transaction` | `BIGINT` | N | N | - | 해당 거래 관점의 거래 후 잔액 |
| `memo` | `VARCHAR(255)` | Y | N | - | 메모 |
| `related_transaction_id` | `BIGINT` | Y | N | `idx_transactions_related_transaction_id` | 송금 양측 거래 연결용 |
| `created_at` | `DATETIME(6)` | N | N | `idx_transactions_created_at` | 생성일 |

연관관계:

- `transactions.sender_wallet_id` → `wallets.id`
- `transactions.receiver_wallet_id` → `wallets.id`

거래 유형별 필드 규칙:

| 유형 | senderWalletId | receiverWalletId | balanceAfterTransaction 기준 |
|---|---|---|---|
| `CHARGE` | null | 충전 대상 지갑 | 충전 대상 지갑의 증가 후 잔액 |
| `TRANSFER_SEND` | 송금자 지갑 | 수신자 지갑 | 송금자 지갑의 차감 후 잔액 |
| `TRANSFER_RECEIVE` | 송금자 지갑 | 수신자 지갑 | 수신자 지갑의 증가 후 잔액 |

설계 이유:

- 거래내역은 append-only 성격이므로 삭제/수정하지 않는 것을 원칙으로 한다.
- 송금은 양쪽 사용자 관점의 내역 조회가 필요하므로 `TRANSFER_SEND`, `TRANSFER_RECEIVE`를 각각 저장한다.
- `balance_after_transaction`을 저장해 조회 시 당시 잔액을 재현할 수 있다.

## 9. charges

테이블명: `charges`

| 컬럼 | 타입 | Nullable | Unique | 인덱스 | 설명 |
|---|---|---:|---:|---|---|
| `id` | `BIGINT` | N | PK | PK | 충전 요청 ID |
| `user_id` | `BIGINT` | N | N | `idx_charges_user_id` | 사용자 FK |
| `wallet_id` | `BIGINT` | N | N | `idx_charges_wallet_id` | 지갑 FK |
| `amount` | `BIGINT` | N | N | - | 충전 금액 |
| `status` | `VARCHAR(30)` | N | N | `idx_charges_status` | `PENDING`, `SUCCESS`, `FAILED`, `CANCELED` |
| `mock_payment_key` | `VARCHAR(100)` | Y | Y | `uk_charges_mock_payment_key` | Mock 결제 승인 키 |
| `approved_at` | `DATETIME(6)` | Y | N | - | 승인 시각 |
| `created_at` | `DATETIME(6)` | N | N | - | 생성일 |
| `updated_at` | `DATETIME(6)` | N | N | - | 수정일 |

연관관계:

- `charges.user_id` → `users.id`
- `charges.wallet_id` → `wallets.id`

설계 이유:

- 충전 요청과 승인을 분리해 실제 PG 결제 승인 흐름을 단순화한다.
- 승인 전에는 잔액을 변경하지 않는다.
- `mock_payment_key`는 중복 승인 방지 보조 식별자로 사용할 수 있다.

## 10. idempotency_keys

테이블명: `idempotency_keys`

| 컬럼 | 타입 | Nullable | Unique | 인덱스 | 설명 |
|---|---|---:|---:|---|---|
| `id` | `BIGINT` | N | PK | PK | 멱등키 ID |
| `user_id` | `BIGINT` | N | N | `idx_idempotency_keys_user_id` | 사용자 FK |
| `idempotency_key` | `VARCHAR(100)` | N | N | 복합 unique | 클라이언트 요청 멱등키 |
| `api_path` | `VARCHAR(255)` | N | N | 복합 unique | API 경로 |
| `request_hash` | `VARCHAR(128)` | N | N | - | 요청 본문 해시 |
| `response_body` | `TEXT` | Y | N | - | 성공 응답 JSON |
| `status` | `VARCHAR(30)` | N | N | `idx_idempotency_keys_status` | `PROCESSING`, `SUCCESS`, `FAILED` |
| `expires_at` | `DATETIME(6)` | N | N | `idx_idempotency_keys_expires_at` | 만료 시각 |
| `created_at` | `DATETIME(6)` | N | N | - | 생성일 |

Unique 제약:

- `uk_idempotency_user_api_key(user_id, api_path, idempotency_key)`

연관관계:

- `idempotency_keys.user_id` → `users.id`

설계 이유:

- 네트워크 재시도, 더블 클릭, 클라이언트 중복 요청으로 인한 중복 충전/송금을 방지한다.
- 같은 키로 다른 요청 본문을 보내는 경우 충돌로 처리한다.
- MySQL unique 제약으로 동시 삽입 경쟁을 차단한다.

## 11. refresh_tokens

테이블명: `refresh_tokens`

| 컬럼 | 타입 | Nullable | Unique | 인덱스 | 설명 |
|---|---|---:|---:|---|---|
| `id` | `BIGINT` | N | PK | PK | Refresh Token ID |
| `user_id` | `BIGINT` | N | Y | `uk_refresh_tokens_user_id` | 사용자 FK, 사용자당 활성 토큰 1개 |
| `refresh_token` | `VARCHAR(512)` | N | Y | `uk_refresh_tokens_refresh_token` | Refresh Token 값 또는 해시 |
| `expires_at` | `DATETIME(6)` | N | N | `idx_refresh_tokens_expires_at` | 만료 시각 |
| `created_at` | `DATETIME(6)` | N | N | - | 생성일 |

연관관계:

- `refresh_tokens.user_id` → `users.id`

설계 이유:

- Access Token은 stateless로 두고 Refresh Token만 서버에서 관리한다.
- 로그아웃과 재발급 rotation을 MySQL 기반으로 구현할 수 있다.
- 사용자당 활성 토큰 1개 정책으로 토큰 관리 복잡도를 낮춘다.

## 12. 인덱스 설계 요약

| 테이블 | 인덱스 | 목적 |
|---|---|---|
| `users` | `email`, `phone_number` unique | 가입/로그인/본인 인증 조회 |
| `wallets` | `user_id`, `account_number` unique | 사용자 지갑 조회, 송금 수신자 조회 |
| `transactions` | `sender_wallet_id`, `receiver_wallet_id`, `created_at` | 내 거래내역 최신순 조회 |
| `transactions` | `transaction_type`, `transaction_status` | 필터링 조회 |
| `charges` | `user_id`, `wallet_id`, `status` | 충전 조회/상태 처리 |
| `idempotency_keys` | `(user_id, api_path, idempotency_key)` unique | 중복 요청 차단 |
| `refresh_tokens` | `user_id`, `refresh_token`, `expires_at` | 재발급/로그아웃/만료 처리 |

## 13. 잔액 정합성 설계 판단

- 지갑 잔액 변경은 반드시 Service 계층의 트랜잭션 안에서만 수행한다.
- 송금 시 송금자와 수신자 지갑을 모두 잠근다.
- 비관적 락 사용 시 `SELECT ... FOR UPDATE`에 대응하는 JPA `PESSIMISTIC_WRITE`를 사용한다.
- 데드락 방지를 위해 두 지갑 ID를 오름차순으로 정렬해 항상 같은 순서로 락을 획득한다.
- `balance >= amount` 검증 후 차감하며, DB 체크 제약으로 `balance >= 0`을 추가하는 것도 권장한다.
