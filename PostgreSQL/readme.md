# PostgreSQL 이중화
데이터베이스의 안정성과 가용성을 높이기 위해 데이터를 복제하여 두개 이상의 서버에 저장하는 작업을 진행합니다. 
이중화를 통해 하나의 서버에 문제가 발생해도 다른 서버에서 데이터를 제공하도록하여 서비스의 연속성을 보장합니다. PostgreSQL에서는 주로 Streaming Replication 방식을 사용하여 이중화를 구현합니다.

### 환경 설정
- PostgreSQL 설치: 이중화를 구성할 모든 서버에 PostgreSQL을 설치합니다.
- 네트워크 설정: 이중화할 서버들이 서로 통신할 수 있도록 네트워크 설정을 확인합니다.
- DB, TABLE 생성
  ~~~
  CREATE DATABASE team13;

  CREATE TABLE car_info_table (
      DATA_ID VARCHAR(255) PRIMARY KEY,
      trmnCd VARCHAR(50),
      detcLot DOUBLE PRECISION,
      detcLat DOUBLE PRECISION,
      gpsUtcTime BIGINT,
      vhcleSped INTEGER,
      vhcleTypeCd VARCHAR(4),
      addr VARCHAR(30)
  );
  ~~~
## 설정
### 1. 복제 사용자 생성
메인 서버에서 PostgreSQL의 psql 명령어 인터페이스를 사용하여 복제 사용자를 생성합니다.

```sql
CREATE ROLE 유저이름 WITH REPLICATION LOGIN PASSWORD '비밀번호';
```

### 2. 메인 서버 설정
- **postgresql.conf 수정**
  - listen_addresses = '*' : 모든 주소에서의 접속을 허용합니다.
  - wal_level = replica : WAL(Write-Ahead Logging) 로그의 레벨을 replica로 설정합니다.
  - max_wal_senders = 10 : 최대 몇 개의 WAL 전송 프로세스를 실행할지 설정합니다.
  - archive_mode = on : 아카이브 모드를 활성화합니다.
  - archive_command = 'test ! -f /path/to/archive/%f && cp %p /path/to/archive/%f' : WAL 파일을 아카이브할 때 사용할 명령어를 설정합니다.
  - 위 경로에 맞게 archive파일 생성
    - 디렉토리 권한 확인 및 수정
    - 먼저, /home/ubuntu/ 디렉토리(또는 archive_command에서 지정한 경로)의 권한을 확인합니다. PostgreSQL프로세스가 파일을 쓸 수 있도록 적절한 권한이 설정되어 있는지 확인해야 합니다.
    - PostgreSQL 프로세스가 실행되는 사용자(일반적으로 postgres)가 해당 디렉토리에 쓸 수 있는 권한을 부여합니다.
      ```sql
      sudo chown postgres:postgres 아카이브 파일 경로
      ```
    - 이 명령어는 /home/ubuntu 디렉토리의 소유권을 postgres 사용자와 그룹에게 부여하고, 소유자만 읽기, 쓰기, 실행 권한을 가지도록 설정합니다.
      ```sql
      sudo chmod 700 아카이브 파일 경로
      ```
- **pg_hba.conf 수정**: 메인 서버의 pg_hba.conf 파일을 열고, 스탠바이 서버의 접속을 허용하는 규칙을 추가합니다.
  ```sql
  host replication all 스탠바이_서버_IP/32 md5
  ```
- **서버 재시작**: 설정을 수정했으면 postgresql을 재시작 한다.
  ```sql
  sudo systemctl restart postgresql.service
  ``` 
### 3. 스탠바이 서버 설정
- **postgresql.conf 수정**: 메인 서버의 postgresql.conf 파일을 열고, 다음 설정을 추가하거나 수정합니다.
  - listen_addresses = '*' : 모든 주소에서의 접속을 허용합니다.
  - restore_command = 'cp /path/to/archive/%f %p' : 스탠바이 서버에서 postgresql.conf 파일 또는 별도의 설정 파일에서 복제 관련 설정을 확인합니다.
  - primary_conninfo = 'host=main_server_ip port=5432 user=유저이름 password=비밀번호'
