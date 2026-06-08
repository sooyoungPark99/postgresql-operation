# 3. PgPool-II 구성

Streaming Replication 구성 후 PgPool-II로 부하 분산 및 커넥션 풀링을 구성한다.

---

## 단계별 수행 노드 요약

| 단계 | 작업 | 수행 노드 |
|------|------|----------|
| 1 | VM 복제 및 IP 변경 | VirtualBox / 각 노드 |
| 2 | Streaming Replication 설정 | Primary, Standby |
| 3 | PgPool-II 설치 및 설정 | Primary (PgPool 겸임) |
| 4 | 동작 확인 | 아무 노드 |

---

## 서버 구성

| 역할 | 호스트명 | IP |
|------|---------|----|
| Primary + PgPool-II | pg-primary | 172.31.0.260 |
| Standby | pg-standby | 172.31.0.261 |

> PostgreSQL Single 설치가 완료된 VM을 복제하여 2대를 구성한다.

---

## 1. VM 복제 및 사전 준비

### 1-1. VM 복제 (VirtualBox)

Single 설치 완료된 VM을 복제하여 2대 구성한다.

1. VirtualBox → 기존 PostgreSQL VM 우클릭 → 복제
2. 복제 방식: 완전한 복제 (Full Clone)
3. MAC 주소 정책: 모든 네트워크 어댑터의 새 MAC 주소 생성
4. 위 과정 반복하여 pg-primary, pg-standby 2대 구성

### 1-2. 호스트명 및 IP 변경 (각 노드에서)

**pg-primary 노드에서:**

```bash
hostnamectl set-hostname pg-primary
nmcli con mod eth0 ipv4.addresses 172.31.0.260/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

**pg-standby 노드에서:**

```bash
hostnamectl set-hostname pg-standby
nmcli con mod eth0 ipv4.addresses 172.31.0.261/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

### 1-3. /etc/hosts 등록 (전 노드 동일)

```bash
vi /etc/hosts
```

```
172.31.0.260  pg-primary
172.31.0.261  pg-standby
```

---

## 2. Streaming Replication 구성

### 2-1. Primary 설정 (pg-primary에서)

**postgresql.conf 수정:**

```bash
vi /var/lib/pgsql/15/data/postgresql.conf
```

```
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64
```

**pg_hba.conf에 복제 접속 허용 추가:**

```bash
vi /var/lib/pgsql/15/data/pg_hba.conf
```

```
host  replication  replicator  172.31.0.261/32  md5
```

**복제 유저 생성:**

```bash
psql -U postgres
```

```sql
CREATE USER replicator WITH REPLICATION PASSWORD 'replpass';
EXIT;
```

```bash
systemctl restart postgresql-15
```

### 2-2. Standby 설정 (pg-standby에서)

**기존 데이터 디렉토리 초기화 후 베이스 백업 수신:**

```bash
systemctl stop postgresql-15
rm -rf /var/lib/pgsql/15/data/*

# Primary에서 베이스 백업 수신
pg_basebackup -h 172.31.0.260 -U replicator \
  -D /var/lib/pgsql/15/data -Fp -Xs -P -R
```

> `-R` 옵션이 standby.signal 파일과 복제 설정을 자동 생성한다.

```bash
# 권한 설정
chown -R postgres:postgres /var/lib/pgsql/15/data
chmod 700 /var/lib/pgsql/15/data

systemctl start postgresql-15
```

### 2-3. 복제 상태 확인

**Primary에서:**

```sql
SELECT client_addr, state, sync_state,
       (sent_lsn - replay_lsn) AS replication_lag
FROM pg_stat_replication;
```

**Standby에서:**

```sql
SELECT * FROM pg_stat_wal_receiver;
```

---

## 3. PgPool-II 설치 및 설정 (pg-primary에서)

### 3-1. PgPool-II 설치

```bash
yum install -y https://www.pgpool.net/yum/rpms/4.4/redhat/rhel-7-x86_64/pgpool-II-release-4.4-1.noarch.rpm
yum install -y pgpool-II-pg15 pgpool-II-pg15-devel
```

설치 확인:

```bash
pgpool --version
```

### 3-2. PgPool-II 설정

```bash
cp /etc/pgpool-II/pgpool.conf.sample /etc/pgpool-II/pgpool.conf
vi /etc/pgpool-II/pgpool.conf
```

