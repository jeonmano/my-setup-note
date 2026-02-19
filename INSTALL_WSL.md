# WSL2 배포판 관리 및 다중 인스턴스 구성 가이드

본 문서는 Windows Subsystem for Linux(WSL2) 환경에서 배포판의 설치, 완전 삭제, 그리고 동일한 OS를 여러 개 복제하여 독립된 환경으로 구성하는 과정을 다룹니다.

---

## 1. 배포판 설치 (Installation)
온라인 리포지토리에서 원하는 리눅스 배포판을 찾아 설치합니다.

```powershell
# 1. 설치 가능한 배포판 목록 확인
wsl -l -o

# 2. 특정 배포판 설치 (예: Ubuntu-22.04)
wsl --install -d Ubuntu-22.04
```

---

## 2. 배포판 삭제 (Unregister)
더 이상 사용하지 않는 배포판을 등록 해제하고 내부 데이터를 모두 삭제합니다.

```powershell
# 1. 현재 설치된 배포판 및 상태 확인
wsl -l -v

# 2. 특정 배포판 완전 삭제 (주의: 복구 불가능)
wsl --unregister <배포판이름>
```

---

## 3. 동일 OS 다중 인스턴스 구성 (Import/Export)
동일한 환경을 복제하여 여러 개의 독립된 서버 환경을 구성할 때 사용합니다.

```powershell
# 1. 기존 배포판 내보내기 (이미지 추출)
# wsl --export <원본이름> <저장경로\파일명.tar>
wsl --export OracleLinux_9_5 C:\WSL_Backup\oracle_base.tar

# 2. 새로운 이름으로 가져오기 (인스턴스 복제)
# wsl --import <새이름> <설치경로> <이미지파일경로>
wsl --import Oracle_Svr_01 C:\WSL\Svr01 C:\WSL_Backup\oracle_base.tar
wsl --import Oracle_Svr_02 C:\WSL\Svr02 C:\WSL_Backup\oracle_base.tar
```

---

## 4. 인스턴스 관리 및 접속 (Management)
설치된 배포판에 접속하거나 네트워크 상태를 관리합니다.

```powershell
# 1. 특정 인스턴스의 홈 디렉토리로 즉시 접속
wsl -d Oracle_Svr_01 --cd ~

# 2. 인스턴스 내부 IP 확인 (통신 테스트용)
# (리눅스 접속 후 입력)
hostname -I

# 3. 실행 중인 모든 WSL 종료 (리소스 확보)
wsl --shutdown
```

---

## 5. 주요 관리 명령어 요약 (Cheat Sheet)

| 기능 | 명령어 | 설명 |
| :--- | :--- | :--- |
| 목록 확인 | `wsl -l -v` | 설치된 OS 목록 및 버전/상태 확인 |
| 기본값 설정 | `wsl -s <이름>` | 기본 실행 배포판 변경 |
| 강제 종료 | `wsl -t <이름>` | 특정 배포판만 즉시 종료 |
| 버전 변경 | `wsl --set-version <이름> 2` | WSL1에서 WSL2로 변환 |


## 6. 방화벽
```
# WSL에서 윈도우 PostgreSQL(5432)로 접속할 수 있도록 허용하는 규칙 추가
New-NetFirewallRule -DisplayName "Allow WSL to Postgres" -Direction Inbound -LocalPort 5432 -Protocol TCP -Action Allow
```
