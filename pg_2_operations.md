# 2. 실무 설정 및 운영 시나리오

---

## 단계별 수행 계정 요약

| 단계 | 계정 |
|------|------|
| backup 디렉토리 생성 | root |
| pg_dump / pg_basebackup / crontab | postgres |
| PITR 복구 (systemctl) | root |
| 모니터링 쿼리 | postgres |

---

## 1. 백업 시나리오

### 1-1. backup 디렉토리 생성 (root에서)

> `/backup` 은 루트 디렉토리 바로 아래에 위치한다.
> 루트 디렉토리는 root만 쓰기 권한이 있으므로 root에서 생성하고
> 실제 백업을 수행하는 postgres 유저에게 소유권을 넘긴다.

```bash
mkdir -p /backup
chown postgres:postgres /backup

# 확인
ls -la / | grep backup
```

---

### 1-2. pg_dump (논리 백업)

> Oracle의 expdp/impdp에 대응하는 논리 백업 방식이다.
> 특정 DB 또는 테이블 단위로 백업 가능하다.
> SQL 문장 형태로 저장되므로 버전이 다른 환경으로 이행할 때도 활용할 수 있다.

#### 사전 준비 — 테이블 및 데이터 생성

> 백업 실습을 위해 testdb에 테이블과 데이터를 먼저 생성한다.

```bash
psql -U postgres -d testdb
```

```sql
CREATE TABLE dept (
  deptno INT PRIMARY KEY,
  dname  VARCHAR(20),
  loc    VARCHAR(20)
);

CREATE TABLE emp (
  empno    INT PRIMARY KEY,
  ename    VARCHAR(10),
  job      VARCHAR(9),
  mgr      INT,
  hiredate DATE,
  sal      NUMERIC(7,2),
  comm     NUMERIC(7,2),
  deptno   INT REFERENCES dept(deptno)
);

INSERT INTO dept VALUES (10,'ACCOUNTING','NEW YORK');
INSERT INTO dept VALUES (20,'RESEARCH','DALLAS');
INSERT INTO dept VALUES (30,'SALES','CHICAGO');
INSERT INTO dept VALUES (40,'OPERATIONS','BOSTON');

INSERT INTO emp VALUES (7839,'KING','PRESIDENT',NULL,'1981-11-17',5000,NULL,10);
INSERT INTO emp VALUES (7698,'BLAKE','MANAGER',7839,'1981-05-01',2850,NULL,30);
INSERT INTO emp VALUES (7782,'CLARK','MANAGER',7839,'1981-05-09',2450,NULL,10);
INSERT INTO emp VALUES (7566,'JONES','MANAGER',7839,'1981-04-01',2975,NULL,20);
INSERT INTO emp VALUES (7900,'JAMES','CLERK',7698,'1981-12-11',950,NULL,30);

COMMIT;

-- 확인
SELECT COUNT(*) FROM emp;
SELECT COUNT(*) FROM dept;
\dt
EXIT;
```

**명령어 구조:**

```
pg_dump [접속 옵션] [백업 옵션] -f [저장 경로]

-U postgres      : 접속 유저
-d testdb        : 백업 대상 DB
-F c             : 포맷 (c=custom, p=plain SQL, t=tar)
-f               : 저장 파일 경로
-t emp           : 특정 테이블만 백업
```

**실행:**

```bash
# DB 전체 백업 (custom 포맷)
pg_dump -U postgres -d testdb -F c -f /backup/testdb_$(date +%Y%m%d).dump

# 특정 테이블만 백업
pg_dump -U postgres -d testdb -t emp -F c -f /backup/emp_$(date +%Y%m%d).dump

# SQL 형식으로 백업 (사람이 읽을 수 있는 형태)
pg_dump -U postgres -d testdb -F p -f /backup/testdb_$(date +%Y%m%d).sql
```

**백업 확인:**

```bash
# 백업 파일 생성 확인
ls -lh /backup/

# 백업 파일 내용 확인 (custom 포맷)
# TOC(Table of Contents) 목록으로 백업에 포함된 오브젝트 확인
# Selected TOC Entries에 테이블/데이터 목록이 있어야 정상
pg_restore --list /backup/testdb_$(date +%Y%m%d).dump

# 백업 파일 내용 확인 (SQL 포맷)
# CREATE TABLE, INSERT 문이 포함되어 있어야 정상
head -50 /backup/testdb_$(date +%Y%m%d).sql
```

