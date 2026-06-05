# 09. 구현 작업 분해

## 1. 작업 분해 원칙

- 아래 작업은 이후 구현 단계에서 사용할 백엔드 작업 목록이다.
- 현재 문서 작성 단계에서는 Controller, Service, Repository, Entity, 테스트 코드, 설정 파일을 생성하거나 수정하지 않는다.
- 각 작업은 가능한 작은 단위로 나누어 리뷰와 테스트가 가능하도록 한다.

## 2. 작업 목록

| 순서 | 작업 | 작업 목적 | 예상 생성/수정 파일 | 선행 조건 | 완료 기준 |
|---:|---|---|---|---|---|
| 1 | 프로젝트 구조 정리 | 도메인별 패키지 구조를 정하고 이후 구현 위치를 통일한다. | `src/main/java/.../global`, `auth`, `user`, `verification`, `wallet`, `charge`, `transaction` 패키지 | 현재 프로젝트 구조 확인 | 패키지 구조와 네이밍 규칙이 README 또는 문서에 정리됨 |
| 2 | 공통 응답 형식 생성 | 모든 API 응답 형식을 통일한다. | `global/response/ApiResponse` 등 | 작업 1 | 성공/실패 응답 포맷이 정의되고 샘플 API에서 사용 가능 |
| 3 | 공통 예외 처리 생성 | 도메인 예외를 일관된 HTTP 응답으로 변환한다. | `global/exception`, `ErrorCode`, `GlobalExceptionHandler` 등 | 작업 2 | 주요 예외가 공통 실패 응답으로 반환됨 |
| 4 | User 엔티티 생성 | 회원과 인증 상태의 영속 모델을 만든다. | `user/domain/User`, `UserRole`, migration 또는 schema | 작업 1 | users 테이블 매핑과 unique 제약 정의 완료 |
| 5 | 회원가입 구현 | 신규 사용자 생성과 비밀번호 암호화를 구현한다. | `auth/controller`, `auth/service`, `user/repository`, DTO | 작업 2, 3, 4 | 회원가입 성공/중복 실패 테스트 통과 |
| 6 | 로그인 구현 | 이메일/비밀번호 인증과 토큰 발급 진입점을 구현한다. | `auth/service`, `auth/controller`, login DTO | 작업 5 | 올바른 로그인 성공, 실패 시 `LOGIN_FAILED` 반환 |
| 7 | JWT 인증 구현 | Access Token 생성/검증과 SecurityContext 연동을 구현한다. | `global/security`, `JwtProvider`, `JwtAuthenticationFilter`, Security config | 작업 6 | 보호 API에서 JWT 인증이 동작함 |
| 8 | Refresh Token 구현 | 재발급과 로그아웃을 위한 Refresh Token 저장소를 구현한다. | `auth/domain/RefreshToken`, repository, service | 작업 7 | 로그인 시 저장, 재발급, 로그아웃 삭제 동작 |
| 9 | 본인 인증 엔티티 생성 | Mock 인증 요청 이력을 저장한다. | `verification/domain/UserVerification`, enums, repository | 작업 4 | user_verifications 테이블 매핑 완료 |
| 10 | Mock 휴대폰 인증 구현 | 인증 코드 요청/확인 흐름을 구현한다. | `verification/controller`, `verification/service`, DTO | 작업 9, 작업 7 | 코드 생성, 만료, 시도 횟수, 성공 처리 동작 |
| 11 | 본인 인증 완료 처리 구현 | 사용자 verified 상태 변경을 구현한다. | `verification/service`, `user` 연동 | 작업 10 | 휴대폰 인증 후 identity 검증 성공 시 `User.verified=true` |
| 12 | Wallet 엔티티 생성 | 지갑/계좌와 잔액 모델을 만든다. | `wallet/domain/Wallet`, `WalletStatus`, repository | 작업 4 | wallets 테이블 매핑, user_id/account_number unique 정의 |
| 13 | 계좌 번호 생성 로직 구현 | 중복 없는 계좌번호를 서버에서 생성한다. | `wallet/service/AccountNumberGenerator` 등 | 작업 12 | unique 계좌번호 생성 및 중복 시 재시도 |
| 14 | 계좌 개설 구현 | 인증 완료 사용자에게 지갑을 생성한다. | `wallet/controller`, `wallet/service`, DTO | 작업 11, 12, 13 | 인증 미완료/중복 개설 차단, 초기 잔액 0원 |
| 15 | 잔액 조회 구현 | 본인 지갑과 잔액 조회 API를 구현한다. | `wallet/controller`, `wallet/service`, response DTO | 작업 14 | `/api/wallets/me`, `/balance` 응답 정상 |
| 16 | Transaction 엔티티 생성 | 충전/송금 거래내역 저장 모델을 만든다. | `transaction/domain/Transaction`, enums, repository | 작업 12 | transactions 테이블 매핑 및 조회 인덱스 정의 |
| 17 | 충전 요청 구현 | Mock 충전 요청을 PENDING으로 생성한다. | `charge/domain/Charge`, `ChargeStatus`, controller/service/repository | 작업 14, 16 | 충전 요청 생성 시 잔액 불변, Charge PENDING 저장 |
| 18 | 충전 승인 구현 | 승인 시 잔액 증가와 거래내역 저장을 처리한다. | `charge/service`, `wallet` 연동, `transaction` 연동 | 작업 17 | 트랜잭션 내 잔액 증가, Charge SUCCESS, Transaction CHARGE 저장 |
| 19 | 송금 구현 | 사용자 간 송금을 처리한다. | `transfer` 또는 `transaction/service/TransferService`, DTO | 작업 15, 16 | 송금 성공 시 양측 잔액 변경 및 거래내역 저장 |
| 20 | 송금 동시성 제어 구현 | 동시 송금 시 잔액 정합성을 보장한다. | `wallet/repository` 락 조회 메서드, transfer service | 작업 19 | 비관적 락 적용, 동시성 테스트에서 잔액 정합성 유지 |
| 21 | 멱등키 구현 | 중복 충전 승인/송금을 방지한다. | `idempotency/domain/IdempotencyKey`, repository, service/filter/aspect | 작업 18, 19 | 같은 key+본문은 기존 응답, 같은 key+다른 본문은 충돌 |
| 22 | 거래 내역 조회 구현 | 내 거래내역 필터/페이징 조회를 구현한다. | `transaction/controller`, `transaction/service`, query repository/specification | 작업 16, 19 | type/status/date/page/size 필터와 최신순 정렬 동작 |
| 23 | Swagger 설정 | API 문서를 자동화한다. | OpenAPI config, annotations | 주요 API 구현 후 | Swagger UI에서 인증 스키마와 API 설명 확인 가능 |
| 24 | 테스트 코드 작성 | 핵심 기능과 동시성 검증을 자동화한다. | `src/test/java/...`, fixtures, Testcontainers 설정 | 기능 구현 완료 | 08_test_scenarios의 핵심 테스트 통과 |
| 25 | README 작성 | 프로젝트 실행/기능/설계 요약을 제공한다. | `README.md` | 주요 기능 및 테스트 완료 | 실행 방법, API 문서 URL, ERD, 테스트 방법 정리 |

