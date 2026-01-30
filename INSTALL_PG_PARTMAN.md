# pg_partman 5.4.0 Installation & Usage Guide

본 문서는 **PostgreSQL 16.3** 엔진이 이미 설치된 환경에서 파티션 관리 도구인 **pg_partman 5.4.0**을 소스 컴파일 방식으로 설치하고 연동하는 과정을 다룹니다.

## 1. 환경 정보 (Environment)
* **DB Version:** PostgreSQL 16.3 (Installed)
* **Extension:** pg_partman 5.4.0
* **Install Prefix:** `$HOME/pgsql16`
* **Data Directory:** `$HOME/pgsql16/data`
* **Port:** `15432`

---

## 2. 소스 빌드 및 설치 (Build & Install)

PostgreSQL의 `bin` 경로가 `PATH`에 등록되어 있어야 합니다. (`pg_config` 명령어 필요)

```bash
# 1. 소스 디렉토리로 이동
# (파일은 $HOME에 다운로드 되어 있다고 가정)
cd $HOME/pg_partman-5.4.0

# 2. 빌드 및 설치
# (별도의 configure 없이 make로 바로 진행)
make
make install
```
> **확인:** 설치 마지막에 `.../share/extension/` 경로로 `pg_partman.control` 파일 등이 복사되면 성공입니다.

---

## 3. PostgreSQL 설정 변경 (Configuration)

pg_partman의 자동 파티션 관리 기능(Background Worker)을 사용하기 위해 `postgresql.conf`를 수정합니다.

```bash
vi $HOME/pgsql16/data/postgresql.conf
```

**[수정할 내용]**
파일 내 `shared_preload_libraries` 항목을 찾아 아래와 같이 수정합니다. (주석 해제 및 값 추가)

```ini
# pg_partman BGW 라이브러리 로드 (필수)
shared_preload_libraries = 'pg_partman_bgw'
```

**[서버 재기동]**
라이브러리 로드 설정은 **반드시 DB를 재기동**해야 적용됩니다.

```bash
pg_ctl restart

# (참고) 로그 확인
# tail -f $HOME/pgsql16/data/log/postgresql-*.log
```

---

## 4. 확장 기능 등록 (Extension Registration)

DB에 접속하여 확장을 설치합니다. 관리를 위해 별도의 `partman` 스키마 사용을 권장합니다.

```bash
# 접속 (포트 15432 주의)
psql -p 15432 -d postgres
```

```sql
-- 1. 전용 스키마 생성
CREATE SCHEMA partman;

-- 2. 확장 기능 설치 (partman 스키마에)
CREATE EXTENSION pg_partman SCHEMA partman;

-- 3. 설치 확인
\dx
```

---

## 5. 파티션 테이블 생성 및 검증 (Verification)

### 5.1 테스트 테이블 생성
```sql
CREATE TABLE part_test (
    id SERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    content TEXT
) PARTITION BY RANGE (created_at);
```

### 5.2 파티션 적용 (create_parent)
**주의 (v5.x 문법 변경 사항):**
* `p_type`: 'native'가 기본값이므로 파라미터 삭제.
* `p_interval`: 'daily' 대신 표준 간격인 **'1 day'** 사용.

```sql
SELECT partman.create_parent(
    p_parent_table => 'public.part_test',
    p_control => 'created_at',
    p_interval => '1 day',
    p_premake => 3
);
```

### 5.3 결과 확인
1.  **Default Partition:** `part_test_default` 생성됨 (범위 밖 데이터 수용).
2.  **Date Partitions:** 현재 날짜 기준 과거 버퍼 + 미래 3일 치 생성됨.
```sql
\dt part_test*
```

---

## 6. 백그라운드 워커(BGW) 동작 확인 (Maintenance)

pg_partman이 자동으로 미래 파티션을 생성하는지 확인합니다.

### 6.1 동작 원리
* `pg_partman`은 `p_premake` 설정만큼 미래 파티션을 **미리** 확보해 둡니다.
* 00시 정각에 생성되는 것이 아니라, BGW가 주기적으로 돌면서 미래 파티션이 부족하면 미리 생성합니다.

### 6.2 확인 방법
**방법 1: OS 프로세스 확인**
```bash
ps -ef | grep "pg_partman master background worker"
```
> 위 프로세스가 떠 있으면 정상 작동 중.

**방법 2: 로그 파일 확인**
```bash
tail -f $HOME/pgsql16/data/log/postgresql-*.log
```
> 주기적으로 `partman ... run_maintenance` 로그가 보이면 정상.

**방법 3: 수동 실행 (즉시 생성)**
```sql
SELECT partman.run_maintenance_proc();
```
