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
pg_restore --list /backup/testdb_$(date +%Y%m%d).dump

# 백업 파일 내용 확인 (SQL 포맷)
head -50 /backup/testdb_$(date +%Y%m%d).sql
```
<img width="1237" height="498" alt="image" src="https://github.com/user-attachments/assets/0fe34080-9196-4f4d-ac1c-eb1923b24877" />

<img width="1259" height="848" alt="image" src="https://github.com/user-attachments/assets/08a4eb88-211e-4390-95de-e4e8b8fd0a11" />

**복구:**

```bash
# testdb 접속
psql -U postgres -d testdb
```

```sql
-- 장애 상황 재현 (테이블 삭제)
DROP TABLE emp;
DROP TABLE dept;

-- 삭제 확인
\dt
```

```bash
# dump 파일로 복구
pg_restore -U postgres -d testdb -F c -v /backup/testdb_20260608.dump

# 복구 결과 확인
psql -U postgres -d testdb -c "\dt"
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM emp;"
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM dept;"
```

> dump 파일 복구 시 테이블이 이미 있으면 오류가 발생할 수 있다.
> 테이블을 먼저 DROP한 후 복구하면 정상적으로 진행된다.

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

---

### 1-3. pg_basebackup (물리 백업)

> Oracle의 RMAN 백업에 대응하는 물리 백업 방식이다.
> 데이터 디렉토리 전체를 그대로 복사하며 Streaming Replication 구성 시에도 사용한다.

**명령어 구조:**

```
pg_basebackup [접속 옵션] -D [저장 경로] [백업 옵션]

-h localhost     : 접속 호스트
-U postgres      : 접속 유저
-D               : 백업 저장 경로
-Fp              : Plain format (파일 그대로 저장)
-Xs              : WAL 파일 스트리밍으로 포함
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
ls -lh /backup/basebackup_$(date +%Y%m%d)/

-- 백업 디렉토리 안에 어떤 파일들이 생성됐는지 확인
-- 정상이라면 아래 파일들이 있어야 함:
PG_VERSION        ← PostgreSQL 버전 파일
postgresql.conf   ← 설정 파일
pg_hba.conf       ← 인증 설정 파일
base/             ← 실제 데이터 디렉토리
pg_wal/           ← WAL 파일
global/           ← 시스템 카탈로그

# 백업 크기 확인
du -sh /backup/basebackup_$(date +%Y%m%d)/

-- 실무에서 중요한 이유:
백업이 너무 작으면 → 일부만 백업됐을 가능성
백업이 0이면 → 백업 실패
디스크 공간 관리를 위해 크기 파악 필요

# WAL 파일 포함 여부 확인
ls -lh /backup/basebackup_$(date +%Y%m%d)/pg_wal/
```
-- Xs 옵션으로 WAL 파일이 실제로 포함됐는지 확인

WAL이 없으면 → PITR 복구 불가
WAL이 있어야 → 특정 시점으로 복구 가능
---

### 1-4. 백업 자동화 (crontab)

> crontab은 Linux에서 명령어를 원하는 시간에 자동 실행하는 스케줄러다.
> Oracle의 DBMS_SCHEDULER와 동일한 개념이며, DB가 내려가도 OS 레벨에서 실행된다.

**crontab 문법:**

```
분  시  일  월  요일  명령어
0   2   *   *    *   명령어  →  매일 새벽 2시 정각 실행
*   =  모든 값 (매분/매시/매일)
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

> `archived_count >= 1` 이고 `failed_count = 0` 이면 정상이다.
>
> `failed_count > 0` 이면 archive_command 경로 또는 권한을 확인한다.

### 2-2. 베이스 백업 먼저 수행

```bash
pg_basebackup -h localhost -U postgres \
  -D /backup/basebackup_pitr \
  -Fp -Xs -P

# 백업 완료 시각 기록
date
```
<img width="1255" height="164" alt="image" src="https://github.com/user-attachments/assets/9712ed24-7218-4d9c-9caf-94953773d830" />

### 2-3. PITR 복구 시나리오

