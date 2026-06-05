# 03. 보안 요구사항

## 1. 보안 설계 목표

이 서비스는 실제 금융 API를 연동하지 않는 포트폴리오 프로젝트이지만, 금융 도메인 특성상 다음 보안 원칙을 기본으로 한다.

- 인증된 사용자만 금융 리소스에 접근한다.
- 사용자는 본인의 사용자 정보, 지갑, 충전, 거래내역만 접근할 수 있다.
- 본인 인증이 완료되지 않은 사용자는 지갑 개설, 충전, 송금을 사용할 수 없다.
- 금액 변경 요청은 입력값, 권한, 상태, 멱등성, 트랜잭션 정합성을 모두 검증한다.
- 비밀번호, 토큰, 인증 코드 등 민감 정보는 응답과 로그에 노출하지 않는다.

## 2. 비밀번호 BCrypt 암호화

- 회원가입 시 비밀번호는 반드시 BCrypt로 단방향 해시 처리한다.
- 평문 비밀번호는 DB, 로그, 예외 메시지에 저장/출력하지 않는다.
- 로그인 시 입력 비밀번호를 BCrypt `matches` 방식으로 검증한다.
- 권장 설정:
  - Spring Security `PasswordEncoder`로 `BCryptPasswordEncoder` 사용
  - 비용 인자는 기본값 또는 운영 성능에 맞게 조정

## 3. JWT 기반 인증 구조

### 3.1 Access Token

| 항목 | 정책 |
|---|---|
| 목적 | API 요청 인증 |
| 저장 위치 | 클라이언트 보관, 서버 저장하지 않음 |
| 포함 정보 | userId, email, role, issuedAt, expiresAt |
| 권장 만료 | MVP 30분 |
| 사용 방식 | `Authorization: Bearer {accessToken}` |

Access Token 검증 절차:

1. Authorization 헤더 존재 여부 확인
2. Bearer prefix 확인
3. JWT 서명 검증
4. 만료 여부 검증
5. 사용자 ID와 권한 추출
6. SecurityContext에 인증 객체 저장

### 3.2 Refresh Token

| 항목 | 정책 |
|---|---|
| 목적 | Access Token 재발급 |
| 저장 위치 | MySQL `refresh_tokens` 테이블 |
| 권장 만료 | MVP 14일 |
| 폐기 시점 | 로그아웃, 재발급 rotation, 만료 |
| 저장 형태 | 구현 시 해시 저장 우선 고려 |

Refresh Token 검증 절차:

1. 요청 Refresh Token 존재 여부 확인
2. DB 저장 토큰과 일치 여부 확인
3. 만료 시각 확인
4. 사용자 상태 확인
5. 새 Access Token 발급
6. Rotation 정책 적용 시 새 Refresh Token 발급 및 기존 토큰 폐기

## 4. Refresh Token 저장 및 검증 전략

MVP에서는 MySQL 기반으로 Refresh Token을 저장한다.

- 사용자당 활성 Refresh Token은 1개로 제한한다.
- 로그인 시 기존 Refresh Token이 있으면 삭제 후 새로 저장한다.
- 재발급 시 Refresh Token Rotation을 적용한다.
- 로그아웃 시 사용자 ID 기준 Refresh Token을 삭제한다.
- 만료된 Refresh Token은 재발급 요청 시 삭제하거나 배치로 정리한다.

Redis는 초기 필수 구성에서 제외한다. 다만 이후 확장 시 Redis를 사용하면 토큰 블랙리스트, 인증 코드 TTL, 멱등키 TTL을 더 효율적으로 관리할 수 있다.

## 5. 인증이 필요한 API와 필요 없는 API

### 5.1 인증 불필요

| API | 설명 |
|---|---|
| `POST /api/auth/signup` | 회원가입 |
| `POST /api/auth/login` | 로그인 |
| `POST /api/auth/reissue` | Refresh Token 기반 토큰 재발급 |
| Swagger UI/API Docs | 개발/문서화 목적. 운영에서는 접근 제한 권장 |
| Health Check | 운영 모니터링 목적. 민감 정보 제외 |

### 5.2 인증 필요

| API | 설명 |
|---|---|
| `POST /api/auth/logout` | 로그아웃 |
| `GET /api/users/me` | 내 정보 조회 |
| `POST /api/verifications/phone/request` | 휴대폰 인증 코드 요청 |
| `POST /api/verifications/phone/confirm` | 휴대폰 인증 코드 확인 |
| `POST /api/verifications/identity` | 본인 인증 |
| `POST /api/wallets` | 지갑 개설 |
| `GET /api/wallets/me` | 내 지갑 조회 |
| `GET /api/wallets/me/balance` | 내 잔액 조회 |
| `PATCH /api/wallets/me/lock` | 내 지갑 잠금 |
| `PATCH /api/wallets/me/unlock` | 내 지갑 잠금 해제 |
| `POST /api/charges` | 충전 요청 |
| `POST /api/charges/{chargeId}/approve` | 충전 승인 |
| `GET /api/charges/{chargeId}` | 충전 조회 |
| `POST /api/transfers` | 송금 |
| `GET /api/transfers/{transactionId}` | 송금/거래 조회 |
| `GET /api/transactions/me` | 내 거래내역 조회 |

## 6. Spring Security 기반 인증/인가

구현 단계에서 다음 구조를 권장한다.

- `SecurityFilterChain`에서 인증 불필요 URL과 필요 URL을 분리한다.
- JWT 인증 필터를 UsernamePasswordAuthenticationFilter 앞에 배치한다.
- Controller에서는 가능한 한 `@AuthenticationPrincipal` 또는 커스텀 인증 객체로 현재 사용자 ID를 받는다.
- Service 계층에서 본인 리소스 접근 권한을 재검증한다.
- 관리자 권한 확장을 위해 `Role.USER`, `Role.ADMIN` enum을 둔다.

