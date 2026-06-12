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
| Primary + PgPool-II | pg-primary | 172.31.0.250 |
| Standby | pg-standby | 172.31.0.251 |

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
nmcli con mod eth0 ipv4.addresses 172.31.0.250/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

**pg-standby 노드에서:**

```bash
hostnamectl set-hostname pg-standby
nmcli con mod eth0 ipv4.addresses 172.31.0.251/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

### 1-3. /etc/hosts 등록 (전 노드 동일)

> IP 주소와 호스트명을 매핑하는 파일이다.
> 호스트명으로 통신할 때 DNS 서버보다 먼저 참조하므로 반드시 등록한다.

```bash
vi /etc/hosts
```

```text
172.31.0.250  pg-primary
172.31.0.251  pg-standby
```

---

## 2. Streaming Replication 구성

### 2-1. Primary 설정 (pg-primary에서)

**postgresql.conf 수정:**

> `wal_level = replica` 이상이어야 Standby로 WAL 전송이 가능하다.
> `max_wal_senders`는 동시에 WAL을 전송할 수 있는 최대 연결 수다.

```bash
vi /var/lib/pgsql/15/data/postgresql.conf
```

```text
listen_addresses = '*'
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64
```

**pg_hba.conf에 복제 접속 허용 추가:**

> Standby(172.31.0.251)가 Primary에 복제 목적으로 접속할 수 있도록 허용한다.

```bash
vi /var/lib/pgsql/15/data/pg_hba.conf
```

```text
host  replication  replication  172.31.0.251/32  md5
```

<img width="407" height="94" alt="image" src="https://github.com/user-attachments/assets/06a52168-4022-415d-95d2-aecc449bcb12" />

**복제 유저 생성:**

```bash
su - postgres
psql
```

```sql
CREATE USER replication WITH REPLICATION PASSWORD '1234';

-- 확인
\du
EXIT;
```

```bash
systemctl restart postgresql-15
```

### 2-2. Standby 설정 (pg-standby에서)

**기존 데이터 디렉토리 초기화 후 베이스 백업 수신:**

> `-R` 옵션이 standby.signal 파일과 복제 설정(postgresql.auto.conf)을 자동 생성한다.

```bash
systemctl stop postgresql-15
rm -rf /var/lib/pgsql/15/data/*

# Primary에서 베이스 백업 수신
pg_basebackup -h 172.31.0.250 -U replication \
  -D /var/lib/pgsql/15/data \
  -Fp -Xs -P -R
```

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
       pg_size_pretty(sent_lsn - replay_lsn) AS replication_lag
FROM pg_stat_replication;
```

> `state = streaming` 이고 `replication_lag = 0 bytes` 이면 정상이다.

**Standby에서:**

```sql
SELECT * FROM pg_stat_wal_receiver;

-- Standby 여부 확인 (true = Standby)
SELECT pg_is_in_recovery();
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

```text
# 접속 설정
listen_addresses = '*'
port = 9999

# 백엔드 설정 (Primary)
backend_hostname0 = 'pg-primary'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/var/lib/pgsql/15/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'

# 백엔드 설정 (Standby)
backend_hostname1 = 'pg-standby'
backend_port1 = 5432
backend_weight1 = 2
backend_data_directory1 = '/var/lib/pgsql/15/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'

# 커넥션 풀 설정
num_init_children = 32
max_pool = 4

# 로드 밸런싱 (읽기 쿼리를 Primary/Standby에 분산)
load_balance_mode = on

# 헬스 체크 (노드 장애 감지)
health_check_period = 10
health_check_timeout = 5
health_check_user = 'postgres'
health_check_password = 'oracle'

# Streaming Replication 모드
backend_clustering_mode = 'streaming_replication'
sr_check_user = 'replication'
sr_check_password = '1234'
```

### 3-3. pool_hba.conf 설정

```bash
cp /etc/pgpool-II/pool_hba.conf.sample /etc/pgpool-II/pool_hba.conf
vi /etc/pgpool-II/pool_hba.conf
```

```text
host  all  all  172.31.0.0/16  md5
```

### 3-4. pool_passwd 파일 생성

> PgPool-II가 백엔드 DB에 접속할 때 사용하는 비밀번호를 MD5로 암호화하여 저장한다.

```bash
pg_md5 -m -u postgres oracle
pg_md5 -m -u replication 1234
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
psql -h 172.31.0.250 -p 9999 -U postgres -d testdb
```

### 4-2. 노드 상태 확인

```sql
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
SELECT inet_server_addr();
SHOW pool_nodes;
```

### 4-4. 장애 테스트

```bash
# pg-standby 서비스 중지 (pg-standby에서)
systemctl stop postgresql-15

# PgPool에서 노드 상태 확인 (standby status = down 으로 변경됨)
psql -h 172.31.0.250 -p 9999 -U postgres -c "SHOW pool_nodes;"

# pg-standby 복구 (pg-standby에서)
systemctl start postgresql-15

# PgPool에 재합류
pcp_attach_node -h 172.31.0.250 -p 9898 -u pgpool -w 1
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
| pg_basebackup 인증 실패 | pg_hba.conf 미설정 또는 Standby IP 오류 | replication 허용 행 추가, IP 확인 |
| pg_basebackup 연결 실패 | listen_addresses = localhost | postgresql.conf에서 `*`로 변경 |
| Standby 시작 실패 | standby.signal 없음 | pg_basebackup -R 옵션 확인 |
| PgPool 접속 실패 | pool_hba.conf 미설정 | 클라이언트 IP 허용 추가 |
| 노드 장애 감지 안 됨 | health_check_period 미설정 | pgpool.conf 헬스 체크 설정 |
| 읽기 분산 안 됨 | load_balance_mode = off | on으로 변경 후 재시작 |
