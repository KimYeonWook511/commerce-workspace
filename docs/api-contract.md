# API 계약

Frontend와 Backend 간 API 계약 문서다.
소스: `commerce-backend/docs/api-spec.md` — Backend 스펙이 변경되면 이 문서도 함께 갱신한다.

---

## 공통

### 응답 구조

```json
{
  "code": "SUCCESS",
  "message": "OK",
  "data": { ... }
}
```

- 모든 정상/오류 응답은 동일한 `ApiResponse<T>` 형태다.
- 검증 실패 시 `data`에 필드별 오류 메시지가 담길 수 있다.

```json
{
  "code": "INVALID_REQUEST",
  "message": "잘못된 요청입니다.",
  "data": {
    "email": "must not be blank",
    "password": "size must be between 8 and 20"
  }
}
```

### 인증

- 로그인/회원가입을 제외한 인증 필요 API는 `Authorization: Bearer <accessToken>` 헤더를 포함한다.
- Access Token 만료 시 `/auth/reissue`로 재발급 후 재요청한다.
- 인증 API는 응답 본문 외에도 `Authorization` 헤더와 `refreshToken` 쿠키를 함께 반환한다.

---

## 인증 API

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| POST | `/auth/signup` | ✗ | 회원 가입 후 회원 정보 + access/refresh token 발급 |
| POST | `/auth/login` | ✗ | 로그인 후 회원 정보 + access/refresh token 발급 |
| POST | `/auth/reissue` | ✗ | refresh token으로 access/refresh token 재발급 |

### POST /auth/signup

```json
// Request
{
  "email": "user@example.com",
  "password": "password123",
  "username": "tester"
}

// Response data
{
  "member": {
    "memberId": 1,
    "email": "user@example.com",
    "username": "tester"
  },
  "accessToken": "jwt-access-token",
  "refreshToken": "jwt-refresh-token"
}
```

- `email`: 이메일 형식, 필수
- `password`: 8자 이상 20자 이하, 필수
- `username`: 12자 이하, 필수

### POST /auth/login

```json
// Request
{
  "email": "user@example.com",
  "password": "password123"
}

// Response data
{
  "member": {
    "memberId": 1,
    "email": "user@example.com",
    "username": "tester"
  },
  "accessToken": "jwt-access-token",
  "refreshToken": "jwt-refresh-token"
}
```

### POST /auth/reissue

`Cookie.refreshToken`을 우선 사용하고, 쿠키가 없으면 body의 `refreshToken`을 사용한다.

```json
// Request — Cookie: refreshToken=<...> 또는 Body
{
  "refreshToken": "jwt-refresh-token"
}

// Response data
{
  "accessToken": "new-access-token",
  "refreshToken": "new-refresh-token"
}
```

---

## 상품 API

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| GET | `/products` | ✗ | 판매 중 상품 목록 조회 (최신 등록순, 재고 미포함) |
| GET | `/products/{productId}` | ✗ | 상품 상세 + 현재 재고 수량 조회 |

- 목록/상세 모두 `status`가 `ON_SALE` 또는 `SOLD_OUT`이고 삭제되지 않은 상품만 조회된다.
- 상세 응답의 `stockQuantity`는 재고 레코드가 없으면 `0`으로 응답한다.
- 미존재/비공개 상품 상세 조회: `PRODUCT-404`

```json
// GET /products/{productId} Response data
{
  "productId": 2,
  "name": "latest-product",
  "price": 3000,
  "stockQuantity": 7
}
```

---

## 장바구니 API

`ROLE` 인증 필요. 다른 aggregate(Member, Product)는 ID로만 참조한다.

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| POST | `/cart/items` | ✅ | 장바구니에 상품 담기 (이미 있으면 수량 합산 UPSERT) |
| GET | `/cart` | ✅ | 내 장바구니 조회 (최신 가격 재조립, 구매 불가 마킹) |
| PATCH | `/cart/items/{productId}` | ✅ | 항목 수량을 절대값으로 변경 |
| DELETE | `/cart/items/{productId}` | ✅ | 항목 삭제 |

