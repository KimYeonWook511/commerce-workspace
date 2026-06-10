# 개발 현황

기능별 Backend/Frontend 구현 상태를 추적하는 문서다.
MVP 범위와 기능 인덱스는 `commerce-backend/docs/prd.md` 기준이다.

## 제품 기능

사용자(회원·관리자)가 실제로 사용하는 기능이다.

| 기능 | 설명 | Backend | Frontend |
|---|---|---|---|
| 인증 | 회원가입 / 로그인 / 토큰 재발급 (JWT access + Redis refresh) | ✅ | ⬜ |
| 상품 조회 | 공개 상품 목록 / 상세 조회 (비로그인 가능) | ✅ | ⬜ |
| 상품 관리 | 관리자 상품 등록 / 수정 / soft delete | ✅ | ⬜ |
| 재고 | 관리자 초기 생성 / 조정 / 이력, 주문 경로 차감·복구 | ✅ | ⬜ |
| 장바구니 | 담기(UPSERT) / 조회(최신가 재조립·구매 불가 마킹) / 수량 변경 / 삭제 | ✅ | ⬜ |
| 주문 | 생성 / 취소 / 만료 배치, `Idempotency-Key` 멱등 | ✅ | ⬜ |
| 결제 | 네이버페이 예약(reserve) / 승인(approve), UNKNOWN 마킹·보상·대사 | ✅ | ⬜ |

## 기반 기술 (Backend)

제품 기능은 아니지만 정합성·운영성을 떠받치는 기반이다. 상세·이력은 `commerce-backend/docs/adr.md`가 단일 출처다.

| 영역 | 내용 | 상태 |
|---|---|---|
| 아키텍처 정책 | cross-aggregate ID 참조 / same-aggregate 객체 참조, method-level 트랜잭션, DB unique find-first + 안전망 | ✅ |
| 마이그레이션·스키마 | Flyway 도입, `ddl-auto: validate`, enum `@JdbcTypeCode(VARCHAR)`, cross-aggregate FK 제거 | ✅ |
| 관측성 | 요청 단위 traceId 전파(HTTP·Kafka·Outbox 경계), MDC 키 통합, 경계·도메인 이벤트 로깅 표준 | ✅ |
| 이벤트·비동기 | 재고 복구 Outbox 패턴 + Kafka 전달 | ✅ |

## MVP 제외 항목

배송, 쿠폰/프로모션, 리뷰, 정산/운영 백오피스, 다중 PG 연동은 MVP 범위 밖이다.

---

## 업데이트 방법

- Backend 기능 완료: ✅ 표시
- Frontend 기능 완료: ✅ 표시
- 새 기능 추가 시: 행을 추가하고 backend `prd.md`와도 정합을 맞춘다
