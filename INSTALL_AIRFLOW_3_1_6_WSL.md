


# Python 3.12 설치
```
[appadmin@DESKTOP-3U0O7D9 ~]$ sudo dnf install python3.12 python3.12-devel python3.12-pip
[appadmin@DESKTOP-3U0O7D9 ~]$ which python3.12
/usr/bin/python3.12
```

# 환경 변수 설정
```
[appadmin@DESKTOP-3U0O7D9 ~]$ vi .bashrc
export AIRFLOW_HOME=/home/appadmin/airflow

alias python='python3.12'
alias pip='pip3.12'
alias airflowon='source /home/appadmin/airflow/airflow_env/bin/activate'

[appadmin@DESKTOP-3U0O7D9 ~]$ python -V
Python 3.12.12
[appadmin@DESKTOP-3U0O7D9 ~]$ pip -V
pip 23.2.1 from /usr/lib/python3.12/site-packages/pip (python 3.12)
```

# 가상 환경 생성
```
[appadmin@DESKTOP-3U0O7D9 ~]$ mkdir airflow && cd airflow
[appadmin@DESKTOP-3U0O7D9 airflow]$ python -m venv airflow_env
```


# airflow 설치
가상 환경 접속 후 수행
```

((airflow_env) ) [appadmin@DESKTOP-3U0O7D9 ~]$ pip install "apache-airflow==3.1.6" "apache-airflow-providers-fab"

((airflow_env) ) [appadmin@DESKTOP-3U0O7D9 ~]$ airflow version
2026-02-06T13:29:27.970198Z [info     ] setup plugin alembic.autogenerate.schemas [alembic.runtime.plugins] loc=plugins.py:37
2026-02-06T13:29:27.970337Z [info     ] setup plugin alembic.autogenerate.tables [alembic.runtime.plugins] loc=plugins.py:37
2026-02-06T13:29:27.970415Z [info     ] setup plugin alembic.autogenerate.types [alembic.runtime.plugins] loc=plugins.py:37
2026-02-06T13:29:27.970487Z [info     ] setup plugin alembic.autogenerate.constraints [alembic.runtime.plugins] loc=plugins.py:37
2026-02-06T13:29:27.970558Z [info     ] setup plugin alembic.autogenerate.defaults [alembic.runtime.plugins] loc=plugins.py:37
2026-02-06T13:29:27.970635Z [info     ] setup plugin alembic.autogenerate.comments [alembic.runtime.plugins] loc=plugins.py:37
3.1.6

```



# DB 작업

```
CREATE DATABASE airflow_db_wsl;
CREATE USER airflow WITH PASSWORD 'airflow';
ALTER DATABASE airflow_db_wsl OWNER TO airflow;
```

# admin 계정 생성
```
airflow users create --username admin --firstname admin --lastname admin --role Admin --email admin@example.com --password admin
```


# airflow.cfg
```
[core]
executor = LocalExecutor
load_examples = False
secret_key = def2a1e5f0535f2390be793e90fb0ca957d7a031356b982148755fdab698
execution_api_server_url = http://localhost:8080/execution
auth_manager = airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
fernet_key = ASnPlM78jbVTcjzYIyf61Ln3PFse2T-khYViJT_o57Y=
[database]
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@172.29.128.1:5432/airflow_db_wsl
[webserver]
secret_key = def2a1e5f0535f2390be793e90fb0ca957d7a031356b982148755fdab698

[api]
secret_key = def2a1e5f0535f2390be793e90fb0ca957d7a031356b982148755fdab698
execution_api_secret_key = def2a1e5f0535f2390be793e90fb0ca957d7a031356b982148755fdab698
auth_backends = airflow.providers.fab.auth_manager.api.auth.backend.session,airflow.providers.fab.auth_manager.api.auth.backend.basic_auth
port=8080
[auth]
jwt_secret = def2a1e5f0535f2390be793e90fb0ca957d7a031356b982148755fdab698

[api_auth]
jwt_issuer=airflow
jwt_secret = def2a1e5f0535f2390be793e90fb0ca957d7a031356b982148755fdab698
```

# airflow 3.1.7 (celery + redis)
```
[core]
executor = CeleryExecutor
dags_forder = /home/appadmin/workspace/airflow/dags
load_examples = False
secret_key = 비밀키
execution_api_server_url = http://127.0.0.1:8080/executiuon
auth_manager = airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
fernet_key = 비밀키

[database]
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@[localip]:5432/airflow_db_317

[api]
secret_key = 비밀키
auth_backends = airflow.providers.fab.auth_manager.api.auth.backend.session,airflow.providers.fab.auth_manager.api.auth.backend.basic_auth,airflow.api_auth.backend.default
workers = 4

[logging]
base_log_folder = /home/appadmin/workspace/airflow/logs

[api_auth]
jwt_issuer = airflow
jwt_secret = 비밀키

[celery]
broker_url = redis://:appadmin@127.0.0.1:6379/0
#broker_url = sentinel://:appadmin@10.10.10.10:26379/0;sentinel://:appadmin@10.10.10.11:26379/0;sentinel://:appadmin@10.10.10.12:26379/0;
result_backend = db+postgresql://airflow:airflow@[localip]:5432/airflow_db_317
worker_concurrency = 16

[celery_broker_transport_options]
master_name = mymaster
```


