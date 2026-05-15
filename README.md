# commerce-workspace

커머스 플랫폼 개발 프로젝트

## 구성

| 서브모듈 | 설명 | 기술 스택 |
|---|---|---|
| [commerce-backend](https://github.com/KimYeonWook511/commerce-backend) | REST API 서버 | Java 21, Spring Boot 3, MySQL, Redis, Kafka |
| [commerce-infra](https://github.com/KimYeonWook511/commerce-infra) | 운영 인프라 | Nginx, Docker Compose, Let's Encrypt |
| [commerce-frontend](https://github.com/KimYeonWook511/commerce-frontend) | 웹 클라이언트 | React 18, TypeScript, Vite, TanStack Query |

## 문서

| 문서 | 설명 |
|---|---|
| [시스템 아키텍처](docs/system-architecture.md) | 전체 시스템 구조 및 환경별 구성 |
| [API 계약](docs/api-contract.md) | Frontend-Backend API 계약 |
| [개발 현황](docs/progress.md) | 기능별 구현 현황 |

## 시작하기

```bash
git clone --recurse-submodules <url>
```
