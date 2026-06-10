# 시스템 아키텍처

## 전체 구성

```
사용자 (Browser)
    │
    ▼
Nginx (역프록시, SSL 종단)          ← commerce-infra/docker/
    │
    ├── / (정적 리소스)              ← Frontend (예정)
    │
    └── /api/** (API 요청)
         │
         ▼
    Backend API (Spring Boot)       ← commerce-backend/
         │
         ├── MySQL 8.0              ← 영속 데이터 (도메인 모델, Flyway 관리)
         ├── Redis 7.4              ← JWT Refresh Token, 주문 멱등 in-flight 마커
         ├── Kafka 4.x              ← 재고 복구 이벤트 (Outbox → relay → consumer)
         └── 네이버페이 PG          ← 외부 결제 (예약·승인·대사)
```

---

## 환경별 구성

### 로컬 개발

`commerce-backend/docker-compose.local.yml`으로 로컬 인프라를 띄운다.

| 서비스 | 이미지 | 호스트 포트 |
|---|---|---|
| MySQL | mysql:8.0 | 13336 |
| Redis | redis:7.4 | 16379 |
| Kafka | apache/kafka:4.1.1 | 19092 (KRaft 단일 브로커) |
| Kafka UI | provectuslabs/kafka-ui | 18080 |

```bash
cd commerce-backend
docker compose -f docker-compose.local.yml up -d
```

### 프로덕션

두 개의 Docker Compose로 역할이 분리된다.

| 파일 | 역할 |
|---|---|
| `commerce-infra/docker/docker-compose.infra.yml` | Nginx, Certbot (인프라 레이어) |
| `commerce-backend/docker-compose.prod.yml` | Spring Boot 앱 컨테이너 |

두 Compose는 `commerce-network` 외부 네트워크로 통신한다.

```
[Nginx 컨테이너]  →  commerce-network  →  [Backend 앱 컨테이너]
     ↑
[Certbot SSL]
```

---

## 인프라 책임

| 구성 요소 | 관리 위치 | 배포 주체 |
|---|---|---|
| Nginx 역프록시 | `commerce-infra/docker/nginx/` | 사용자 직접 |
| SSL 인증서 | Let's Encrypt + certbot | 사용자 직접 (`cert-issue.sh`, `cert-renew.sh`) |
| SSL 자동 갱신 | OS cron (`commerce-infra/docs/cron.md`) | 사용자 직접 |
| 로컬 인프라 | `commerce-backend/docker-compose.local.yml` | 개발자 |
| 앱 배포 | `commerce-backend/docker-compose.prod.yml` | CI/CD (`IMAGE_NAME` 환경변수로 주입) |

---

## Backend 내부 레이어

```
presentation  →  application  →  domain  ←  infrastructure
(Controller)     (Service)      (Entity)     (JPA, Redis, Kafka, PG)
```

- `application/port/`: Redis, 결제 PG 등 외부 시스템 연동 인터페이스
- `infrastructure/`: port 구현체, JPA Repository, 외부 API Client
- 도메인 레이어는 infrastructure를 직접 참조하지 않는다
- cross-aggregate는 ID 참조, same-aggregate만 객체 참조한다 (DB 레벨 cross-aggregate FK는 제거됨)

### 주요 도메인

`auth`, `member`, `product`, `stock`, `cart`, `order`, `payment`(`naverpay` 포함), `outbox`로 구성된다.

### 핵심 비동기·외부 연동 흐름

- **결제 (2단계)**: 예약(`/payments/reserve`, `PaymentReservation` 발급) → 승인(`/payments/naverpay/approve`, `Payment` append-only 기록). PG 결과 불명 시 UNKNOWN 마킹 후 `@Scheduled` 대사(reconciliation)가 PG 이력 조회로 확정·보상한다.
- **재고 복구 이벤트**: 주문 취소·만료 시 Outbox에 적재 → 스케줄러 relay → Kafka → consumer가 재고를 복구한다.
- **관측성**: 요청 단위 traceId가 HTTP·Kafka·Outbox 경계를 넘어 전파되어 로그에 일관 추적된다.

자세한 내용은 `commerce-backend/docs/architecture.md` 참고.