## 7. 사용자 본인 리소스 접근 제한

다음 리소스는 반드시 소유자 검증을 수행한다.

| 리소스 | 검증 기준 |
|---|---|
| User | Access Token의 userId와 요청 대상 userId 일치 |
| Wallet | wallet.userId와 현재 userId 일치 |
| Charge | charge.userId와 현재 userId 일치 |
| Transaction | senderWallet.userId 또는 receiverWallet.userId가 현재 userId와 관련 |
| RefreshToken | refreshToken.userId와 요청 사용자 일치 |

API 경로에는 가능하면 `/me`를 사용해 클라이언트가 타인 ID를 직접 넣지 않도록 한다.

## 8. 송금/충전 API 권한 검증

### 8.1 충전 권한 검증

- 로그인 사용자만 요청 가능
- 본인 인증 완료 필요
- 본인 지갑만 충전 가능
- 지갑 상태가 `ACTIVE`여야 함
- Charge 소유자가 현재 사용자여야 함
- `Idempotency-Key` 필수

### 8.2 송금 권한 검증

- 로그인 사용자만 요청 가능
- 본인 인증 완료 필요
- 송금자 지갑은 현재 사용자의 지갑이어야 함
- 받는 사람 지갑은 존재하고 `ACTIVE`여야 함
- 송금자 지갑은 `ACTIVE`여야 함
- 자기 자신에게 송금 불가
- `Idempotency-Key` 필수

## 9. 본인 인증 완료 여부 검증

다음 기능은 `User.verified=true`인 사용자만 수행할 수 있다.

- 지갑 개설
- 충전 요청 생성
- 충전 승인
- 송금
- 향후 한도 변경, 외부 계좌 연결 등 금융 기능

본인 인증 미완료 시 `403 FORBIDDEN / VERIFICATION_REQUIRED`를 반환한다.

## 10. 금액 검증

- 요청 금액은 필수값이다.
- 금액은 정수여야 한다.
- 금액은 0보다 커야 한다.
- 음수 금액과 0원 요청은 차단한다.
- 송금 금액은 송금자 잔액 이하이어야 한다.
- 충전 금액은 MVP에서는 상한을 두지 않아도 되지만, 확장 시 1회/1일 한도를 둔다.

예외 코드:

| 조건 | Error Code |
|---|---|
| 금액 null | `INVALID_AMOUNT` |
| 0원 | `INVALID_AMOUNT` |
| 음수 | `INVALID_AMOUNT` |
| 잔액 부족 | `INSUFFICIENT_BALANCE` |

## 11. 멱등키 검증

### 11.1 적용 대상

| API | 멱등키 필요 여부 | 이유 |
|---|---:|---|
| `POST /api/charges` | 선택 | 요청 생성은 중복 생성 허용 여부 정책에 따라 결정 |
| `POST /api/charges/{chargeId}/approve` | 필수 | 중복 승인 시 잔액이 중복 증가할 수 있음 |
| `POST /api/transfers` | 필수 | 중복 송금 시 잔액이 중복 차감될 수 있음 |

### 11.2 검증 규칙

- `Idempotency-Key`는 사용자별, API 경로별로 유일하게 관리한다.
- 같은 사용자, 같은 API 경로, 같은 key, 같은 requestHash이면 기존 응답을 반환한다.
- 같은 사용자, 같은 API 경로, 같은 key, 다른 requestHash이면 `409 IDEMPOTENCY_KEY_CONFLICT`를 반환한다.
- 처리 중 상태인 key에 동일 요청이 들어오면 `409 DUPLICATE_REQUEST_PROCESSING` 또는 기존 처리 완료까지 대기하지 않고 실패 응답한다.
- 만료 시간은 24시간을 권장한다.

## 12. 민감 정보 응답 제외

응답에서 제외해야 하는 정보:

- 비밀번호 해시
- Refresh Token 저장값
- 인증 코드
- JWT 서명 키
- 내부 requestHash
- 내부 version 값은 일반 조회에서는 제외 가능
- 전체 휴대폰 번호는 마스킹 권장

로그에서도 위 정보는 마스킹 또는 제외한다.

## 13. 관리자 권한 확장 구조

MVP에서는 일반 사용자만 구현하되 다음 확장을 고려한다.

- `User.role`: `USER`, `ADMIN`
- Spring Security에서 `hasRole('ADMIN')` 정책 적용 가능
- 관리자 전용 확장 API 예시:
  - 사용자 목록 조회
  - 지갑 강제 잠금/해제
  - 거래내역 모니터링
  - 이상 거래 조회
- 관리자 기능은 반드시 별도 감사 로그를 남기는 방향으로 확장한다.

## 14. 공통 예외 응답 형식

### 14.1 성공 응답

```json
{
  "success": true,
  "message": "요청이 성공했습니다.",
  "data": {}
}
```

### 14.2 실패 응답

```json
{
  "success": false,
  "errorCode": "INSUFFICIENT_BALANCE",
  "message": "잔액이 부족합니다."
}
```

### 14.3 예외 처리 원칙

- 인증 실패는 `401`을 사용한다.
- 권한 부족은 `403`을 사용한다.
- 리소스 없음은 `404`를 사용한다.
- 중복 생성/멱등키 충돌은 `409`를 사용한다.
- 입력값 오류는 `400`을 사용한다.
- 잠긴 지갑은 `423 LOCKED` 또는 호환성을 위해 `400/409` 중 하나로 통일한다. 본 설계에서는 `423`을 우선 사용한다.
