# 05. REST API 명세

## 1. 공통 규칙

### 1.1 Base URL

```text
/api
```

### 1.2 공통 성공 응답

```json
{
  "success": true,
  "message": "요청이 성공했습니다.",
  "data": {}
}
```

### 1.3 공통 실패 응답

```json
{
  "success": false,
  "errorCode": "INSUFFICIENT_BALANCE",
  "message": "잔액이 부족합니다."
}
```

### 1.4 공통 인증 헤더

```http
Authorization: Bearer {accessToken}
```

### 1.5 공통 멱등키 헤더

```http
Idempotency-Key: {client-generated-unique-key}
```

- 충전 승인과 송금 API는 필수이다.
- 같은 사용자, 같은 API, 같은 key, 같은 request body이면 기존 응답을 재사용한다.
- 같은 key이나 request body가 다르면 `409 IDEMPOTENCY_KEY_CONFLICT`를 반환한다.

---

## 2. Auth API

### 2.1 회원가입

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/auth/signup` |
| 설명 | 신규 사용자를 생성한다. |
| 인증 | 불필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `201 Created` |

Request Header:

```http
Content-Type: application/json
```

Request Body:

```json
{
  "email": "user@example.com",
  "password": "Password1234!",
  "name": "홍길동",
  "phoneNumber": "01012345678",
  "birthDate": "1990-01-01"
}
```

Response Body:

```json
{
  "success": true,
  "message": "회원가입이 완료되었습니다.",
  "data": {
    "userId": 1,
    "email": "user@example.com",
    "name": "홍길동",
    "verified": false,
    "role": "USER"
  }
}
```

실패 상태 코드:

- `400 INVALID_INPUT_VALUE`
- `409 DUPLICATE_EMAIL`
- `409 DUPLICATE_PHONE_NUMBER`

예외 케이스:

- 이메일 형식 오류
- 비밀번호 정책 미충족
- 중복 이메일
- 중복 휴대폰 번호

### 2.2 로그인

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/auth/login` |
| 설명 | 이메일/비밀번호로 인증하고 토큰을 발급한다. |
| 인증 | 불필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Request Body:

```json
{
  "email": "user@example.com",
  "password": "Password1234!"
}
```

Response Body:

```json
{
  "success": true,
  "message": "로그인에 성공했습니다.",
  "data": {
    "tokenType": "Bearer",
    "accessToken": "access-token-value",
    "refreshToken": "refresh-token-value",
    "accessTokenExpiresIn": 1800
  }
}
```

실패 상태 코드:

- `400 INVALID_INPUT_VALUE`
- `401 LOGIN_FAILED`

예외 케이스:

- 존재하지 않는 이메일
- 비밀번호 불일치

### 2.3 토큰 재발급

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/auth/reissue` |
| 설명 | Refresh Token으로 새 Access Token을 발급한다. |
| 인증 | Access Token 불필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Request Body:

```json
{
  "refreshToken": "refresh-token-value"
}
```

Response Body:

```json
{
  "success": true,
  "message": "토큰이 재발급되었습니다.",
  "data": {
    "tokenType": "Bearer",
    "accessToken": "new-access-token-value",
    "refreshToken": "new-refresh-token-value",
    "accessTokenExpiresIn": 1800
  }
}
```

실패 상태 코드:

- `401 INVALID_REFRESH_TOKEN`
- `401 REFRESH_TOKEN_EXPIRED`

예외 케이스:

- DB에 존재하지 않는 Refresh Token
- 만료된 Refresh Token
- 이미 rotation으로 폐기된 Refresh Token

### 2.4 로그아웃

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/auth/logout` |
| 설명 | 현재 사용자의 Refresh Token을 폐기한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Request Header:

```http
Authorization: Bearer {accessToken}
```

Request Body: 없음

Response Body:

```json
{
  "success": true,
  "message": "로그아웃되었습니다.",
  "data": null
}
```

실패 상태 코드:

- `401 JWT_AUTHENTICATION_FAILED`

예외 케이스:

- Access Token 없음
- 유효하지 않은 Access Token

### 2.5 내 정보 조회

| 항목 | 내용 |
|---|---|
| Method/Path | `GET /api/users/me` |
| 설명 | 로그인 사용자의 정보를 조회한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Response Body:

```json
{
  "success": true,
  "message": "내 정보 조회에 성공했습니다.",
  "data": {
    "userId": 1,
    "email": "user@example.com",
    "name": "홍길동",
    "phoneNumber": "010****5678",
    "birthDate": "1990-01-01",
    "verified": true,
    "role": "USER",
    "createdAt": "2026-06-05T10:00:00"
  }
}
```

