# 2. 실무 설정 및 운영 시나리오

---

## 1. 백업 시나리오

### 1-1. pg_dump (논리 백업)

> Oracle의 expdp/impdp에 대응하는 논리 백업 방식이다.
> 특정 DB 또는 테이블 단위로 백업 가능하다.

```bash
# DB 전체 백업
pg_dump -U postgres -d testdb -F c -f /backup/testdb_$(date +%Y%m%d).dump

# 특정 테이블만 백업
pg_dump -U postgres -d testdb -t emp -F c -f /backup/emp_$(date +%Y%m%d).dump

# SQL 형식으로 백업
pg_dump -U postgres -d testdb -F p -f /backup/testdb_$(date +%Y%m%d).sql
```

복구:

```bash
# dump 파일로 복구
pg_restore -U postgres -d testdb -F c /backup/testdb_20260101.dump

# SQL 파일로 복구
psql -U postgres -d testdb -f /backup/testdb_20260101.sql
```

---

### 1-2. pg_basebackup (물리 백업)

> Oracle의 RMAN 백업에 대응하는 물리 백업 방식이다.
> Streaming Replication 구성 시에도 사용한다.

```bash
# 물리 전체 백업
pg_basebackup -U postgres -D /backup/basebackup_$(date +%Y%m%d) -Fp -Xs -P
```

| 옵션 | 설명 |
|------|------|
| -D | 백업 저장 경로 |
| -Fp | Plain format (파일 그대로) |
| -Xs | WAL 파일 포함 |
| -P | 진행률 표시 |

---

### 1-3. 백업 자동화 (crontab)

```bash
crontab -e
```

```
# 매일 새벽 2시 pg_dump 백업
0 2 * * * pg_dump -U postgres -d testdb -F c -f /backup/testdb_$(date +\%Y\%m\%d).dump

# 30일 이상 된 백업 파일 자동 삭제
0 3 * * * find /backup -name "*.dump" -mtime +30 -delete
```

---

## 2. PITR (Point In Time Recovery) 시나리오

> Archive 모드가 활성화된 상태에서 특정 시점으로 복구하는 방법이다.
> Oracle의 Archive Log 기반 복구와 동일한 개념이다.

### 2-1. 사전 준비 (1번 가이드에서 설정 완료)

```
postgresql.conf
  wal_level = replica
  archive_mode = on
  archive_command = 'cp %p /var/lib/pgsql/15/archive/%f'
```

### 2-2. 복구 시나리오

```bash
# 1. 현재 시각 기록 (복구 목표 시점)
date
# 2026-06-05 14:30:00

# 2. 실수로 테이블 삭제
psql -U postgres -d testdb -c "DROP TABLE emp;"

# 3. 서비스 중지
systemctl stop postgresql-15

# 4. 데이터 디렉토리 백업
cp -r /var/lib/pgsql/15/data /var/lib/pgsql/15/data_bak

# 5. 베이스 백업 복원
rm -rf /var/lib/pgsql/15/data/*
cp -r /backup/basebackup/* /var/lib/pgsql/15/data/

# 6. 복구 설정 파일 생성
cat > /var/lib/pgsql/15/data/recovery.conf << 'RECOVEOF'
restore_command = 'cp /var/lib/pgsql/15/archive/%f %p'
recovery_target_time = '2026-06-05 14:30:00'
recovery_target_action = 'promote'
RECOVEOF

# 7. 서비스 시작 (복구 모드)
systemctl start postgresql-15

# 8. 복구 완료 후 확인
psql -U postgres -d testdb -c "SELECT COUNT(*) FROM emp;"
```

---

## 3. 모니터링 쿼리

### 3-1. 인스턴스 / 세션 상태

```sql
-- 현재 접속 세션 확인 (Oracle의 v$session)
SELECT pid, usename, application_name, client_addr,
       state, query_start, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- 실행 중인 쿼리 확인
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- 세션 강제 종료 (Oracle의 ALTER SYSTEM KILL SESSION)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid = 12345;
```

---

### 3-2. DB / 테이블 크기 확인

```sql
-- DB 크기 (Oracle의 dba_segments)
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- 테이블 크기 상위 10개
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

---

### 3-3. 슬로우 쿼리 확인

```sql
-- 누적 실행 통계 (pg_stat_statements 확장 필요)
SELECT query,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

pg_stat_statements 활성화:

```bash
vi /var/lib/pgsql/15/data/postgresql.conf
# 아래 추가
shared_preload_libraries = 'pg_stat_statements'
```

```sql
CREATE EXTENSION pg_stat_statements;
```

---

### 3-4. Lock 확인

```sql
-- Lock 현황 (Oracle의 v$lock)
SELECT pid, relation::regclass, mode, granted
FROM pg_locks
WHERE relation IS NOT NULL;

-- Lock 대기 중인 쿼리
SELECT pid, usename, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

---

### 3-5. 복제 상태 확인 (Streaming Replication 구성 후)

```sql
-- Primary에서 Standby 상태 확인 (Oracle의 v$managed_standby)
SELECT client_addr, state, sent_lsn, write_lsn,
       flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS replication_lag
FROM pg_stat_replication;
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
# WAL 파일 위치
ls -la /var/lib/pgsql/15/data/pg_wal/

# Archive 파일 위치
ls -la /var/lib/pgsql/15/archive/

# WAL 아카이브 상태 확인
psql -U postgres -c "SELECT * FROM pg_stat_archiver;"
```

---

## 5. 자주 쓰는 psql 명령어

| 명령어 | 설명 | Oracle 대응 |
|--------|------|-----------|
| `\l` | DB 목록 | SELECT name FROM v$database |
| `\c testdb` | DB 전환 | ALTER SESSION (개념 다름) |
| `\dt` | 테이블 목록 | SELECT table_name FROM user_tables |
| `\d emp` | 테이블 구조 | DESC emp |
| `\du` | 유저 목록 | SELECT username FROM dba_users |
| `\dn` | 스키마 목록 | SELECT username FROM dba_users |
| `\di` | 인덱스 목록 | SELECT index_name FROM user_indexes |
| `\timing` | 쿼리 실행 시간 표시 | SET TIMING ON |
| `\e` | 외부 편집기로 쿼리 편집 | - |
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
# hostname:port:database:username:password
172.31.0.250:5432:*:postgres:oracle
172.31.0.250:5432:*:scott:tiger
```

```bash
chmod 600 ~/.pgpass
```

이후 비밀번호 없이 접속 가능하다:

```bash
psql -U postgres -h 172.31.0.250
```
