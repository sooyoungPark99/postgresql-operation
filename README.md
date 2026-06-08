# PostgreSQL 15 운영 및 PgPool-II 구성 가이드

Oracle Linux 7.7 환경에서 PostgreSQL 15 실무 운영 시나리오와 PgPool-II를 구성하는 가이드입니다.

백업/복구, 모니터링, Streaming Replication, PgPool-II 까지 함께 구성합니다.

---

### 환경 구성

| 항목 | 내용 |
|------|------|
| OS | Oracle Linux 7.7 |
| DB 버전 | PostgreSQL 15 |
| 미들웨어 | PgPool-II 4.4 |
| 가상화 | VirtualBox 7.0.18 |

---

### 기존 저장소와 차이점

| 항목 | [postgresql-install](https://github.com/sooyoungPark99/postgresql-install) | 이 저장소 |
|------|-----------|---------|
| 목적 | PostgreSQL + Oracle 19c 환경 구축 | 실무 운영 + HA 구성 |
| 주요 내용 | 리눅스 설치 / DBeaver 연결 / Oracle 소프트웨어 | 백업/복구 / 모니터링 / 복제 / PgPool-II |
| 대상 | 환경 세팅 | DBE 실무 운영 |

---

### 서버 구성

| 역할 | 호스트명 | IP |
|------|---------|----|
| Single / Primary + PgPool-II | pg-primary | 172.31.0.260 |
| Standby (PgPool 구성 시) | pg-standby | 172.31.0.261 |

---

### 구성 순서

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [PostgreSQL 15 설치](./1.%20PostgreSQL%2015%20설치.md) | PGDG repo / 설치 / 초기 설정 / alias / Oracle 비교 |
| 2 | [실무 설정 및 운영](./2.%20실무%20설정%20및%20운영.md) | 백업/복구 / PITR / 모니터링 쿼리 / WAL |
| 3 | [PgPool-II 구성](./3.%20PgPool-II%20구성.md) | Streaming Replication / PgPool-II 설치 / 부하 분산 |

---

### 주요 특징

- PostgreSQL 15 PGDG repo 기반 설치 및 초기 설정
- Oracle DBA 관점의 PostgreSQL 개념 비교 정리
- 실무 alias 및 .pgpass 설정으로 편리한 운영 환경 구성
- pg_dump / pg_basebackup 백업 및 PITR 복구 시나리오
- WAL / Archive Log 개념 및 관리
- Streaming Replication + PgPool-II 부하 분산 구성
- 실제 구성 중 발생한 오류 및 해결 방법 포함