<img width="1237" height="498" alt="image" src="https://github.com/user-attachments/assets/0fe34080-9196-4f4d-ac1c-eb1923b24877" />

<img width="1259" height="848" alt="image" src="https://github.com/user-attachments/assets/08a4eb88-211e-4390-95de-e4e8b8fd0a11" />

**복구:**

> dump 파일 복구 시 테이블이 이미 있으면 오류가 발생할 수 있다.
> 테이블을 먼저 DROP한 후 복구하면 정상적으로 진행된다.

```bash
psql -U postgres -d testdb
```

```sql
-- 장애 상황 재현 (테이블 삭제)
DROP TABLE emp;
DROP TABLE dept;

-- 삭제 확인
\dt
EXIT;
```

```bash
# dump 파일로 복구
pg_restore -U postgres -d testdb -F c -v /backup/testdb_20260608.dump

# 복구 결과 확인
psql -U postgres -d testdb -c "\dt"
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM emp;"
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM dept;"
```

**SQL 파일로 복구:**

```bash
psql -U postgres -d testdb
```

```sql
-- 장애 상황 재현
DROP TABLE emp;
DROP TABLE dept;
\dt
EXIT;
```

```bash
psql -U postgres -d testdb -f /backup/testdb_20260608.sql

# 복구 결과 확인
psql -U postgres -d testdb -c "\dt"
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM emp;"
```

**custom 포맷(.dump) vs SQL 포맷(.sql) 비교:**

| 항목 | .dump (custom) | .sql (plain) |
|------|---------------|-------------|
| 복구 명령어 | pg_restore | psql |
| 선택적 복구 | 가능 (특정 테이블만) | 불가 (전체만) |
| 파일 크기 | 작음 (압축) | 큼 |
| 사람이 읽기 | 불가 | 가능 |
| 실무 선호 | 높음 | 내용 확인용 |

---

### 1-3. pg_basebackup (물리 백업)

> Oracle의 RMAN 백업에 대응하는 물리 백업 방식이다.
> 데이터 디렉토리 전체를 그대로 복사하며 Streaming Replication 구성 시에도 사용한다.
> pg_dump와 달리 DB 서비스 중단 없이 온라인 백업이 가능하다.

**명령어 구조:**

```
pg_basebackup [접속 옵션] -D [저장 경로] [백업 옵션]

-h localhost     : 접속 호스트
-U postgres      : 접속 유저
-D               : 백업 저장 경로 (빈 디렉토리여야 함)
-Fp              : Plain format (파일 그대로 저장)
-Xs              : WAL 파일 스트리밍으로 포함 (PITR 복구에 필요)
-P               : 진행률 표시
-R               : Standby 설정 파일 자동 생성 (복제 구성 시 사용)
```

**실행:**

```bash
pg_basebackup -h localhost -U postgres \
  -D /backup/basebackup_$(date +%Y%m%d) \
  -Fp -Xs -P
```

**백업 확인:**

```bash
# 백업 디렉토리 구조 확인
# 정상이라면 PG_VERSION / postgresql.conf / base/ / pg_wal/ 등이 있어야 함
ls -lh /backup/basebackup_$(date +%Y%m%d)/

# 백업 크기 확인
# 너무 작으면 일부만 백업됐을 가능성, 0이면 백업 실패
du -sh /backup/basebackup_$(date +%Y%m%d)/

# WAL 파일 포함 여부 확인
# -Xs 옵션으로 WAL이 포함됐는지 확인
# WAL이 없으면 PITR 복구 불가
ls -lh /backup/basebackup_$(date +%Y%m%d)/pg_wal/
```

---

### 1-4. 백업 자동화 (crontab)

> crontab은 Linux에서 명령어를 원하는 시간에 자동 실행하는 스케줄러다.
> Oracle의 DBMS_SCHEDULER와 동일한 개념이며, DB가 내려가도 OS 레벨에서 실행된다.
> 수동 백업은 누락 위험이 있으므로 crontab으로 자동화하는 것이 실무 표준이다.

**crontab 문법:**