### POST /cart/items

```json
// Request
{
  "productId": 123,
  "quantity": 2
}

// Response (201) data
{
  "productId": 123,
  "quantity": 5
}
```

- `quantity`: 1 이상 99 이하, 필수. 합산 결과가 99를 초과하면 거부된다.

### GET /cart

- `items`는 최근 담은 항목이 위로 오는 `createdAt DESC` 순서다.
- 각 항목 가격은 저장값이 아니라 **최신 `Product` 가격**으로 재조회된다.
- 판매 중지(`STOPPED`)되거나 삭제된 상품은 `unavailable=true`로 표시되며 `totalAmount`에서 제외된다(항목은 보존되어 사용자가 직접 삭제 가능).

```json
// Response data
{
  "items": [
    {
      "productId": 123,
      "name": "상품명",
      "price": 10000,
      "imageUrl": "https://...",
      "quantity": 2,
      "lineAmount": 20000,
      "unavailable": false
    }
  ],
  "totalAmount": 20000
}
```

### PATCH /cart/items/{productId} · DELETE /cart/items/{productId}

```json
// PATCH Request
{ "quantity": 5 }

// PATCH Response data
{ "productId": 123, "quantity": 5 }

// DELETE Response data
null
```

- 존재하지 않는 항목 또는 잘못된 path 입력(0·음수 등) 요청은 모두 `CART-404-1`로 응답한다.

### 장바구니 실패 응답 코드

| 코드 | 의미 |
|---|---|
| `CART-404-1` | 존재하지 않는 항목 수량 변경/삭제 (잘못된 path 입력 포함) |
| `CART-404-2` | 담기 시 상품 미존재 또는 삭제됨 |
| `CART-409` | 담기 시 상품 판매 중지(`STOPPED`) |
| `CART-400-1` | 잘못된 수량 입력 |
| `CART-400-2` | 수량 invariant 위반(합산 > 99 등) |
| `AUTH-401` | 비인증/잘못된 토큰 |

> 주문 생성 성공 시 주문된 `productId`는 cart에서 자동 제거된다. 별도 API는 없다.

---

## 주문 API

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| POST | `/orders` | ✅ | 주문 생성 |
| POST | `/orders/{orderId}/cancel` | ✅ | 주문 취소 |

### POST /orders

중복 요청 방지를 위해 `Idempotency-Key` 헤더가 필수다.

```json
// Request Header
Idempotency-Key: <uuid>

// Request Body
{
  "items": [
    { "productId": 1, "quantity": 2 }
  ]
}

// Response data
{
  "orderId": 1,
  "totalPrice": 20000,
  "status": "INIT"
}
```

실패 응답:
- `400 INVALID_REQUEST`: `Idempotency-Key` 헤더 누락 또는 빈 값
- `409 ORDER_IDEMPOTENCY_IN_PROGRESS`: 같은 키로 다른 요청이 처리 중 — backoff 후 재시도 권장

### POST /orders/{orderId}/cancel

```json
// Response data
{ "orderId": 1, "status": "CANCELED" }
```

---

## 결제 API

결제는 **예약(reserve) → 승인(approve)** 2단계로 동작한다. `merchantPayKey`는 서버가 발급하며 클라이언트가 만들지 않는다.

| Method | Path | 인증 필요 | 설명 |
|---|---|---|---|
| POST | `/payments/reserve` | ✅ | 결제창 준비(예약). 구 `/payments/ready` |
| GET | `/payments/naverpay/return` | ✗ | 네이버페이 완료 후 리다이렉트 수신 엔드포인트 |
| POST | `/payments/naverpay/approve` | ✅ | 네이버페이 승인 결과 반영 |

### POST /payments/reserve

> ⚠️ 경로 변경: 구 `/payments/ready` → `/payments/reserve`. 응답 본문 구조는 동일하다.