- **서버 종료하기**: 메인 서버 지우기 전에 postgresql stop합니다. 
- **메인 서버 비우기**: 메인 서버를 비우고 그 자리에 복제 데이터가 넘어오게 합니다.
  ```sql
  rm -rf /var/lib/postgresql/16/main/*
  ```
- **메인 서버 데이터 복제**: 스탠바이 서버에서 메인 서버의 데이터를 복제합니다.
  ```sql
  pg_basebackup -h 메인서버IP -D /var/lib/postgresql/16/main -U 유저이름 -P --wal-method=stream
  ```
- **standby.signal 생성**: 복제가 완료된 후, 스탠바이 서버의 PostgreSQL 데이터 디렉토리 내에 standby.signal 파일을 생성하고, 다음 설정을 추가합니다.
  - vim /var/lib/postgresql/16/main/standby.signal
    - primary_conninfo = 'host=메인_서버_IP port=5432 user=유저이름 password=비밀번호'
    - trigger_file = '/tmp/postgresql.trigger'
## 서비스 시작 및 확인
- **PostgreSQL 서비스 시작**: 메인 서버와 스탠바이 서버에서 PostgreSQL 서비스를 시작합니다.
  ```sql
  sudo systemctl start postgresql
  ```
- **이중화 상태 확인**: psql을 사용하여 메인 서버와 스탠바이 서버의 이중화 상태를 확인할 수 있습니다.
- **리플리케이션 확인** 
  - 메인 서버
  ```sql
  SELECT * FROM pg_stat_replication;
  ```
  - 스탠바이 서버
  ```sql
  SELECT * FROM pg_stat_wal_receiver;
  ```
---------------------------------------------------------------------------------------------
# 이중화 구현 시 replication slot의 사용
PostgreSQL에서 replication slot은 스탠바이 서버가 메인 서버로부터 데이터를 복제받기 위해 사용하는 메커니즘입니다. replication slot을 사용하면 스탠바이 서버가 일시적으로 연결이 끊겼다가 다시 연결되더라도, 누락된 데이터 없이 복제를 계속 진행할 수 있습니다. 하지만, 필요하지 않게 된 replication slot은 시스템 리소스를 낭비할 수 있으므로 초기화하여 사용하는 것이 좋습니다.
### slot 생성
- 물리적 복제 슬롯 생성
  ```sql
  SELECT pg_create_physical_replication_slot('my_physical_slot');
  ```
- 논리적 복제 슬롯 생성
  ```sql
  SELECT pg_create_logical_replication_slot('my_logical_slot', 'pgoutput');
  ```
### replication slot 초기화 방법
1. **현재 설정된 replication slot 확인**
   먼저, 현재 설정된 replication slot을 확인해야 합니다. 이는 메인 서버에서 psql을 사용하여 확인할 수 있습니다.
   ```sql
   SELECT * FROM pg_replication_slots;
   ```
2. **replication slot 삭제**
   더 이상 필요하지 않은 replication slot을 확인했다면, 해당 slot을 삭제할 수 있습니다. 이 작업 역시 메인 서버에서 수행합니다.
    ```sql
    SELECT pg_drop_replication_slot('slot_name');
    ``` 
### 주의사항
- replication slot을 삭제하기 전에, 해당 slot을 사용하는 스탠바이 서버가 더 이상 해당 slot을 사용하지 않거나, 서비스에 영향을 주지 않는지 확인해야 합니다.
- replication slot을 삭제하면 해당 slot에 대한 정보와 스탠바이 서버가 복제해야 할 WAL 파일 정보가 삭제됩니다. 따라서, 스탠바이 서버가 해당 slot을 여전히 필요로 한다면, 데이터 복제에 문제가 발생할 수 있습니다.
replication slot을 초기화하는 작업은 시스템의 리소스를 관리하고, 불필요한 복제 작업을 방지하기 위해 중요합니다. 하지만, 신중하게 수행해야 하며, 작업 전에 반드시 현재 복제 상태와 시스템의 요구 사항을 충분히 이해하고 있어야 합니다.

### 서버 종료시 순서
- 스탠바이 서버 먼저 종료 후, 메인 서버 종료
- 반대로 시작할 때는 메인 서버 먼저 시작하고, 스탠바이 서버 시작