```
분  시  일  월  요일  명령어
0   2   *   *    *   명령어  →  매일 새벽 2시 정각 실행
*   =  모든 값 (매분/매시/매일)

0=일 1=월 2=화 3=수 4=목 5=금 6=토

-- 예시

# 매주 금요일 오후 9시 pg_dump 백업
0 21 * * 5 pg_dump -U postgres -d testdb -F c -f /backup/testdb_$(date +\%Y\%m\%d).dump

# 매주 화,금 오전 12시(자정) 2달 이상 된 백업 파일 자동 삭제
# -mtime +60 : 수정된 지 60일 이상 된 파일 대상
0 0 * * 2,5 find /backup -name "*.dump" -mtime +60 -delete
```

```bash
crontab -e
```

아래 내용을 입력한다.

```text
# 매일 새벽 2시 pg_dump 백업
# \% : crontab에서 %는 특수문자이므로 \로 이스케이프 필요
0 2 * * * pg_dump -U postgres -d testdb -F c -f /backup/testdb_$(date +\%Y\%m\%d).dump

# 매일 새벽 3시 30일 이상 된 백업 파일 자동 삭제
# -mtime +30 : 수정된 지 30일 이상 된 파일 대상
0 3 * * * find /backup -name "*.dump" -mtime +30 -delete
```

**등록 확인:**

```bash
# 등록된 crontab 목록 확인
crontab -l

# cron 서비스 상태 확인
systemctl status crond
```

<img width="1254" height="315" alt="image" src="https://github.com/user-attachments/assets/bfff8f2f-8c7c-4f20-b428-5617813b6cb7" />

---

## 2. PITR (Point In Time Recovery) 시나리오

> Archive 모드가 활성화된 상태에서 특정 시점으로 복구하는 방법이다.
> Oracle의 Archive Log 기반 불완전 복구와 동일한 개념이다.
> 실수로 테이블을 삭제했거나 잘못된 데이터가 입력된 시점 이전으로 복구할 때 사용한다.

**PITR 복구 흐름:**

```
베이스 백업 복원 (데이터 디렉토리 덮어씌움)
        ↓
WAL을 순서대로 재실행 (archive 디렉토리에서 읽어옴)
        ↓
recovery_target_time에 지정한 시점에서 중지
        ↓
DB 오픈 (promote)
```

---

### 2-1. 사전 준비 확인

```bash
# archive 디렉토리 확인
ls -la /var/lib/pgsql/15/archive/
```

```bash
psql -U postgres
```

```sql
-- archive 모드 활성화 확인
-- on 이어야 WAL 파일이 archive 디렉토리에 복사된다
SHOW archive_mode;

-- archive 명령어 확인
-- WAL 파일을 어디로 복사하는지 경로 확인
SHOW archive_command;

-- WAL 레벨 확인
-- replica 이상이어야 Streaming Replication 및 PITR 사용 가능
SHOW wal_level;

-- 아카이브 상태 확인
-- archived_count : 정상적으로 archive된 WAL 파일 수 (1 이상이면 정상)
-- failed_count   : archive 실패 횟수 (0 이어야 정상)
-- last_archived_wal  : 마지막으로 archive된 WAL 파일명
-- last_archived_time : 마지막 archive 시각
SELECT archived_count, failed_count,
       last_archived_wal, last_archived_time
FROM pg_stat_archiver;

EXIT;
```

<img width="1459" height="242" alt="image" src="https://github.com/user-attachments/assets/6cb6cf1a-9771-4b05-a1e3-ba8ed9309090" />

> `archived_count >= 1` 이고 `failed_count = 0` 이면 정상이다.
>
> `failed_count > 0` 이면 archive_command 경로 또는 권한을 확인한다.

---

### 2-2. 베이스 백업 수행

> PITR 복구의 시작점이 되는 물리 백업이다.
> 이 백업 이후 발생한 변경사항은 archive된 WAL로 재실행하여 복구한다.
> pg_basebackup은 빈 디렉토리에만 백업 가능하다. 이미 존재하면 다른 이름으로 지정한다.

```bash
pg_basebackup -h localhost -U postgres \
  -D /backup/basebackup_pitr \
  -Fp -Xs -P

# 백업 완료 시각 기록
date
```

<img width="1255" height="164" alt="image" src="https://github.com/user-attachments/assets/9712ed24-7218-4d9c-9caf-94953773d830" />

---

### 2-3. PITR 복구 시나리오