아래 항목을 수정한다:

```
# 접속 설정
listen_addresses = '*'
port = 9999                    # PgPool 접속 포트

# 백엔드 설정 (Primary)
backend_hostname0 = 'pg-primary'
backend_port0 = 5432
backend_weight0 = 1            # 쓰기 비중
backend_data_directory0 = '/var/lib/pgsql/15/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'

# 백엔드 설정 (Standby)
backend_hostname1 = 'pg-standby'
backend_port1 = 5432
backend_weight1 = 2            # 읽기 비중 (Primary보다 높게)
backend_data_directory1 = '/var/lib/pgsql/15/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'

# 커넥션 풀 설정
num_init_children = 32         # 미리 생성할 프로세스 수
max_pool = 4                   # 프로세스당 최대 캐시 연결 수

# 로드 밸런싱
load_balance_mode = on         # 읽기 쿼리 분산 활성화

# 헬스 체크 (노드 장애 감지)
health_check_period = 10       # 10초마다 체크
health_check_timeout = 5
health_check_user = 'postgres'
health_check_password = 'oracle'

# 복제 모드
backend_clustering_mode = 'streaming_replication'
sr_check_user = 'replicator'
sr_check_password = 'replpass'
```

### 3-3. pool_hba.conf 설정

```bash
cp /etc/pgpool-II/pool_hba.conf.sample /etc/pgpool-II/pool_hba.conf
vi /etc/pgpool-II/pool_hba.conf
```

```
host  all  all  172.31.0.0/16  md5
```

### 3-4. pool_passwd 파일 생성

```bash
pg_md5 -m -u postgres oracle
pg_md5 -m -u replicator replpass
```

### 3-5. PgPool-II 시작

```bash
systemctl start pgpool
systemctl enable pgpool
systemctl status pgpool
```

---

## 4. 동작 확인

### 4-1. PgPool-II 접속

```bash
# PgPool을 통해 DB 접속 (포트 9999)
psql -h 172.31.0.260 -p 9999 -U postgres -d testdb
```

### 4-2. 노드 상태 확인

```sql
-- PgPool에서 백엔드 노드 상태 확인
SHOW pool_nodes;
```

```
 node_id | hostname    | port | status | weight | role    |
---------+-------------+------+--------+--------+---------|
 0       | pg-primary  | 5432 | up     | 0.333  | primary |
 1       | pg-standby  | 5432 | up     | 0.667  | standby |
```

### 4-3. 부하 분산 확인

```sql
-- 여러 번 실행하여 다른 노드에서 처리되는지 확인
SHOW pool_nodes;
SELECT inet_server_addr();
```

### 4-4. 장애 테스트

```bash
# pg-standby 서비스 중지
systemctl stop postgresql-15  # pg-standby에서

# PgPool에서 노드 상태 확인 (failover 감지)
psql -h 172.31.0.260 -p 9999 -U postgres -c "SHOW pool_nodes;"
# standby 노드 status = down 으로 변경됨

# pg-standby 복구
systemctl start postgresql-15  # pg-standby에서

# PgPool에 재합류
pcp_attach_node -h 172.31.0.260 -p 9898 -u pgpool -w 1
```

---

## 5. Oracle vs PostgreSQL HA 비교

| 항목 | PostgreSQL | Oracle |
|------|-----------|--------|
| 복제 방식 | Streaming Replication | Data Guard |
| 복제 방향 | Primary → Standby (단방향) | Primary → Standby |
| 프록시 | PgPool-II | Oracle Connection Manager |
| 부하 분산 | PgPool-II | Oracle RAC / Service |
| Failover | PgPool-II 자동 | DGMGRL 자동 |
| Multi-Primary | 없음 (단방향만) | RAC |

---

## 6. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| pg_basebackup 실패 | pg_hba.conf 미설정 | replication 허용 행 추가 |
| Standby 시작 실패 | standby.signal 없음 | pg_basebackup -R 옵션 확인 |
| PgPool 접속 실패 | pool_hba.conf 미설정 | 클라이언트 IP 허용 추가 |
| 노드 장애 감지 안 됨 | health_check_period 미설정 | pgpool.conf 헬스 체크 설정 |
| 읽기 분산 안 됨 | load_balance_mode = off | on으로 변경 후 재시작 |
