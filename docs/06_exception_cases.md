# 06. 예외 케이스 정의

## 1. 공통 예외 응답 형식

```json
{
  "success": false,
  "errorCode": "ERROR_CODE",
  "message": "사용자에게 보여줄 메시지"
}
```

## 2. 예외 케이스 표

| 예외 상황 | HTTP Status | Error Code | 사용자 메시지 | 서버 처리 방식 |
|---|---:|---|---|---|
| 중복 회원가입 - 이메일 | `409 Conflict` | `DUPLICATE_EMAIL` | 이미 가입된 이메일입니다. | users.email unique 조회 또는 DB unique 예외를 도메인 예외로 변환한다. |
| 중복 회원가입 - 휴대폰 번호 | `409 Conflict` | `DUPLICATE_PHONE_NUMBER` | 이미 사용 중인 휴대폰 번호입니다. | users.phone_number unique 조회 또는 DB unique 예외를 변환한다. |
| 로그인 실패 | `401 Unauthorized` | `LOGIN_FAILED` | 이메일 또는 비밀번호가 올바르지 않습니다. | 이메일 존재 여부와 비밀번호 불일치 사유를 구분해 노출하지 않는다. |
| JWT 인증 실패 | `401 Unauthorized` | `JWT_AUTHENTICATION_FAILED` | 인증이 필요합니다. 다시 로그인해주세요. | JWT 필터에서 서명 오류, 형식 오류, 만료 등을 공통 인증 실패로 처리한다. |
| Access Token 만료 | `401 Unauthorized` | `ACCESS_TOKEN_EXPIRED` | 로그인 세션이 만료되었습니다. 토큰을 재발급해주세요. | 만료 예외를 감지해 재발급 API 사용을 유도한다. |
| Refresh Token 없음/불일치 | `401 Unauthorized` | `INVALID_REFRESH_TOKEN` | 유효하지 않은 재발급 토큰입니다. | DB 저장 토큰과 비교하고 불일치 시 재로그인을 요구한다. |
| Refresh Token 만료 | `401 Unauthorized` | `REFRESH_TOKEN_EXPIRED` | 재발급 토큰이 만료되었습니다. 다시 로그인해주세요. | 만료된 토큰은 삭제하고 재로그인을 요구한다. |
| 인증 코드 불일치 | `400 Bad Request` | `VERIFICATION_CODE_MISMATCH` | 인증 코드가 일치하지 않습니다. | attemptCount를 증가시키고 최대 시도 횟수를 검사한다. |
| 인증 코드 만료 | `400 Bad Request` | `VERIFICATION_CODE_EXPIRED` | 인증 코드가 만료되었습니다. 다시 요청해주세요. | 인증 요청 상태를 EXPIRED로 변경하거나 만료로 간주한다. |
| 인증 시도 횟수 초과 | `429 Too Many Requests` | `VERIFICATION_ATTEMPT_EXCEEDED` | 인증 시도 횟수를 초과했습니다. 다시 요청해주세요. | 해당 인증 요청을 FAILED 처리하고 재요청을 요구한다. |
| 휴대폰 인증 미완료 | `400 Bad Request` | `PHONE_VERIFICATION_REQUIRED` | 휴대폰 인증을 먼저 완료해주세요. | 최신 VERIFIED 상태의 PHONE 인증 이력을 확인한다. |
| 본인 인증 정보 불일치 | `400 Bad Request` | `IDENTITY_VERIFICATION_FAILED` | 본인 인증 정보가 일치하지 않습니다. | 사용자 이름, 생년월일, 휴대폰 번호와 요청값을 비교한다. |
| 본인 인증 미완료 | `403 Forbidden` | `VERIFICATION_REQUIRED` | 본인 인증 완료 후 이용할 수 있습니다. | 금융 기능 진입 전 User.verified 여부를 검증한다. |
| 이미 지갑/계좌가 존재함 | `409 Conflict` | `WALLET_ALREADY_EXISTS` | 이미 개설된 지갑이 있습니다. | wallets.user_id unique 제약과 사전 조회로 중복 생성을 차단한다. |
| 존재하지 않는 지갑/계좌 | `404 Not Found` | `WALLET_NOT_FOUND` | 지갑을 찾을 수 없습니다. | 현재 사용자 기준 지갑을 조회하고 없으면 예외 처리한다. |
| 존재하지 않는 수신 지갑/계좌 | `404 Not Found` | `RECEIVER_WALLET_NOT_FOUND` | 받는 사람 계좌를 찾을 수 없습니다. | receiverAccountNumber로 ACTIVE 가능 대상 지갑을 조회한다. |
| 잠긴 지갑/계좌 | `423 Locked` | `WALLET_LOCKED` | 잠긴 지갑은 사용할 수 없습니다. | 충전 승인, 송금 전 wallet.status를 검사한다. |
| 닫힌 지갑/계좌 | `400 Bad Request` | `WALLET_CLOSED` | 해지된 지갑은 사용할 수 없습니다. | unlock/charge/transfer 전 CLOSED 상태를 차단한다. |
| 잔액 부족 | `409 Conflict` | `INSUFFICIENT_BALANCE` | 잔액이 부족합니다. | 락 획득 후 최신 balance로 amount 이하 여부를 검사한다. |
| 자기 자신에게 송금 | `400 Bad Request` | `CANNOT_TRANSFER_TO_SELF` | 본인에게는 송금할 수 없습니다. | senderWallet.id와 receiverWallet.id를 비교한다. |
| 0원 이하 금액 요청 | `400 Bad Request` | `INVALID_AMOUNT` | 금액은 0원보다 커야 합니다. | 요청 DTO 검증 및 서비스 계층에서 amount > 0을 재검증한다. |
| 금액 형식 오류 | `400 Bad Request` | `INVALID_AMOUNT_FORMAT` | 금액 형식이 올바르지 않습니다. | JSON 파싱/Bean Validation 예외를 공통 응답으로 변환한다. |
| 중복 요청 처리 중 | `409 Conflict` | `DUPLICATE_REQUEST_PROCESSING` | 동일한 요청이 처리 중입니다. 잠시 후 다시 시도해주세요. | idempotency status가 PROCESSING이면 중복 실행을 차단한다. |
| 동일 멱등 요청 재시도 | `200 OK` 또는 기존 성공 코드 | `-` | 기존 요청 결과를 반환합니다. | 같은 requestHash이면 저장된 responseBody를 반환한다. |
| 멱등키 누락 | `400 Bad Request` | `IDEMPOTENCY_KEY_REQUIRED` | 중복 방지 키가 필요합니다. | 필수 API에서 Idempotency-Key 헤더 존재 여부를 검사한다. |
| 멱등키 충돌 | `409 Conflict` | `IDEMPOTENCY_KEY_CONFLICT` | 동일한 중복 방지 키로 다른 요청을 보낼 수 없습니다. | 같은 key이나 requestHash가 다르면 처리하지 않는다. |
| 권한 없는 리소스 접근 | `403 Forbidden` | `RESOURCE_ACCESS_DENIED` | 접근 권한이 없습니다. | 리소스 소유자 userId와 인증 userId를 비교한다. |
| 존재하지 않는 사용자 | `404 Not Found` | `USER_NOT_FOUND` | 사용자를 찾을 수 없습니다. | 토큰 userId로 사용자 조회 실패 시 처리한다. |
| 존재하지 않는 충전 요청 | `404 Not Found` | `CHARGE_NOT_FOUND` | 충전 요청을 찾을 수 없습니다. | chargeId 조회 후 소유자 검증 전에 존재 여부를 확인한다. |
| 이미 처리된 충전 요청 | `409 Conflict` | `CHARGE_ALREADY_PROCESSED` | 이미 처리된 충전 요청입니다. | Charge.status가 PENDING이 아니면 승인 처리하지 않는다. |
| 존재하지 않는 거래 | `404 Not Found` | `TRANSACTION_NOT_FOUND` | 거래내역을 찾을 수 없습니다. | transactionId 조회 후 사용자 관련 여부를 검증한다. |
| 잘못된 거래 유형 | `400 Bad Request` | `INVALID_TRANSACTION_TYPE` | 거래 유형이 올바르지 않습니다. | query parameter enum 변환 실패를 처리한다. |
| 잘못된 거래 상태 | `400 Bad Request` | `INVALID_TRANSACTION_STATUS` | 거래 상태가 올바르지 않습니다. | query parameter enum 변환 실패를 처리한다. |
| 잘못된 조회 기간 | `400 Bad Request` | `INVALID_DATE_RANGE` | 조회 기간이 올바르지 않습니다. | startDate가 endDate보다 늦으면 차단한다. |
| 입력값 검증 실패 | `400 Bad Request` | `INVALID_INPUT_VALUE` | 요청 값이 올바르지 않습니다. | Bean Validation 오류를 필드 단위로 정리해 반환할 수 있다. |
| 내부 서버 오류 | `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 일시적인 오류가 발생했습니다. | 예상하지 못한 예외는 로그를 남기고 민감 정보 없이 응답한다. |

## 3. 예외 처리 구현 기준

구현 단계에서는 다음 구조를 권장한다.

- 도메인 예외별 커스텀 Error Code enum 정의
- `BusinessException` 또는 도메인별 예외 클래스 정의
- `@RestControllerAdvice`로 공통 예외 응답 변환
- Bean Validation 예외, 인증 예외, 인가 예외, DB unique 예외를 공통 응답으로 매핑
- 로그에는 요청 ID, 사용자 ID, API path, errorCode를 남기되 토큰/비밀번호/인증 코드는 남기지 않는다.
