# PostgreSQL 16.3 Source Installation Guide (Oracle Linux 9.5)

본 문서는 **Oracle Linux 9.5 (WSL 환경)**에서 **PostgreSQL 16.3**을 소스 컴파일 방식으로 설치하는 과정을 다룹니다.  
일반 사용자(`appadmin`) 계정의 홈 디렉토리에 설치하여 시스템 영역과 분리된 환경을 구성합니다.

## 1. 환경 정보 (Environment)
* **OS:** Oracle Linux 9.5 (WSL)
* **DB Version:** PostgreSQL 16.3
* **Install Prefix:** `$HOME/pgsql16` (User Home Directory)
* **Data Directory:** `$HOME/pgsql16/data`
* **Port:** `15432` (기본 5432 포트 충돌 방지)

## 2. 사전 준비 (Prerequisites)

### 2.1 OS 업데이트 및 필수 패키지 설치
소스 빌드에 필요한 컴파일러와 라이브러리를 설치합니다.
> **Note:** `libicu-devel` 패키지가 없으면 `./configure` 단계에서 ICU 라이브러리 미검출 에러가 발생합니다.

```bash
# 시스템 업데이트
sudo yum update -y

# 개발 툴킷 및 필수 라이브러리 설치
sudo yum groupinstall "Development Tools" -y
sudo yum install readline-devel zlib-devel libicu-devel -y
```

### 2.2 소스 파일 준비
PostgreSQL 16.3 및 추후 설치할 pg_partman 소스를 다운로드합니다.

```bash
# 작업 디렉토리 이동
cd $HOME

# 소스 파일 다운로드 (또는 윈도우에서 복사)
# postgresql-16.3.tar.gz
# pg_partman-5.4.0.tar.gz

# 압축 해제
tar -zxvf postgresql-16.3.tar.gz
```

## 3. 빌드 및 설치 (Build & Install)

### 3.1 환경 설정 (Configure)
사용자 홈 디렉토리(`$HOME/pgsql16`)에 설치되도록 `--prefix` 옵션을 지정합니다.

```bash
cd postgresql-16.3

# Prefix 지정 및 설정 검사
./configure --prefix=$HOME/pgsql16
```

### 3.2 컴파일 및 설치 (Make)
```bash
# 빌드 수행
make

# 설치 (지정한 Prefix 경로로 바이너리 복사)
make install
```

## 4. 환경 변수 설정 (Environment Variables)

설치된 PostgreSQL 명령어를 어디서든 실행할 수 있도록 `.bash_profile`에 경로를 등록합니다.

```bash
vi ~/.bash_profile
```

**추가할 내용:**
```bash
# PostgreSQL 16 Paths
export PATH=$HOME/pgsql16/bin:$PATH
export LD_LIBRARY_PATH=$HOME/pgsql16/lib:$LD_LIBRARY_PATH
export PGDATA=$HOME/pgsql16/data
# export PGPORT=15432  # (선택 사항) 포트를 환경변수로 고정하고 싶을 때 사용
```

설정 적용:
```bash
source ~/.bash_profile
```

## 5. 데이터베이스 초기화 (Initialize Database)

데이터가 저장될 클러스터(Data Directory)를 생성하고 초기화합니다.

```bash
# 데이터 폴더 생성
mkdir -p $HOME/pgsql16/data

# DB 초기화
initdb -D $HOME/pgsql16/data
```

## 6. 서버 설정 (Configuration)

### 6.1 포트 및 로깅 설정 변경
기본 포트(`5432`) 충돌 방지를 위해 포트를 변경하고, 운영 관리를 위해 로그 설정을 구체화합니다.

```bash
vi $HOME/pgsql16/data/postgresql.conf
```

**주요 변경 사항:**
```ini
# 1. 포트 변경 (기본 5432 -> 15432)
port = 15432

# 2. 로깅 설정 (Logging Configuration)
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_truncate_on_rotation = off
```

### 6.2 로그 디렉토리 생성
설정 파일에서 지정한 `log` 폴더를 미리 생성합니다.
```bash
mkdir -p $HOME/pgsql16/data/log
```

## 7. 서버 기동 및 접속 (Start & Connect)

### 7.1 서버 시작
```bash
# 로그 옵션은 설정 파일(postgresql.conf)에 지정했으므로 생략 가능
pg_ctl start

# (참고) 실행 로그 확인
# tail -f $HOME/pgsql16/data/log/postgresql-*.log
```

**정상 기동 로그 예시:**
```
LOG:  listening on IPv4 address "127.0.0.1", port 15432
LOG:  database system is ready to accept connections
```

### 7.2 접속 테스트
변경된 포트(`15432`)를 명시하여 접속합니다.

```bash
psql -U appadmin -d postgres -p 15432
```


**정상 접속 시 예시:**
```
[appadmin@DESKTOP-3U0O7D9 data]$ psql -U appadmin -d postgres -p 15432
psql (16.3)
Type "help" for help.

postgres=#
```

---