실패 상태 코드:

- `401 JWT_AUTHENTICATION_FAILED`
- `404 USER_NOT_FOUND`

---

## 3. Verification API

### 3.1 휴대폰 인증 코드 요청

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/verifications/phone/request` |
| 설명 | Mock 휴대폰 인증 코드를 생성한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Request Body:

```json
{
  "phoneNumber": "01012345678"
}
```

Response Body:

```json
{
  "success": true,
  "message": "인증 코드가 생성되었습니다.",
  "data": {
    "verificationId": 10,
    "expiresAt": "2026-06-05T10:05:00",
    "mockCode": "123456"
  }
}
```

> `mockCode`는 포트폴리오 개발 편의를 위한 값이다. 운영 프로필에서는 응답에서 제외해야 한다.

실패 상태 코드:

- `401 JWT_AUTHENTICATION_FAILED`
- `400 INVALID_PHONE_NUMBER`
- `429 VERIFICATION_REQUEST_LIMIT_EXCEEDED`

### 3.2 휴대폰 인증 코드 확인

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/verifications/phone/confirm` |
| 설명 | Mock 인증 코드를 검증한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Request Body:

```json
{
  "phoneNumber": "01012345678",
  "verificationCode": "123456"
}
```

Response Body:

```json
{
  "success": true,
  "message": "휴대폰 인증이 완료되었습니다.",
  "data": {
    "phoneVerified": true,
    "verifiedAt": "2026-06-05T10:03:00"
  }
}
```

실패 상태 코드:

- `400 VERIFICATION_CODE_MISMATCH`
- `400 VERIFICATION_CODE_EXPIRED`
- `429 VERIFICATION_ATTEMPT_EXCEEDED`

### 3.3 본인 인증

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/verifications/identity` |
| 설명 | 이름, 생년월일, 휴대폰 번호 기반 Mock 본인 인증을 완료한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Request Body:

```json
{
  "name": "홍길동",
  "birthDate": "1990-01-01",
  "phoneNumber": "01012345678"
}
```

Response Body:

```json
{
  "success": true,
  "message": "본인 인증이 완료되었습니다.",
  "data": {
    "userId": 1,
    "verified": true,
    "verifiedAt": "2026-06-05T10:04:00"
  }
}
```

실패 상태 코드:

- `400 PHONE_VERIFICATION_REQUIRED`
- `400 IDENTITY_VERIFICATION_FAILED`
- `401 JWT_AUTHENTICATION_FAILED`

---

## 4. Wallet API

### 4.1 지갑 개설

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/wallets` |
| 설명 | 본인 인증 완료 사용자에게 지갑/계좌를 생성한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `201 Created` |

Request Body: 없음 또는 `{}`

Response Body:

```json
{
  "success": true,
  "message": "지갑이 개설되었습니다.",
  "data": {
    "walletId": 100,
    "accountNumber": "900-123456-78901",
    "balance": 0,
    "status": "ACTIVE",
    "createdAt": "2026-06-05T10:10:00"
  }
}
```

실패 상태 코드:

- `403 VERIFICATION_REQUIRED`
- `409 WALLET_ALREADY_EXISTS`

### 4.2 내 지갑 조회

| 항목 | 내용 |
|---|---|
| Method/Path | `GET /api/wallets/me` |
| 설명 | 로그인 사용자의 지갑을 조회한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Response Body:

```json
{
  "success": true,
  "message": "지갑 조회에 성공했습니다.",
  "data": {
    "walletId": 100,
    "accountNumber": "900-123456-78901",
    "balance": 50000,
    "status": "ACTIVE",
    "createdAt": "2026-06-05T10:10:00",
    "updatedAt": "2026-06-05T10:30:00"
  }
}
```

실패 상태 코드:

- `404 WALLET_NOT_FOUND`

### 4.3 내 잔액 조회

| 항목 | 내용 |
|---|---|
| Method/Path | `GET /api/wallets/me/balance` |
| 설명 | 로그인 사용자의 지갑 잔액을 조회한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Response Body:

```json
{
  "success": true,
  "message": "잔액 조회에 성공했습니다.",
  "data": {
    "walletId": 100,
    "accountNumber": "900-123456-78901",
    "balance": 50000,
    "checkedAt": "2026-06-05T10:30:00"
  }
}
```

실패 상태 코드:

- `404 WALLET_NOT_FOUND`

### 4.4 내 지갑 잠금

| 항목 | 내용 |
|---|---|
| Method/Path | `PATCH /api/wallets/me/lock` |
| 설명 | 본인 지갑을 잠근다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Response Body:

