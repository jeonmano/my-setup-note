# Airflow 3.1.6 Installation & Setup Guide

본 문서는 **Docker Compose**를 사용하여 **Airflow 3.1.6 (Dev)** 환경을 구축하는 과정을 다룹니다.
특히 **API Server, Scheduler, Triggerer, DAG Processor**가 분리된 3.0 아키텍처의 인증(JWT) 및 네트워크 설정, 트러블슈팅 경험을 기반으로 작성되었습니다.

## 1. 환경 정보 (Environment)
* **OS:** Windows (Docker Desktop / WSL2)
* **Airflow Version:** 3.1.6 (image: `apache/airflow:latest`)
* **Executor:** LocalExecutor
* **Database:** External PostgreSQL (Host Machine)
* **Network Strategy:** Host IP Direct Connection (Reliability)

---

## 2. 프로젝트 구조 (Project Structure)

설정과 민감 정보를 분리하여 관리합니다.

```text
airflow-3.0-project/
├── .env                  # [필수] DB 접속 정보, 보안 키 등 환경변수
├── docker-compose.yml    # [필수] 4개 서비스(API, Sch, Trig, DP) 정의
├── rb.bat                # [옵션] 재기동 자동화 스크립트 (Windows)
├── dags/                 # DAG 파일 저장소
└── logs/                 # 로그 저장소 (Git 제외)
```

---

## 3. 환경 변수 설정 (.env)

`docker-compose.yml`에서 사용할 공통 변수입니다. **보안 키**와 **DB 주소**를 이곳에서 관리합니다.

```ini
# .env 파일 예시

# 1. Database Connection (Host IP 사용 권장)
# host.docker.internal 대신 실제 물리 IP를 사용하는 것이 가장 확실함
DB_URL=postgresql+psycopg2://airflow:airflow@192.168.0.100:5432/airflow_db

# 2. Security Keys (모든 서비스가 동일한 값을 가져야 함)
# openssl rand -hex 16 등으로 생성 가능
SECRET_KEY=9e866671ad6249679f22c8332d0c242c

# 3. Fernet Key (Connection 정보 암호화용)
# 500 에러 방지를 위해 반드시 고정된 값을 사용
# python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
FERNET_KEY=7fbR-H0u7L-Lz_X1xYV-qLwK_H7V6_Z-8fH-wLz-X1x=

# 4. Timezone
TZ=Asia/Seoul
```

---

## 4. Docker Compose 구성 (docker-compose.yml)

Airflow 3.0의 핵심 4개 서비스를 정의합니다. **환경 변수 명칭 규칙(언더바 2개)**에 주의해야 합니다.

```yaml
version: '3.8'

x-airflow-common: &airflow-common
  image: apache/airflow:latest
  environment:
    # 1. Core & DB
    - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=${DB_URL}
    - AIRFLOW__CORE__EXECUTOR=LocalExecutor
    - AIRFLOW__CORE__LOAD_EXAMPLES=False
    - AIRFLOW__CORE__FERNET_KEY=${FERNET_KEY}
    - TZ=${TZ}

    # 2. Security & Auth (Signature Verification)
    - AIRFLOW__CORE__SECRET_KEY=${SECRET_KEY}
    - AIRFLOW__API__SECRET_KEY=${SECRET_KEY}
    - AIRFLOW__API__EXECUTION_API_SECRET_KEY=${SECRET_KEY}
    
    # [중요] 섹션(API_AUTH)과 키(JWT_SECRET) 사이는 언더바 2개(__)
    - AIRFLOW__API_AUTH__JWT_SECRET=${SECRET_KEY}

    # 3. Internal Communication
    # 스케줄러 등이 API 서버를 찾을 때 사용하는 주소
    - AIRFLOW__CORE__EXECUTION_API_SERVER_URL=http://airflow-api:8080/execution
    
    # [중요] internal_api가 가장 앞에 와야 함
    - AIRFLOW__API__AUTH_BACKENDS=airflow.api.auth.backend.internal_api,airflow.providers.fab.auth_manager.api.auth.backend.session,airflow.providers.fab.auth_manager.api.auth.backend.basic_auth

  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
  networks:
    - airflow-net

services:
  # 1. API Server (Core)
  airflow-api:
    <<: *airflow-common
    container_name: airflow-api
    ports:
      - "8080:8080"
    command: api-server

  # 2. Scheduler
  airflow-scheduler:
    <<: *airflow-common
    container_name: airflow-scheduler
    depends_on:
      - airflow-api
    command: scheduler

  # 3. Triggerer (Deferrable Operators)
  airflow-triggerer:
    <<: *airflow-common
    container_name: airflow-triggerer
    depends_on:
      - airflow-api
    command: triggerer

  # 4. DAG Processor (Parsing)
  airflow-dag-processor:
    <<: *airflow-common
    container_name: airflow-dag-processor
    depends_on:
      - airflow-api
    command: dag-processor

networks:
  airflow-net:
    driver: bridge
```

---

## 5. 실행 및 관리 (Operations)

### 5.1 재기동 스크립트 (rb.bat)
윈도우 환경에서 `make` 대신 사용할 배치 파일입니다.

```batch
@echo off
echo [Reboot] Stopping all services...
docker-compose down --remove-orphans

echo [Reboot] Pulling latest images...
docker-compose pull

echo [Reboot] Starting Airflow 3.0 Full Stack...
docker-compose up -d --force-recreate
docker ps
pause
```

### 5.2 초기화 명령어
최초 실행 시 DB 스키마 생성 및 사용자 생성이 필요합니다.

```bash
# DB 마이그레이션 (필수)
docker exec -it airflow-scheduler airflow db migrate

# 관리자 계정 생성
docker exec -it airflow-scheduler airflow users create ^
    --username admin --firstname Minho --lastname Lee ^
    --role Admin --email admin@example.com --password admin
```

---

## 6. 주요 트러블슈팅 (Troubleshooting)

### 6.1 커넥션 화면 500 에러 (Internal Server Error)
* **증상:** Web UI에서 Connections 메뉴 진입 시 500 에러 발생.
* **원인:** DB에 저장된 암호화된 커넥션 정보를 현재의 `FERNET_KEY`로 복호화 실패. (키가 변경됨)
* **해결:**
  1. `.env`에 `FERNET_KEY`를 고정값으로 설정.
  2. DB에서 기존 오염된 데이터 삭제.
     ```sql
     TRUNCATE TABLE connection CASCADE;
     ```

### 6.2 Signature verification failed (Auth Token Error)
* **증상:** 스케줄러 로그에 `Invalid auth token: Signature verification failed` 반복.
* **원인:**
  1. `airflow-api`와 `airflow-scheduler`가 서로 다른 `SECRET_KEY`를 가짐.
  2. 환경 변수 명칭 오타 (예: `AIRFLOW__API_AUTH_JWT_SECRET` -> 언더바 1개).
* **해결:**
  1. `docker-compose.yml`에서 `AIRFLOW__API_AUTH__JWT_SECRET` (언더바 2개) 확인.
  2. `AUTH_BACKENDS`에 `internal_api`가 최우선 순위인지 확인.
  3. `docker-compose pull`로 이미지 버전 동기화.

### 6.3 DB Sequence 충돌 (Duplicate Key Error)
* **증상:** `UniqueViolation: Key (id)=(24) already exists.`
* **원인:** 비정상 종료 후 DB 시퀀스와 실제 데이터 불일치.
* **해결:**
  ```bash
  # Job 테이블 정리 (미래 시점으로 정리)
  docker exec -it airflow-scheduler airflow db clean --tables job --clean-before-timestamp '2099-12-31' --yes
  ```