```bash
# 1. 복구 목표 시점 기록
date
# 예: 2026-06-08 11:32:00 ← 이 시점으로 복구할 것

# 2. 장애 상황 재현 (실수로 테이블 삭제)
psql -U postgres -d testdb -c "DROP TABLE emp;"
psql -U postgres -d testdb -c "\dt"

# 3. 서비스 중지 (root에서)
# 복구 중 DB가 데이터 파일을 읽고 쓰면 충돌이 발생할 수 있으므로 반드시 중지
systemctl stop postgresql-15

# 4. 현재 데이터 디렉토리 백업
# 복구 실패 시 롤백을 위한 안전장치
cp -r /var/lib/pgsql/15/data /var/lib/pgsql/15/data_broken

# 5. 베이스 백업 복원
# 기존 데이터와 백업 파일이 섞이면 충돌 발생하므로 완전히 비우고 복사
rm -rf /var/lib/pgsql/15/data/*
cp -r /backup/basebackup_pitr/* /var/lib/pgsql/15/data/

# cp 실행 계정이 root라서 소유자가 root로 복사됨
# PostgreSQL은 postgres 유저로 실행되므로 반드시 소유자 변경 필요
chown -R postgres:postgres /var/lib/pgsql/15/data/

# 6. 복구 설정 추가 (postgresql.conf에)
# restore_command : archive에서 WAL 파일을 가져오는 명령어
# recovery_target_time : 이 시점까지만 WAL을 재실행하고 중지
# recovery_target_action : 복구 완료 후 promote(Primary로 전환)
cat >> /var/lib/pgsql/15/data/postgresql.conf << 'EOF'
restore_command = 'cp /var/lib/pgsql/15/archive/%f %p'
recovery_target_time = '2026-06-08 11:32:00'
recovery_target_action = 'promote'
EOF

# standby.signal 파일 생성 (복구 모드 진입용)
# 이 파일이 있으면 PostgreSQL이 복구 모드로 시작됨
touch /var/lib/pgsql/15/data/standby.signal
chown postgres:postgres /var/lib/pgsql/15/data/standby.signal

# 7. 서비스 시작 (복구 모드로 자동 진입)
systemctl start postgresql-15

# 8. 복구 진행 로그 확인
tail -50 /var/lib/pgsql/15/data/log/$(ls -t /var/lib/pgsql/15/data/log/ | head -1)
```

**복구 완료 확인:**

```bash
psql -U postgres -d testdb -c "\dt"
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM emp;"

# false = Primary로 정상 전환 완료
psql -U postgres -c "SELECT pg_is_in_recovery();"
```

<img width="1230" height="481" alt="image" src="https://github.com/user-attachments/assets/aaa44dc2-0053-42b0-8528-21c5d02e72bb" />

---

## 3. 모니터링 쿼리

> PostgreSQL은 `pg_stat_` 로 시작하는 시스템 뷰로 DB 상태를 확인한다.
> Oracle의 `v$` 동적 성능 뷰와 동일한 개념이다.
> 일반 유저도 대부분의 뷰를 조회할 수 있어 Oracle보다 접근이 자유롭다.

---

### 3-1. 인스턴스 / 세션 상태

```bash
psql -U postgres
```

```sql
-- 현재 접속 세션 전체 (Oracle의 v$session)
SELECT pid, usename, application_name, client_addr,
       state, query_start, query
FROM pg_stat_activity
ORDER BY query_start;

-- 실행 중인 쿼리만 확인
SELECT pid, now() - query_start AS duration,
       usename, state, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- 유휴 세션 확인
SELECT pid, usename, state, query_start
FROM pg_stat_activity
WHERE state = 'idle'
ORDER BY query_start;

-- 세션 강제 종료 (Oracle의 ALTER SYSTEM KILL SESSION)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid = 12345;

-- 특정 유저 세션 모두 종료
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'scott';
```

---

### 3-2. DB / 테이블 크기 확인