```json
{
  "success": true,
  "message": "지갑이 잠금 처리되었습니다.",
  "data": {
    "walletId": 100,
    "status": "LOCKED"
  }
}
```

실패 상태 코드:

- `404 WALLET_NOT_FOUND`
- `400 WALLET_CLOSED`

### 4.5 내 지갑 잠금 해제

| 항목 | 내용 |
|---|---|
| Method/Path | `PATCH /api/wallets/me/unlock` |
| 설명 | 본인 지갑 잠금을 해제한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Response Body:

```json
{
  "success": true,
  "message": "지갑 잠금이 해제되었습니다.",
  "data": {
    "walletId": 100,
    "status": "ACTIVE"
  }
}
```

실패 상태 코드:

- `404 WALLET_NOT_FOUND`
- `400 WALLET_CLOSED`

---

## 5. Charge API

### 5.1 충전 요청 생성

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/charges` |
| 설명 | Mock 충전 요청을 생성한다. 잔액은 아직 증가하지 않는다. |
| 인증 | 필요 |
| 멱등키 | 선택 |
| 성공 코드 | `201 Created` |

Request Body:

```json
{
  "amount": 10000
}
```

Response Body:

```json
{
  "success": true,
  "message": "충전 요청이 생성되었습니다.",
  "data": {
    "chargeId": 200,
    "amount": 10000,
    "status": "PENDING",
    "createdAt": "2026-06-05T10:20:00"
  }
}
```

실패 상태 코드:

- `403 VERIFICATION_REQUIRED`
- `404 WALLET_NOT_FOUND`
- `423 WALLET_LOCKED`
- `400 INVALID_AMOUNT`

### 5.2 충전 승인

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/charges/{chargeId}/approve` |
| 설명 | Mock 결제 승인을 처리하고 지갑 잔액을 증가시킨다. |
| 인증 | 필요 |
| 멱등키 | 필수 |
| 성공 코드 | `200 OK` |

Request Header:

```http
Authorization: Bearer {accessToken}
Idempotency-Key: charge-approve-200-001
```

Request Body:

```json
{
  "mockPaymentKey": "mock_payment_20260605_001"
}
```

Response Body:

```json
{
  "success": true,
  "message": "충전이 승인되었습니다.",
  "data": {
    "chargeId": 200,
    "walletId": 100,
    "amount": 10000,
    "status": "SUCCESS",
    "balance": 60000,
    "transactionId": 300,
    "approvedAt": "2026-06-05T10:21:00"
  }
}
```

실패 상태 코드:

- `400 IDEMPOTENCY_KEY_REQUIRED`
- `403 RESOURCE_ACCESS_DENIED`
- `404 CHARGE_NOT_FOUND`
- `409 CHARGE_ALREADY_PROCESSED`
- `409 IDEMPOTENCY_KEY_CONFLICT`
- `423 WALLET_LOCKED`

### 5.3 충전 조회

| 항목 | 내용 |
|---|---|
| Method/Path | `GET /api/charges/{chargeId}` |
| 설명 | 본인의 충전 요청/승인 상태를 조회한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Response Body:

```json
{
  "success": true,
  "message": "충전 조회에 성공했습니다.",
  "data": {
    "chargeId": 200,
    "amount": 10000,
    "status": "SUCCESS",
    "mockPaymentKey": "mock_payment_20260605_001",
    "approvedAt": "2026-06-05T10:21:00",
    "createdAt": "2026-06-05T10:20:00"
  }
}
```

실패 상태 코드:

- `403 RESOURCE_ACCESS_DENIED`
- `404 CHARGE_NOT_FOUND`

---

## 6. Transfer API

### 6.1 송금

| 항목 | 내용 |
|---|---|
| Method/Path | `POST /api/transfers` |
| 설명 | 사용자 간 송금을 처리한다. |
| 인증 | 필요 |
| 멱등키 | 필수 |
| 성공 코드 | `201 Created` |

Request Header:

```http
Authorization: Bearer {accessToken}
Idempotency-Key: transfer-20260605-001
```

Request Body:

```json
{
  "receiverAccountNumber": "900-987654-32109",
  "amount": 5000,
  "memo": "점심값"
}
```

Response Body:

```json
{
  "success": true,
  "message": "송금이 완료되었습니다.",
  "data": {
    "sendTransactionId": 400,
    "receiveTransactionId": 401,
    "senderWalletId": 100,
    "receiverWalletId": 101,
    "amount": 5000,
    "senderBalance": 55000,
    "status": "SUCCESS",
    "createdAt": "2026-06-05T10:40:00"
  }
}
```

