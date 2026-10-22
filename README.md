# PawBridge Backend — MSA 버전 보존 저장소

> 이 저장소는 PawBridge 백엔드의 **마이크로서비스(MSA) 운영 버전**을 보존합니다. 백엔드는 Java 21 모듈형 모놀리스로 전환 중이며, 전환 후 개발은 [pawbridge-backend-k8s](https://github.com/pawbridge/pawbridge-backend-k8s)에서 이어집니다.
>
> - 커밋 이력은 원본 저장소 `main`(운영 릴리스 소스)을 그대로 옮겼습니다. 운영 전환 직전에 최종본을 한 번 더 반영하고 읽기 전용(Archived)으로 전환합니다.
> - 이 저장소는 포트폴리오와, 이후 MSA로 다시 나눌 때의 참고용입니다. 운영 설정·비밀값은 포함하지 않습니다.

### 보존 시점의 운영 구성 (2026-10-22, Mac mini Docker Compose)

| 서비스 | 운영 이미지 빌드 커밋 |
| --- | --- |
| api-gateway, user-service, community-service, store-service, payment-service | [`501d513`](https://github.com/pawbridge/pawbridge-msa/commit/501d513b76a7340c6a03fcb34ebb55f0d1984650) |
| animal-service | [`3909f4f`](https://github.com/pawbridge/pawbridge-msa/commit/3909f4faab0f1e954922697aca587c2992c0328f) |
| photo, python-api, native AI 연동 | 별도 저장소 |

서비스 간 통신은 각 서비스의 Transactional Outbox → Debezium CDC → Kafka 소비자(찜 수 반영·찜 보상·결제→주문)로 이루어졌고, 외부 진입은 Spring Cloud Gateway가 JWT 검증과 경로 분배를 맡았습니다. 데이터는 단일 PostgreSQL 17의 서비스별 스키마로 나뉩니다.

**모듈형 모놀리스로 바꾼 이유:** 개인 프로젝트 규모에서 서비스 5개와 Gateway·Kafka·Debezium을 함께 운영하는 비용이 컸고, 서비스들이 이미 같은 PostgreSQL을 쓰고 있어 하나의 트랜잭션으로 묶으면 이벤트 전달·보상·재시도 없이 정합성을 지킬 수 있었기 때문입니다.

---

## 현재 운영과 개발

2026-10-19 기준 운영 환경은 Mac의 Docker Compose이며, PostgreSQL 17과 pgvector를 사용합니다. 기존 Java 17 마이크로서비스를 운영하고, 동물 사진 검색·추천용 Python AI는 별도 Mac 네이티브 프로세스로 실행합니다. 현재 Mac 운영에서도 Kafka·Debezium CDC·Redis를 계속 사용합니다.

`dev`에서는 Java 21 기반 [모듈형 모놀리스 backend-app](https://github.com/pawbridge/pawbridge-backend-k8s/tree/dev/backend-app)을 개발·통합합니다.

| 브랜치 | 역할 |
| --- | --- |
| `main` | 승인된 운영 릴리스의 소스 기준과 운영 hotfix 통합 |
| `dev` | 모듈형 모놀리스 전환 기능 개발·통합 |

현재 CI 정의는 검사 전용으로 이미지 발행을 차단합니다. main 기준선 정렬과 CI 변경은 실제 운영 이미지 교체나 배포 완료를 뜻하지 않습니다.

## 이전 Kubernetes 환경의 구현 기록

아래 Kubernetes·MySQL·Elasticsearch·로컬 VM 구성과 기술 표, 구현 경험·성과·이미지는 이전 환경의 기록입니다. 현재 운영 구성과 구분해 보존합니다.

<img width="1919" height="920" alt="Image" src="https://github.com/user-attachments/assets/2d016c69-7d91-4f1a-8111-abbb773ec5a7" />
반려동물 입양 프로세스, 커뮤니티, 스토어 통합 커머스 플랫폼인 PawBridge의 마이크로서비스 백엔드 저장소입니다. 
기존 AWS EC2 다중 노드 환경에서 운영되던 서비스를 개인 로컬 PC 기반의 Kubernetes(K8s) 클러스터로 마이그레이션한 버전입니다.

## 핵심 기술 아키텍처 (Key Architecture)

본 프로젝트는 대용량 트래픽 환경과 데이터 정합성 강화를 고려한 아키텍처 적용에 집중했습니다.

### 1. Transactional Outbox + CDC 기반 데이터 동기화
- Problem: 애플리케이션이 DB 저장과 후속 이벤트 발행을 직접 각각 처리할 경우, 이중 쓰기 문제로 인해 데이터 정합성이 깨질 수 있음
- Solution (CDC + Kafka): 
  - Debezium Source Connector가 MySQL Binlog를 읽어 outbox 변경 이벤트를 캡처하도록 구성
  - 캡처된 변경 이벤트를 Apache Kafka로 전달하고, 필요한 후속 소비자가 이를 처리하도록 구성
  - 애플리케이션에서는 비즈니스 데이터와 outbox 이벤트를 MySQL 단일 트랜잭션으로 함께 저장하고, Debezium CDC가 이를 비동기 전파하도록 구성해 이중 쓰기 문제를 줄이고 데이터 정합성을 강화

### 2. Redis 기반 분산 캐시 및 인증 처리
- 다중 인스턴스 환경에서 상태를 공유하기 위해 Redis를 인증 데이터 저장소와 분산 캐시로 활용
- 이메일 인증 과정에서 임시 인증 코드의 TTL을 관리하고, 빠른 읽기/쓰기 성능이 필요한 토큰 처리에 활용

### 3. Elasticsearch 기반 한국어 검색 최적화
- Nori (한글 형태소 분석기) 플러그인을 적용하여 유기동물 품종, 특징, 상품 카테고리를 자연어로 정확히 검색할 수 있도록 커스텀 인덱스 맵핑 및 Ingest 파이프라인 구축

### 4. Spring Batch 데이터 파이프라인
- 공공데이터(APMS)를 주기적으로 수집·정제하고, 서비스 DB 및 검색 인덱스에 반영하는 배치 파이프라인을 구현했습니다.

### 5. Monitoring & Observability
- Prometheus + Grafana + Zipkin 자체 호스팅으로 5개 마이크로서비스 통합 모니터링 구축
- JVM 힙/GC/스레드, HTTP p95·p99·RPS·에러율, Node Exporter 기반 CPU·메모리 통합 관리
- k6 부하 테스트(100 VUs, 5분) 결과를 Prometheus remote write로 Grafana에 실시간 연동
- **결과: 100 VUs 부하 테스트에서 Peak RPS 58.2/s, 에러율 0%를 기록했습니다. 또한 Worker 노드 메모리 사용률이 92%까지 상승하는 구간을 관찰해 리소스 사용량 점검 필요성을 확인했습니다.**
<img width="1920" height="918" alt="Image" src="https://github.com/user-attachments/assets/4838b2c7-4d34-466e-b9db-2249253cd0a1" />
---

## 마이그레이션 배경 (Why K8s & Local VM)
<img width="1041" height="786" alt="Image" src="https://github.com/user-attachments/assets/bd6107f9-3596-42dd-a58b-f8aa744f6f5e" />
기존 AWS 무료 티어 종료 이후, 비용 제약 속에서도 Kubernetes 기반 MSA 환경과 Kafka를 직접 구축하고 운영해보기 위해 로컬 클러스터로 이전했습니다.

1. AWS EKS 비용 문제 우회: 관리형 서비스인 EKS의 유지 비용이 개인 프로젝트 수준에서는 매우 부담스러웠습니다. 이에 대한 대안으로 보유 중인 고사양 로컬 PC(Ryzen 7800X3D, 64GB RAM) 인프라를 활용하여 직접 클러스터를 프로비저닝(Vagrant)하였습니다.
2. 실무형 인프라 경험 확보: 단순한 클라우드 VM 배포를 넘어서 Kubernetes 기반 컨테이너 오케스트레이션과 Kafka 비동기 통신을 직접 구성·운영하며, MSA 환경의 트러블슈팅 경험을 쌓는 것을 주요 목표로 삼았습니다.

---

## Tech Stack
- Framework: Java 17, Spring Boot 3.x, Spring Cloud Gateway
- Database: 
  - MySQL 8.x: 개인 프로젝트 환경의 자원 한계를 고려해 단일 인스턴스를 사용하고, 서비스별 스키마 분리로 데이터 경계를 나눔
  - Redis: 일회성 인증 데이터 관리 및 분산 캐시
- Data Pipeline: Apache Kafka, Debezium (CDC), Kafka Connect
- Search Engine: Elasticsearch 8.x
- Infrastructure: Kubernetes (Vagrant Local VM), Helm, Docker, Nginx Ingress

## Microservices
| Service | Internal Port | Description |
|---|---|---|
| api-gateway | 8080 | 공통 라우팅, JWT 인증/인가 필터링, 로드밸런싱 |
| user-service | 8080 | 회원가입, 로그인, JWT 토큰 발급, 이메일 인증 |
| animal-service | 8081 | 공공데이터(APMS) 유기동물 배치 수집, 검색, 관심 등록 |
| community-service | 8082 | 커뮤니티 피드, 게시글 및 댓글 작성 |
| store-service | 8083 | 이커머스 상품 관리, 장바구니, 검색 |
| payment-service | 8084 | Toss Payments 연동 결제 처리, 주문 내역 관리 |

---

## 인프라 구축 매뉴얼 (Infrastructure Setup)
- 애플리케이션이 올라갈 Kubernetes 클러스터 하부 망(Vagrant, DB, Kafka 등) 설치 가이드는 전용 저장소인 [pawbridge-infra-k8s](https://github.com/pawbridge/pawbridge-infra-k8s)에서 확인할 수 있습니다.
- 동물 및 상품 검색 코어가 의존하는 Elasticsearch 수동 인덱스 매핑 파일 및 파이프라인 스크립트는 현재 백엔드 저장소의 infrastructure/elasticsearch/ 디렉토리에 포함되어 있습니다.

---
