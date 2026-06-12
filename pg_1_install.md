# 1. PostgreSQL 15 설치

---

## 단계별 수행 계정 요약

| 단계 | 계정 |
|------|------|
| 1~6 (설치 및 초기화) | root |
| 7 (alias 설정) | oracle 또는 일반 계정 |
| 8 (접속 확인) | root 또는 postgres |

---

## 1. 사전 확인

```bash
# 기존 PostgreSQL 설치 여부 확인
rpm -qa | grep postgresql

# 5432 포트 사용 여부 확인
ss -tlnp | grep 5432
```

---

## 2. PGDG repo 등록

> Oracle Linux 기본 repo에는 구버전만 있다. PostgreSQL 공식 repo를 등록해야 최신 버전 설치가 가능하다.
> repo(Repository) = 소프트웨어 패키지를 저장하고 배포하는 저장소

```bash
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

repo 확인:

```bash
yum repolist | grep pgdg
```

---

## 3. PostgreSQL 15 설치

```bash
yum install -y postgresql15-server postgresql15-contrib
```

설치 확인:

```bash
rpm -qa | grep postgresql15
postgres --version
```

---

## 4. DB 초기화 (initdb)

> initdb는 PostgreSQL 데이터 디렉토리를 처음 생성하는 작업이다.
> Oracle의 DBCA로 DB를 생성하는 것과 유사한 개념이다.

#### initdb가 하는 일
```
/var/lib/pgsql/15/data/ 아래에

postgresql.conf    ← 설정 파일 생성
pg_hba.conf        ← 인증 설정 파일 생성
base/              ← 시스템 DB 생성
  ├── 1/           ← template1
  ├── 4/           ← template0  
  └── 5/           ← postgres DB
pg_wal/            ← WAL 디렉토리 생성
global/            ← 시스템 카탈로그 생성
```

```bash
/usr/pgsql-15/bin/postgresql-15-setup initdb
```

데이터 디렉토리 확인:

```bash
ls -la /var/lib/pgsql/15/data/
```

---

## 5. 서비스 시작 및 자동 시작 등록

```bash
systemctl start postgresql-15
systemctl enable postgresql-15
systemctl status postgresql-15
```

---

## 6. 기본 설정

### 6-1. postgresql.conf 수정

> DB 동작 전반을 제어하는 설정 파일이다. Oracle의 spfile/init.ora에 대응한다.

```bash
vi /var/lib/pgsql/15/data/postgresql.conf
```

아래 항목을 수정한다:

```
listen_addresses = '*'          # 모든 IP에서 접속 허용 (기본: localhost)
port = 5432                     # 기본 포트
max_connections = 100           # 최대 동시 접속 수
shared_buffers = 256MB          # 메모리 캐시 (전체 RAM의 25% 권장)
work_mem = 4MB                  # 정렬/해시 작업 메모리
maintenance_work_mem = 64MB     # VACUUM/인덱스 생성 메모리
wal_level = replica             # Streaming Replication 대비 설정
archive_mode = on               # Archive 모드 활성화
archive_command = 'cp %p /var/lib/pgsql/15/archive/%f'
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_min_duration_statement = 1000  # 1초 이상 쿼리 로그 기록
```

### 6-2. Archive 디렉토리 생성

```bash
mkdir -p /var/lib/pgsql/15/archive
chown postgres:postgres /var/lib/pgsql/15/archive
```

### 6-3. pg_hba.conf 수정

> 클라이언트 인증 방식을 제어하는 파일이다.
> Oracle의 sqlnet.ora + listener.ora 역할을 일부 담당한다.

```bash
vi /var/lib/pgsql/15/data/pg_hba.conf
```

아래 내용을 추가한다:

```
# TYPE  DATABASE  USER  ADDRESS         METHOD
host    all       all   172.31.0.0/16   md5
host    all       all   127.0.0.1/32    md5
```

### 6-4. 설정 적용

```bash
systemctl restart postgresql-15
```

---

## 7. postgres 유저 비밀번호 설정 및 접속

```bash
# postgres OS 유저로 전환
su - postgres

# psql 접속
psql
```

```sql
-- postgres DB 유저 비밀번호 설정
ALTER USER postgres WITH PASSWORD 'oracle';

-- 확인
\conninfo

