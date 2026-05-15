# API 계약

Frontend와 Backend 간 API 계약 문서다.
소스: `commerce-backend/docs/api-spec.md` — Backend 스펙이 변경되면 이 문서도 함께 갱신한다.

---

## 공통

### 응답 구조

```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": { ... }
}
```

### 인증

- 로그인/회원가입을 제외한 인증 필요 API는 `Authorization: Bearer <accessToken>` 헤더를 포함한다.
- Access Token 만료 시 `/auth/reissue`로 재발급 후 재요청한다.

---

## 인증 API

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| POST | `/auth/signup` | ✗ | 회원 가입 후 access/refresh token 발급 |
| POST | `/auth/login` | ✗ | 로그인 후 access/refresh token 발급 |
| POST | `/auth/reissue` | ✗ | refresh token으로 access token 재발급 |

### POST /auth/signup

```json
// Request
{
  "email": "user@example.com",
  "password": "password123",
  "username": "홍길동"
}

// Response data
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ..."
}
```

### POST /auth/login

```json
// Request
{
  "email": "user@example.com",
  "password": "password123"
}

// Response data
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ..."
}
```

### POST /auth/reissue

```json
// Request Header
// Authorization: Bearer <refreshToken>

// Response data
{
  "accessToken": "eyJ..."
}
```

---

## 상품 API

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| GET | `/products` | ✗ | 판매 중 상품 목록 조회 |
| GET | `/products/{productId}` | ✗ | 상품 상세 + 현재 재고 수량 조회 |

---

## 주문 API

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| POST | `/orders` | ✅ | 주문 생성 |
| POST | `/orders/{orderId}/cancel` | ✅ | 주문 취소 |

### POST /orders

중복 요청 방지를 위해 `Idempotency-Key` 헤더가 필수다.

```
// Request Header
Idempotency-Key: <uuid>
```

---

## 결제 API

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| POST | `/payments/ready` | ✅ | 결제 준비 정보 생성 |
| GET | `/payments/naverpay/return` | ✗ | 네이버페이 완료 후 리다이렉트 엔드포인트 |
| POST | `/payments/naverpay/approve` | ✅ | 네이버페이 승인 결과 반영 |

---

## 관리자 API

`ROLE_ADMIN` 권한 필요.

### 상품 관리

| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/products` | 상품 등록 |
| PATCH | `/admin/products/{productId}` | 상품 수정 |
| DELETE | `/admin/products/{productId}` | 상품 soft delete |

### 재고 관리

| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/products/{productId}/stock` | 초기 재고 생성 |
| POST | `/admin/products/{productId}/stock/increase` | 재고 증가 |
| POST | `/admin/products/{productId}/stock/decrease` | 재고 감소 |
| GET | `/admin/products/{productId}/stock/histories` | 재고 변경 이력 조회 |

---

## 스펙 변경 시 절차

1. `commerce-backend/docs/api-spec.md` 변경 내용 확인
2. 이 문서(`docs/api-contract.md`) 갱신
3. 영향받는 Frontend 코드 확인 및 반영