## 3. 권장 구현 순서 상세

### 3.1 기반 작업

1. 프로젝트 구조 정리
2. 공통 응답 형식 생성
3. 공통 예외 처리 생성
4. User 엔티티 생성

완료 기준:

- 모든 도메인이 공통 응답/예외 구조를 사용할 준비가 된다.
- users 테이블과 기본 사용자 모델이 준비된다.

### 3.2 인증 작업

1. 회원가입 구현
2. 로그인 구현
3. JWT 인증 구현
4. Refresh Token 구현
5. 로그아웃 구현
6. 내 정보 조회 구현

완료 기준:

- 보호 API에 Access Token이 없으면 접근할 수 없다.
- Refresh Token으로 Access Token을 재발급할 수 있다.
- 로그아웃 후 Refresh Token 재사용이 불가능하다.

### 3.3 본인 인증 작업

1. 본인 인증 엔티티 생성
2. Mock 휴대폰 인증 코드 요청 구현
3. Mock 휴대폰 인증 코드 확인 구현
4. 이름/생년월일/휴대폰 번호 기반 본인 인증 완료 처리

완료 기준:

- 휴대폰 인증 성공 이력이 있어야 본인 인증 완료가 가능하다.
- 본인 인증 완료 후 `User.verified=true`가 된다.

### 3.4 지갑 작업

1. Wallet 엔티티 생성
2. 계좌번호 생성 로직 구현
3. 계좌 개설 구현
4. 내 지갑 조회 구현
5. 잔액 조회 구현
6. 잠금/잠금 해제 구현

완료 기준:

- 인증 완료 사용자만 지갑을 만들 수 있다.
- 사용자당 지갑은 1개만 생성된다.
- 계좌번호는 서버에서 unique하게 생성된다.

### 3.5 충전/송금/거래 작업

1. Transaction 엔티티 생성
2. Charge 엔티티 및 충전 요청 구현
3. 충전 승인 구현
4. 송금 구현
5. 송금 동시성 제어 구현
6. 멱등키 구현
7. 거래내역 조회 구현

완료 기준:

- 충전 승인 시 잔액과 거래내역이 함께 반영된다.
- 송금 시 두 지갑 잔액과 양측 거래내역이 함께 반영된다.
- 동시 송금에서도 잔액이 음수가 되지 않는다.
- 멱등키로 중복 요청이 방지된다.

### 3.6 문서화/검증 작업

1. Swagger 설정
2. 테스트 코드 작성
3. README 작성

완료 기준:

- Swagger UI에서 전체 API를 확인할 수 있다.
- 핵심 테스트 시나리오가 자동화되어 통과한다.
- README만 보고 프로젝트 실행과 테스트가 가능하다.

## 4. README 작성 계획

README에는 다음 내용을 포함한다.

- 프로젝트 소개
- 기술 스택
- 현재 프로젝트 설정과 목표 스택 차이
- 주요 기능
- ERD 이미지 또는 테이블 요약
- API 문서 링크
- 실행 방법
- 테스트 실행 방법
- 핵심 트랜잭션/동시성 설계 설명
- 멱등키 사용 방법
- Mock 인증/Mock 결제 사용 방법
- 향후 개선 사항

## 5. 구현 시 주의사항

- 프론트엔드 파일을 수정하지 않는다.
- 백엔드 구현 단계에서도 한 작업 단위마다 테스트를 작성한다.
- 설정 파일 변경은 작업 목적과 영향 범위를 명확히 한 뒤 수행한다.
- 실제 금융 API, 실제 본인 인증 API, 실제 결제 API는 연동하지 않는다.
- 비밀번호, JWT secret, 토큰 값, 인증 코드는 하드코딩하지 않는다.
- 민감 정보는 응답과 로그에서 제외한다.