실패 상태 코드:

- `400 INVALID_AMOUNT`
- `400 CANNOT_TRANSFER_TO_SELF`
- `400 IDEMPOTENCY_KEY_REQUIRED`
- `403 VERIFICATION_REQUIRED`
- `404 WALLET_NOT_FOUND`
- `404 RECEIVER_WALLET_NOT_FOUND`
- `409 INSUFFICIENT_BALANCE`
- `409 IDEMPOTENCY_KEY_CONFLICT`
- `423 WALLET_LOCKED`

예외 케이스:

- 자기 자신에게 송금
- 0원 이하 요청
- 잔액 부족
- 송금자 또는 수신자 지갑 잠금
- 동일 멱등키로 다른 요청 본문 전송
- 동시 송금 중 잔액 부족으로 후속 요청 실패

### 6.2 송금/거래 단건 조회

| 항목 | 내용 |
|---|---|
| Method/Path | `GET /api/transfers/{transactionId}` |
| 설명 | 송금 관련 거래 단건을 조회한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Response Body:

```json
{
  "success": true,
  "message": "송금 내역 조회에 성공했습니다.",
  "data": {
    "transactionId": 400,
    "type": "TRANSFER_SEND",
    "status": "SUCCESS",
    "senderAccountNumber": "900-123456-78901",
    "receiverAccountNumber": "900-987654-32109",
    "amount": 5000,
    "balanceAfterTransaction": 55000,
    "memo": "점심값",
    "createdAt": "2026-06-05T10:40:00"
  }
}
```

실패 상태 코드:

- `403 RESOURCE_ACCESS_DENIED`
- `404 TRANSACTION_NOT_FOUND`

---

## 7. Transaction API

### 7.1 내 거래내역 조회

| 항목 | 내용 |
|---|---|
| Method/Path | `GET /api/transactions/me` |
| 설명 | 로그인 사용자의 거래내역을 최신순으로 조회한다. |
| 인증 | 필요 |
| 멱등키 | 불필요 |
| 성공 코드 | `200 OK` |

Query Parameters:

| 이름 | 필수 | 설명 |
|---|---:|---|
| `type` | N | `CHARGE`, `TRANSFER_SEND`, `TRANSFER_RECEIVE` |
| `status` | N | `PENDING`, `SUCCESS`, `FAILED`, `CANCELED` |
| `startDate` | N | 조회 시작일시, ISO-8601 |
| `endDate` | N | 조회 종료일시, ISO-8601 |
| `page` | N | 기본 `0` |
| `size` | N | 기본 `20` |

Request 예시:

```http
GET /api/transactions/me?type=TRANSFER_SEND&status=SUCCESS&startDate=2026-06-01T00:00:00&endDate=2026-06-05T23:59:59&page=0&size=20
```

Response Body:

```json
{
  "success": true,
  "message": "거래내역 조회에 성공했습니다.",
  "data": {
    "content": [
      {
        "transactionId": 400,
        "type": "TRANSFER_SEND",
        "status": "SUCCESS",
        "amount": 5000,
        "balanceAfterTransaction": 55000,
        "memo": "점심값",
        "createdAt": "2026-06-05T10:40:00"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "totalPages": 1,
    "first": true,
    "last": true
  }
}
```

실패 상태 코드:

- `400 INVALID_TRANSACTION_TYPE`
- `400 INVALID_TRANSACTION_STATUS`
- `400 INVALID_DATE_RANGE`
- `404 WALLET_NOT_FOUND`

### 7.2 필터링 조회

`GET /api/transactions/me?type=&status=&startDate=&endDate=&page=&size=`는 7.1 API와 동일한 엔드포인트이며, query parameter 조합으로 필터링한다.

필터 정책:

- `type`이 없으면 전체 유형 조회
- `status`가 없으면 전체 상태 조회
- `startDate`만 있으면 시작일 이후 조회
- `endDate`만 있으면 종료일 이전 조회
- `startDate > endDate`이면 `INVALID_DATE_RANGE`
- 정렬은 항상 `createdAt DESC`

---

## 8. Swagger/OpenAPI 문서화 계획

구현 단계에서 Swagger/OpenAPI에는 다음 내용을 포함한다.

- API 그룹: Auth, Verification, Wallet, Charge, Transfer, Transaction
- JWT Bearer 인증 스키마
- 공통 성공/실패 응답 스키마
- Error Code enum 설명
- 멱등키 필요 API에 `Idempotency-Key` 헤더 명시
- 요청/응답 예시 JSON
- 인증 필요 여부 설명
- 본인 인증 필요 API 설명
