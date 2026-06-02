# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 언어 규칙

- 모든 설명과 답변은 반드시 한국어로 작성한다.
- 클래스, 메서드, 변수, 패키지 등 코드 식별자는 반드시 영어로 작성한다.
- 코드 식별자에는 한국어를 섞지 않는다.
- 한국어 문장에서 영문 용어 뒤 조사는 붙여 쓴다 (예: `race가`, `mock으로`, `latch는`, `stub한다`). 의존명사(`자체`, `간`, `등`)나 일반 명사는 띄어 쓴다 (예: `mock 응답`, `thread 간`).

## 워크스페이스 역할

이 워크스페이스에서 Claude Code의 역할은 다음과 같다.

- **Frontend 개발**: `commerce-frontend/` 구현을 담당한다.
- **계약 싱크**: Backend 문서(`commerce-backend/docs/`)가 변경되면 `docs/api-contract.md`를 갱신하고 Frontend에 반영한다.
- **전체 진행 현황 관리**: `docs/progress.md`로 기능별 구현 상태를 추적한다.
- **`commerce-infra/` 설정 파일 수정**: Nginx, Docker Compose 등 설정 파일은 agent가 수정한다. 단 실제 서버 배포(docker 명령 실행, 서버 적용)는 사용자가 직접 처리한다.

## 워크스페이스 구조

이 저장소는 Git 서브모듈로 구성된 모노레포다.

```
commerce-workspace/
├── docs/                  ← 서브모듈 공유 문서 (API 계약, 시스템 구조, 진행 현황)
├── commerce-backend/      ← Java/Spring Boot 백엔드 (서브모듈)
├── commerce-infra/        ← 인프라 설정 (서브모듈)
└── commerce-frontend/     ← Frontend (미개발)
```

- **`commerce-backend/`**: Java 21, Spring Boot 3.x, Gradle 기반. 레이어 구조는 `presentation → application → domain ← infrastructure`. 상세 규칙은 `commerce-backend/CLAUDE.md` 참고.
- **`commerce-infra/`**: Nginx 역프록시, Let's Encrypt SSL, Docker Compose로 운영 인프라를 관리한다. 설정 파일 수정은 agent가 담당하지만, 실제 서버 배포(docker 명령 실행, 서버 적용)는 사용자가 직접 처리한다.
- **`commerce-frontend/`**: 아직 미개발. 이 워크스페이스 세션에서 구현한다.

각 서브모듈은 독립된 Git 저장소이며, 변경사항은 해당 서브모듈 저장소에 커밋한 뒤 워크스페이스에서 포인터를 갱신한다.

## 공유 문서 (`docs/`)

서브모듈 간 계약과 전체 현황을 관리하는 문서다. Frontend 개발 시작 전 반드시 읽는다.

- `docs/system-architecture.md`: 전체 시스템 구조 (서비스 경계, 통신, 배포 환경)
- `docs/api-contract.md`: Frontend가 소비하는 Backend API 계약
- `docs/progress.md`: 기능별 Backend/Frontend 구현 현황

## 백엔드 빌드 및 실행

```bash
cd commerce-backend

# 로컬 인프라 실행 (MySQL, Redis, Kafka)
docker compose -f docker-compose.local.yml up -d

# 빌드
./gradlew build

# 애플리케이션 실행 (로컬 프로파일)
./gradlew bootRun

# 단위 + 슬라이스 테스트 (Docker 불필요)
./gradlew test

# Testcontainers 통합 테스트 (Docker 필요)
./gradlew integrationTest

# 동시성 / 데드락 테스트 (수동 실행)
./gradlew concurrencyTest

# Spring Batch 통합 테스트
./gradlew batchTest

# 외부 PG 샌드박스 테스트 (수동 실행)
./gradlew sandboxTest
```

CI는 `./gradlew test`와 `./gradlew integrationTest batchTest`를 병렬 실행한다. `concurrencyTest`, `sandboxTest`는 CI 미포함이며 영향 범위에 맞춰 수동으로 실행한다.

**테스트 태그 분류**

태그는 두 축으로 구성된다.

| 태그 | 축 | 부여 대상 | 실행 태스크 |
|---|---|---|---|
| `docker` | 환경 | Testcontainers 사용 | `integrationTest` |
| `sandbox` | 환경 | 외부 sandbox API | `sandboxTest` (수동) |
| `concurrency` | 격리 | 동시성 / 데드락 | `concurrencyTest` (수동) |
| `batch` | 격리 | Spring Batch 통합 | `batchTest` |
| `learning` | 격리 | 학습 / 디버그 (`@Disabled`) | 실행 안 됨 |

한 클래스에 두 축의 태그를 동시에 부여할 수 있다 (예: `docker + concurrency`). 격리 축 태그가 함께 붙으면 해당 격리 태스크 쪽으로만 매칭되어 자동 검증에서 빠진다.

기본 `test` 태스크는 위 5개 태그를 모두 제외하고 단위/슬라이스 테스트만 실행한다.

## Git 규칙

### 브랜치

`develop`에서 분기하고, PR도 `develop`을 base로 생성한다. 브랜치는 반드시 `git worktree add`로 생성한다.

```bash
git worktree add worktrees/<type>-<name> -b <type>/<name> develop
```

형식: `feature/<name>`, `fix/<name>`, `refactor/<name>`, `test/<name>`, `docs/<name>`, `chore/<name>`

### 커밋 메시지

```
<type>: <subject>
```

- subject는 `~한다` 형태의 한국어 동사형으로 작성한다 (예: `feat: 상품 조회 API를 추가한다`)
- `Co-Authored-By` 줄을 붙이지 않는다

허용 타입: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

### PR

- 제목: `<type>: <명사형 요약>` (예: `feat: 상품 조회 API 추가`)
- PR 생성 전 `commerce-backend/docs/pr-conventions.md`를 읽는다
- 구현·테스트 미완료 시 draft PR로 생성한다

## 컨벤션 확인 규칙

- `git commit` 전: `commerce-backend/docs/commit-conventions.md` 확인
- `gh pr create` 전: `commerce-backend/docs/pr-conventions.md` 확인
- 브랜치 생성 전: `commerce-backend/docs/branch-conventions.md` 확인

## 참고 문서

### 워크스페이스 공유

- 시스템 구조: `docs/system-architecture.md`
- API 계약: `docs/api-contract.md`
- 개발 현황: `docs/progress.md`

### 백엔드

- 아키텍처: `commerce-backend/docs/architecture.md`
- DB 스키마: `commerce-backend/docs/db-schema.md`
- API 스펙: `commerce-backend/docs/api-spec.md`
- 기능 범위: `commerce-backend/docs/PRD.md`
- 설계 결정: `commerce-backend/docs/ADR.md`
- 예외 처리 정책: `commerce-backend/docs/exception-strategy.md`
- 로깅 컨벤션: `commerce-backend/docs/logging-conventions.md`
- 테스트 컨벤션: `commerce-backend/docs/testing-conventions.md`
- 태스크별 문서 운영 가이드: `commerce-backend/docs/tasks/README.md`
- Claude Code 하네스 원칙: `commerce-backend/docs/claude-harness.md`
- DDD 마이그레이션 회고: `commerce-backend/docs/ddd/`