```sql
-- DB 크기 전체 목록 (Oracle의 dba_segments)
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- 현재 접속 DB 크기
SELECT pg_size_pretty(pg_database_size(current_database())) AS current_db_size;

-- 테이블 크기 상위 10개
-- pg_relation_size : 테이블 데이터만의 크기
-- pg_total_relation_size : 테이블 + 인덱스 + TOAST 전체 크기
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

<img width="1458" height="702" alt="image" src="https://github.com/user-attachments/assets/34298a63-7c58-481b-9e80-91883afc920b" />

---

### 3-3. 슬로우 쿼리 확인

> 어떤 SQL이 느린지 찾아내기 위해 `pg_stat_statements` 확장을 사용한다.
> 평균 실행 시간 / 호출 횟수 / 총 실행 시간을 한눈에 확인하여
> 튜닝이 필요한 쿼리를 빠르게 파악할 수 있다.
> Oracle의 `v$sql`과 동일한 역할이다.

**Extension이란:**

> PostgreSQL에 기능을 추가하는 플러그인이다.
> `CREATE EXTENSION`으로 설치하고 `DROP EXTENSION`으로 제거한다.
> Oracle의 옵션 패키지와 유사한 개념이며 대부분 무료로 제공된다.

**pg_stat_statements 활성화 (최초 1회, root에서):**

> `pg_stat_statements`는 `postgresql15-contrib` 패키지에 포함되어 있다.
> 설치되어 있지 않으면 아래 명령어로 먼저 설치한다.

```bash
yum install -y postgresql15-contrib
```

`postgresql.conf`에 설정을 추가한다.

```bash
vi /var/lib/pgsql/15/data/postgresql.conf
```

```text
# DB 시작 시 pg_stat_statements 라이브러리를 미리 로드
# 재시작 후 CREATE EXTENSION으로 활성화 가능
shared_preload_libraries = 'pg_stat_statements'
```

```bash
systemctl restart postgresql-15
```

```bash
psql -U postgres
```

```sql
-- 확장 설치
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 설치 확인
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- 평균 실행 시간 상위 20개
-- 한 번 실행할 때 가장 오래 걸리는 쿼리 확인 → 튜닝 우선순위
SELECT query,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- 호출 횟수 상위 20개
-- 가장 자주 실행되는 쿼리 확인 → 최적화 시 효과가 가장 큰 쿼리
SELECT query, calls, round(mean_exec_time::numeric, 2) AS mean_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;

-- 통계 초기화 (튜닝 전/후 비교 시 사용)
SELECT pg_stat_reset();
SELECT pg_stat_statements_reset();
```

**주요 Extension 목록:**

| Extension | 용도 | Oracle 대응 |
|-----------|------|-----------|
| `pg_stat_statements` | SQL 실행 통계 수집 | `v$sql` |
| `pg_buffercache` | Buffer Pool 상태 확인 | `v$buffer_pool` |
| `pgcrypto` | 암호화 함수 | `DBMS_CRYPTO` |
| `uuid-ossp` | UUID 생성 | `SYS_GUID()` |
| `postgres_fdw` | 외부 DB 연결 | Database Link |

**주요 시스템 뷰 정리:**

| 뷰 | 용도 | Oracle 대응 |
|----|------|-----------|
| `pg_stat_activity` | 현재 세션/쿼리 상태 | `v$session` |
| `pg_stat_statements` | SQL 누적 실행 통계 | `v$sql` |
| `pg_stat_archiver` | WAL 아카이브 상태 | `v$archive_dest_status` |
| `pg_stat_replication` | 복제 상태 | `v$managed_standby` |
| `pg_locks` | Lock 현황 | `v$lock` |
| `pg_tables` | 테이블 목록 | `user_tables` |
| `pg_database` | DB 목록 | `v$database` |
| `pg_user` | 유저 목록 | `dba_users` |

---

### 3-4. Lock 확인

> Lock은 여러 세션이 동일한 데이터를 동시에 변경할 때 발생하는 대기 상태다.
> Lock이 오래 지속되면 서비스 응답 지연이 발생하므로 원인 세션을 찾아 종료해야 한다.

```sql
-- Lock 현황 전체 (Oracle의 v$lock)
SELECT pid,
       relation::regclass AS table_name,
       mode,
       granted
FROM pg_locks
WHERE relation IS NOT NULL
ORDER BY pid;

-- Lock 대기 중인 쿼리
SELECT pid, usename, query,
       wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY pid;

-- Lock 블로킹 관계 확인 (누가 누구를 막고 있는지)
SELECT blocked.pid AS blocked_pid,
       blocked.usename AS blocked_user,
       blocking.pid AS blocking_pid,
       blocking.usename AS blocking_user,
       blocked.query AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event IS NOT NULL;