```bash
# 1. 복구 목표 시점 기록
date
# 예: 2026. 06. 08. (월) 11:32:57 KST ← 이 시점으로 복구할 것

# 2. 실수로 테이블 삭제 (장애 상황 재현)
psql -U postgres -d testdb -c "DROP TABLE emp;"

# 삭제 확인
psql -U postgres -d testdb -c "\dt"

# 3. 서비스 중지 (root에서)
systemctl stop postgresql-15

# 4. 현재 데이터 디렉토리 백업
cp -r /var/lib/pgsql/15/data /var/lib/pgsql/15/data_broken

# 5. 베이스 백업 복원
1. rm -rf /var/lib/pgsql/15/data/*
   현재 데이터 디렉토리 완전히 비움
   (디렉토리 자체는 유지, 안의 내용만 삭제)

2. cp -r /backup/basebackup_pitr/* /var/lib/pgsql/15/data/
   백업 파일을 비어있는 디렉토리에 복사
   (백업 시점의 깨끗한 상태로 복원)
        ↓
3. chown -R postgres:postgres /var/lib/pgsql/15/data/
   복사한 파일의 소유자를 postgres로 변경
   (cp 명령어 실행 계정이 root라서
    소유자가 root로 복사됨 → postgres로 변경 필요)

# 6. 복구 설정 추가 (postgresql.conf에)
cat >> /var/lib/pgsql/15/data/postgresql.conf << 'EOF'
restore_command = 'cp /var/lib/pgsql/15/archive/%f %p'
recovery_target_time = '2026-06-08 10:30:00'
recovery_target_action = 'promote'
EOF

# standby.signal 파일 생성 (복구 모드 진입용)
touch /var/lib/pgsql/15/data/standby.signal
chown postgres:postgres /var/lib/pgsql/15/data/standby.signal

# 7. 서비스 시작 (복구 모드)
systemctl start postgresql-15

# 8. 복구 진행 로그 확인
tail -50 /var/lib/pgsql/15/data/log/$(ls -t /var/lib/pgsql/15/data/log/ | head -1)
```

**복구 완료 확인:**

```bash
# 복구 완료 후 테이블 복원 확인
psql -U postgres -d testdb -c "\dt"
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM emp;"

# 복구 모드 종료 확인 (pg_is_in_recovery = false 이면 완료)
psql -U postgres -c "SELECT pg_is_in_recovery();"
```

---

## 3. 모니터링 쿼리

### 3-1. 인스턴스 / 세션 상태

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
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

> `pg_relation_size`: 테이블 데이터만의 크기
> `pg_total_relation_size`: 테이블 + 인덱스 + TOAST 전체 크기

---

### 3-3. 슬로우 쿼리 확인

pg_stat_statements 확장 활성화 (최초 1회, root에서):

```bash
vi /var/lib/pgsql/15/data/postgresql.conf
# 아래 추가
# shared_preload_libraries = 'pg_stat_statements'

systemctl restart postgresql-15
```

```sql
-- 확장 설치
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 평균 실행 시간 상위 20개
SELECT query,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- 호출 횟수 상위 20개
SELECT query, calls, round(mean_exec_time::numeric, 2) AS mean_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;

-- 통계 초기화
SELECT pg_stat_reset();
SELECT pg_stat_statements_reset();
```

---

### 3-4. Lock 확인

```sql
-- Lock 현황 전체 (Oracle의 v$lock)
SELECT pid,
       relation::regclass AS table_name,
       mode,
       granted,
       transactionid
FROM pg_locks
WHERE relation IS NOT NULL
ORDER BY pid;

-- Lock 대기 중인 쿼리
SELECT pid, usename, query,
       wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY pid;

-- Lock 블로킹 관계 확인
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
SELECT pg_is_in_recovery();
-- false = Primary
-- true  = Standby
```

---

## 4. WAL (Write-Ahead Log)

> Oracle의 Redo Log + Archive Log에 대응하는 개념이다.

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

```bash
vi ~/.pgpass
```

```
# 형식: hostname:port:database:username:password
# *는 모든 값에 매칭
172.31.0.250:5432:*:postgres:oracle
172.31.0.250:5432:*:scott:tiger
localhost:5432:*:postgres:oracle
```

```bash
# 반드시 600 권한 설정 (본인만 읽기 가능)
# 권한이 600이 아니면 .pgpass 파일이 무시된다
chmod 600 ~/.pgpass

# 확인
ls -la ~/.pgpass
```

비밀번호 없이 접속 확인:

```bash
psql -U postgres -h 172.31.0.250
psql -U scott -h 172.31.0.250 -d testdb
```