# systemd service 등록
``` shell
sudo tee /etc/systemd/system/airflow-api-server.service <<EOF
[Unit]
Description=Airflow API Server Daemon
After=network.target postgresql.service

[Service]
User=appadmin
Group=appadmin
Type=simple
Environment="AIRFLOW_HOME=/home/appadmin/airflow"
# 가상환경의 bin을 PATH 맨 앞에 두어 activate 없이 실행되게 함
Environment="PATH=/home/appadmin/airflow/airflow_env/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"
ExecStart=/home/appadmin/airflow/airflow_env/bin/airflow api-server
Restart=on-failure
RestartSec=10s
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

``` shell
sudo tee /etc/systemd/system/airflow-scheduler.service <<EOF
[Unit]
Description=Airflow Scheduler Daemon
After=network.target postgresql.service

[Service]
User=appadmin
Group=appadmin
Type=simple
Environment="AIRFLOW_HOME=/home/appadmin/airflow"
Environment="PATH=/home/appadmin/airflow/airflow_env/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"
ExecStart=/home/appadmin/airflow/airflow_env/bin/airflow scheduler
Restart=on-failure
RestartSec=10s
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

``` shell
sudo tee /etc/systemd/system/airflow-dag-processor.service <<EOF
[Unit]
Description=Airflow DAG Processor Daemon
After=network.target postgresql.service

[Service]
User=appadmin
Group=appadmin
Type=simple
Environment="AIRFLOW_HOME=/home/appadmin/airflow"
Environment="PATH=/home/appadmin/airflow/airflow_env/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"
ExecStart=/home/appadmin/airflow/airflow_env/bin/airflow dag-processor
Restart=on-failure
RestartSec=10s
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

``` shell
sudo tee /etc/systemd/system/airflow-triggerer.service <<EOF
[Unit]
Description=Airflow Triggerer Daemon
After=network.target postgresql.service

[Service]
User=appadmin
Group=appadmin
Type=simple
Environment="AIRFLOW_HOME=/home/appadmin/airflow"
Environment="PATH=/home/appadmin/airflow/airflow_env/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"
ExecStart=/home/appadmin/airflow/airflow_env/bin/airflow triggerer
Restart=on-failure
RestartSec=10s
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

``` shell

# 1. 데몬 리로드 (새로 만든 파일 인식)
sudo systemctl daemon-reload

# 2. 서비스 시작 및 부팅 시 자동 실행 등록
sudo systemctl enable --now airflow-api-server
sudo systemctl enable --now airflow-scheduler
sudo systemctl enable --now airflow-dag-processor
sudo systemctl enable --now airflow-triggerer

```


# 트러블 슈팅 
## local에서 공유기를 사용 시 sql_alchemy_conn
```
아래 출력되는 ip를 넣고 사용한다.
ip route show | grep default | awk '{print $3}' 
```
## wsl -> local ping 안나갈 때
``` powershell

Get-NetAdapter | Where-Object { $_.Name -like "*WSL*" } | Select-Object Name
위에서 출력한 값을 아래에 넣는다
New-NetFirewallRule -DisplayName "WSL2 Inbound" -InterfaceAlias "위에서확인한이름" -Direction Inbound -Action Allow
New-NetFirewallRule -DisplayName "WSL2 Inbound" -InterfaceAlias "vEthernet (WSL (Hyper-V firewall))" -Direction Inbound -Action Allow
```

## WEB 접속 시 JWT token is not valid: Signature verification failed 에러
```
아래 정보 추가 OR 확인
[webserver]
secret_key = def2a1e5f0535f2390be793e90fb0ca957d7a031356b982148755fdab698
```

## TypeError: Issuer (iss) must be a string 
```
pip show PyJWT
# 여기서 Version: 2.11.0 이 나오면 100% 이 문제입니다.

pip install "PyJWT==2.10.1"
```

## postgresql 커넥션이 없음
<img width="922" height="372" alt="image" src="https://github.com/user-attachments/assets/5b89b357-fef4-40ad-8565-22c1426645b9" />

```
((airflow_env) ) [appadmin@DESKTOP-3U0O7D9 airflow]$ pip install apache-airflow-providers-postgres

((airflow_env) ) [appadmin@DESKTOP-3U0O7D9 airflow]$ pip list | grep postgres
apache-airflow-providers-postgres        6.5.3
```