```
<img width="1445" height="294" alt="image" src="https://github.com/user-attachments/assets/4a33fbbf-31be-4981-a90d-6eb50c076fbd" />

- 지금 결과처럼 granted = t → 정상 (Lock 획득 완료)
- 문제가 되는 건 granted = f → Lock 대기 중 (다른 세션이 막고 있음)
---

### 3-5. 복제 상태 확인 (Streaming Replication 구성 후)

```sql
-- Primary에서 Standby 상태 확인 (Oracle의 v$managed_standby)
SELECT client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_size_pretty(sent_lsn - replay_lsn) AS replication_lag
FROM pg_stat_replication;

-- 현재 노드가 Primary인지 Standby인지 확인
-- false = Primary / true = Standby
SELECT pg_is_in_recovery();
```

---

## 4. WAL (Write-Ahead Log)

> Oracle의 Redo Log + Archive Log에 대응하는 개념이다.
> 데이터를 실제 파일에 쓰기 전에 WAL에 먼저 기록하여 장애 시 복구에 사용한다.
> Streaming Replication에서는 Primary의 WAL을 Standby로 전송하여 복제에 활용한다.

| 항목 | Oracle | PostgreSQL |
|------|--------|-----------|
| 온라인 변경 기록 | Redo Log | WAL (pg_wal 디렉토리) |
| 아카이브 | Archive Log | Archive WAL |
| 복구 사용 | 예 | 예 |
| 복제 사용 | Data Guard | Streaming Replication |

```bash
# WAL 파일 위치 및 목록
ls -lh /var/lib/pgsql/15/data/pg_wal/

# Archive 파일 위치 및 목록
ls -lh /var/lib/pgsql/15/archive/

# WAL 파일 개수 확인
ls /var/lib/pgsql/15/data/pg_wal/ | wc -l
```

```sql
-- WAL 아카이브 상태 확인
SELECT archived_count,
       failed_count,
       last_archived_wal,
       last_archived_time,
       last_failed_wal,
       last_failed_time
FROM pg_stat_archiver;

-- 현재 WAL 위치 확인
SELECT pg_current_wal_lsn();

-- WAL 파일명 확인
SELECT pg_walfile_name(pg_current_wal_lsn());
```

---

## 5. 자주 쓰는 psql 명령어

| 명령어 | 설명 | Oracle 대응 |
|--------|------|-----------|
| `\l` | DB 목록 | SELECT name FROM v$database |
| `\c testdb` | DB 전환 | CONNECT |
| `\dt` | 테이블 목록 | SELECT table_name FROM user_tables |
| `\dt *.*` | 전체 스키마 테이블 목록 | SELECT table_name FROM all_tables |
| `\d emp` | 테이블 구조 | DESC emp |
| `\di` | 인덱스 목록 | SELECT index_name FROM user_indexes |
| `\du` | 유저 목록 | SELECT username FROM dba_users |
| `\dn` | 스키마 목록 | SELECT username FROM dba_users |
| `\df` | 함수 목록 | SELECT object_name FROM user_objects |
| `\dv` | 뷰 목록 | SELECT view_name FROM user_views |
| `\timing` | 쿼리 실행 시간 표시 | SET TIMING ON |
| `\x` | 세로 출력 모드 전환 | - |
| `\e` | 외부 편집기로 쿼리 편집 | - |
| `\i 파일명` | SQL 파일 실행 | @파일명 |
| `\o 파일명` | 결과를 파일로 저장 | SPOOL 파일명 |
| `\q` | 종료 | EXIT |
| `\?` | psql 명령어 도움말 | - |
| `\h SELECT` | SQL 문법 도움말 | - |

---

## 6. .pgpass 파일 (비밀번호 자동 입력)

> 매번 비밀번호를 입력하지 않도록 설정한다.
> Oracle의 wallet과 유사한 개념이다.
> 반드시 600 권한을 설정해야 하며, 권한이 600이 아니면 파일이 무시된다.

```bash
vi ~/.pgpass
```

```text
# 형식: hostname:port:database:username:password
# *는 모든 값에 매칭
172.31.0.250:5432:*:postgres:oracle
172.31.0.250:5432:*:scott:tiger
localhost:5432:*:postgres:oracle
```

```bash
chmod 600 ~/.pgpass
ls -la ~/.pgpass
```

비밀번호 없이 접속 확인:

```bash
psql -U postgres -h 172.31.0.250
psql -U scott -h 172.31.0.250 -d testdb
```