```json
// Request
{
  "orderId": 1,
  "provider": "NAVERPAY"
}

// Response (200) data
{
  "clientId": "client-id",
  "chainId": "chain-id",
  "merchantPayKey": "PAY-01HXXX...",
  "productName": "대표 상품명",
  "productCount": 2,
  "totalPayAmount": 20000,
  "taxScopeAmount": 20000,
  "taxExScopeAmount": 0,
  "returnUrl": "https://.../return?merchantPayKey=PAY-01HXXX..."
}
```

실패 응답:
- `PAYMENT_RESULT_PENDING` (409): 주문에 UNKNOWN 상태 결제 시도가 있어 차단
- `ORDER-404`: 주문 미존재 또는 다른 회원 주문
- `ORDER-409-1`: 결제 불가 주문 상태

### GET /payments/naverpay/return

네이버페이 결제 완료 후 리다이렉트되는 엔드포인트. 현재 리턴 파라미터 수신 용도로만 존재하며 별도 응답 바디가 없다.

- Query: `merchantPayKey`(필수), `resultCode`(필수), `resultMessage`/`paymentId`/`reserveId`(선택)

### POST /payments/naverpay/approve

`merchantPayKey` 기반으로 예약을 역조회해 승인을 진행한다.

```json
// Request
{
  "merchantPayKey": "PAY-01HXXX...",
  "pgPaymentId": "naver-pg-id-xxx"
}

// Response (200) data
{
  "pgPaymentId": "naver-pg-id-xxx",
  "status": "SUCCESS"
}
```

- 같은 `merchantPayKey` + 같은 `pgPaymentId`의 중복 redirect는 차단이 아니라 **기존 결제 결과를 200으로 멱등 반환**한다.

### 결제 실패 응답 코드

| 코드 | HTTP | 의미 |
|---|---|---|
| `PAYMENT_RESULT_PENDING` | 409 | 주문에 UNKNOWN 상태 결제 시도가 있어 차단. "결제 결과 확인 중" 안내 |
| `PAYMENT_RESERVATION_NOT_FOUND` | 404 | `(memberId, merchantPayKey)`로 예약 미발견 (없는 키 또는 다른 회원 키 — 존재 비노출) |
| `PAYMENT_DUPLICATE` | 409 | 이미 성공한 결제가 있는 주문에 새 승인 진입 |
| `PAYMENT_RESERVATION_ALREADY_USED` | 409 | 같은 예약을 다른 `pgPaymentId`로 동시/순차 재사용 |

---

## 관리자 API

`ROLE_ADMIN` 권한 필요.

### 상품 관리

| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/products` | 상품 등록 (201, 재고 레코드는 생성 안 함) |
| PATCH | `/admin/products/{productId}` | 상품 수정 |
| DELETE | `/admin/products/{productId}` | 상품 soft delete |

- `status`: `ON_SALE`, `SOLD_OUT`, `STOPPED` 중 하나

### 재고 관리

| Method | Path | 설명 |
|---|---|---|
| POST | `/admin/products/{productId}/stock` | 초기 재고 생성 (201, 상품별 1회) |
| POST | `/admin/products/{productId}/stock/increase` | 재고 증가 |
| POST | `/admin/products/{productId}/stock/decrease` | 재고 감소 |
| GET | `/admin/products/{productId}/stock/histories` | 재고 변경 이력 조회 (최신순) |

- `reason`: `INBOUND`, `DISPOSAL`, `ADMIN_ADJUSTMENT`, `ORDER_CANCEL_RESTORE` 중 하나
- 실패 코드: `PRODUCT-404`, `STOCK-404`, `STOCK-409`(재고 부족), `STOCK-409-2`(이미 재고 존재), `AUTH-403`, `COMMON-400`

---

## 스펙 변경 시 절차

1. `commerce-backend/docs/api-spec.md` 변경 내용 확인
2. 이 문서(`docs/api-contract.md`) 갱신
3. 영향받는 Frontend 코드 확인 및 반영