EXIT;
```

---

## 8. alias 설정 (편의 설정)

> 자주 쓰는 명령어를 짧게 줄여 실무에서 빠르게 접속/운영할 수 있도록 한다.
> alias는 root가 아닌 postgres 유저의 ~/.bash_profile에 추가한다.

```bash
# postgres 유저로 전환
su - postgres

vi ~/.bash_profile
```

아래 내용을 추가한다:

```bash
# PostgreSQL alias
export PGDATA=/var/lib/pgsql/15/data
export PGPORT=5432
export PATH=/usr/pgsql-15/bin:$PATH

alias pg='psql -U postgres -p 5432'
alias pgdb='psql -U postgres -p 5432 -d'
alias pgstart='systemctl start postgresql-15'
alias pgstop='systemctl stop postgresql-15'
alias pgstatus='systemctl status postgresql-15'
alias pgreload='systemctl reload postgresql-15'
alias pglog='tail -100f /var/lib/pgsql/15/data/log/$(ls -t /var/lib/pgsql/15/data/log/ | head -1)'
alias pgconf='vi /var/lib/pgsql/15/data/postgresql.conf'
alias pghba='vi /var/lib/pgsql/15/data/pg_hba.conf'
```

```bash
source ~/.bash_profile
```

이후 아래처럼 간단하게 접속 가능하다:

```bash
pg                    # postgres DB 접속
pgdb testdb           # testdb 접속
pglog                 # 최신 로그 확인
pgstatus              # 서비스 상태 확인
```

---

## 9. 유저 및 DB 생성

```bash
pg
```

```sql
-- 유저 생성
CREATE USER scott WITH PASSWORD 'tiger';

-- DB 생성
CREATE DATABASE testdb OWNER scott;

-- 권한 부여
GRANT ALL PRIVILEGES ON DATABASE testdb TO scott;

-- 확인
\du        -- 유저 목록
\l         -- DB 목록
EXIT;
```
<img width="1188" height="576" alt="image" src="https://github.com/user-attachments/assets/2559cf42-2068-49c1-92b0-d7fdc00e459a" />

접속 테스트:

```bash
psql -U scott -d testdb -h 172.31.0.250
```

---

## 10. 최종 확인

```bash
systemctl status postgresql-15
ss -tlnp | grep 5432
psql -U postgres -c "SELECT version();"
```
<img width="1244" height="170" alt="image" src="https://github.com/user-attachments/assets/4704d7a1-4d1d-4470-a915-6b46e5a7b53b" />

---

## 11. Oracle vs PostgreSQL 주요 차이점

| 항목 | Oracle | PostgreSQL |
|------|--------|-----------|
| 설정 파일 | spfile / init.ora | postgresql.conf |
| 인증 설정 | sqlnet.ora | pg_hba.conf |
| 데이터 디렉토리 | $ORACLE_BASE/oradata | /var/lib/pgsql/15/data |
| 로그 | alert log | /data/log/*.log |
| 접속 명령어 | sqlplus user/pass@sid | psql -U user -d db |
| 기본 포트 | 1521 | 5432 |
| 스키마 개념 | User = Schema | Schema는 DB 안의 네임스페이스 |
| 자동 커밋 | OFF | OFF (Oracle과 동일) |
| 시퀀스 | CREATE SEQUENCE | CREATE SEQUENCE (동일) |
| 아카이브 로그 | Archive Log | WAL (Write-Ahead Log) |
| 복제 | Data Guard | Streaming Replication |
| 백업 | RMAN | pg_dump / pg_basebackup |

> PostgreSQL은 Oracle과 아키텍처가 가장 유사한 오픈소스 DB다.
> autocommit이 OFF로 기본 설정되어 있어 Oracle DBA에게 가장 친숙하다.

---

## 12. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| initdb 실패 | 데이터 디렉토리 권한 문제 | chown postgres:postgres /var/lib/pgsql |
| 서비스 시작 실패 | postgresql.conf 문법 오류 | journalctl -xe 로 오류 확인 |
| 원격 접속 실패 | pg_hba.conf 미설정 | pg_hba.conf에 클라이언트 IP 추가 |
| 원격 접속 실패 | listen_addresses = localhost | listen_addresses = '*' 로 변경 |
| 비밀번호 인증 실패 | peer 인증 방식 | pg_hba.conf에서 md5로 변경 |
