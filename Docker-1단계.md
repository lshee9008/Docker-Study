# 학습할 키워드들
**[핵심 기술 및 스택]**
#Docker #도커 #FastAPI #Python #파이썬 #백엔드 #Backend #AI백엔드 #서버개발

**[도커 핵심 개념]**
#가상화 #컨테이너 #Container #도커이미지 #DockerImage #도커컨테이너 #포트포워딩 #PortForwarding #호스트 #Host

**[도커 설정 및 명령어]**
#Dockerfile #도커명령어 #DockerCLI #docker_build #docker_run #docker_pull #docker_ps

**[파이썬 및 실행 환경]**
#Uvicorn #requirements_txt #개발환경세팅 #로컬환경

# 1. Docker 개론: 가상머신(VM)과 컨테이너의 차이

### 가상머신(Virtual Machine) 의 차이

- 가상머신 (VM)
    - 컴퓨터 안에 또 다른 완전한 컴퓨터를 만드는 것
    - Windows 컴퓨터 안에 가상의 공간을 뚝 떼어내어 Linux OS(운영체제)를 통째로 설치
    - 무겁고, 켜는 데 오래 걸리며, 자원(메모리, CPU)을 많이 차지
- 도커 컨테이너 (Docker Container)
    - OS 전체를 새로 설치하지 않음
    - 내 컴퓨터의 OS 커널(핵심 기능)은 그대로 공유하면서, 애플리케이션(ex: FastAPI)과 그 실행에 필요한 라이브러리(Python 등)만 딱 격리해서 실행
    - 훨씬 가볍고, 눈 깜짝할 새 켜지며, 자원을 효율적으로 씀

# 2. 도커 필수 명령어 숙지

- 도커는 명령어(CLI) 기반으로 작동
- 가장 많이 쓰는 7가지 명령어가 있음
- 여기서 ‘이미지(Image)’는 프로그램 설치 파일(또는 붕어빵 틀), 컨테이너(Container)’는 설치되어 실행 중인 프로그램(구어진 붕어빵) 이라고 생각하면 됨

| **명령어** | **설명** | **비고** |
| --- | --- | --- |
| `docker pull [이미지명]` | 도커 허브(저장소)에서 이미지를 다운로드 | 예: `docker pull python:3.11` |
| `docker images` | 내 컴퓨터에 다운로드된 이미지 목록을 보여줌 |  |
| `docker run [옵션] [이미지명]` | 이미지를 기반으로 **컨테이너를 생성하고 실행** | 도커의 꽃이라고 할 수 있는 명령어 |
| `docker ps` | 현재 실행 중인 컨테이너 목록을 보여줌 | `-a` 옵션을 붙이면 종료된 것까지 다 보여줌 |
| `docker stop [컨테이너ID]` | 실행 중인 컨테이너를 부드럽게 종료(정지) |  |
| `docker rm [컨테이너ID]` | 정지된 컨테이너를 삭제 |  |
| `docker rmi [이미지명]` | 더 이상 필요 없는 이미지를 삭제하여 용량을 확보 |  |

# 3. FastAPI 기초 앱 작성

- 도커에 올릴 아주 간단한 FastAPI 코드를 준비
1. requirements.txt (설치할 파이썬 패키지 목록)
    
    ```
    fastapi
    uvicorn
    ```
    
2. [main.py](http://main.py) (FastAPI 서버 코드)
    
    ```python
    from fastapi import FastAPI
    
    app = FastAPI()
    
    @app.get("/hello")
    def read_root():
    	return {"messagae" : "안녕하세요! 도커 안에서 돌아가는 FastAPI입니다,"}
    ```
    

# 4. Dockerfile (도커 이미지 레시피)

- 위의 파이썬 코드를 도커 ‘이미지’로 구워내기 위한 레시피를 작성
- 파일 이름을 확장자 없이 Dockerfile로 지정

```docker
# 1. 베이스 이미지 지정: 파이썬 3.11 버전의 미리 설치된 환경을 가져옴
FROM python:3.11

# 2. 작업 디렉토리 설정: 컨테이너 내부에서 '/app'이라는 폴더를 기본 작업 공간으로 사용
WORKDIR /app

# 3. 파일 복사: 내 컴퓨터(현재 폴더)에 있는 requirements.txt와 main.py를 컨테이너의 /app 폴더로 복사
COPY requirements.txt
COPY main.py

# 4. 명령어 실행: 복사된 requirements.txt를 읽어 필요한 라이브러리를 설치
RUN pin install --no-cache-dir -r requirements.txt

# 5. 실행 명령: 컨테이너가 켜질 때 마지막으로 실행할 명령어 (uvicorn으로 FastAPI 서버 켜기)
# 외부에서 접속할 수 있도록 --hosts 0.0.0.0으로 설정
CMd ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

# 5. 빌드(Build) 및 실행(Run)

- 이제 준비된 레시피(Dockerfile)로 이미지를 만들고 실행해 봄
- 터미널(명령 프롬프트)을 열고 해당 폴더로 이동 한 뒤 아래 명령어를 차례로 입력
1. 이미지 빌드하기

```bash
docker build -t my-fastapi-app .
```

- -t my-fastapi-app
    - 만들어질 이미지의 이름을 ‘my-fastapi-app’으로 짓겠다는 뜻(태그 기능)
    - . (마침표): Dockerfile이 있는 위치가 현재 폴더라는 뜻.(이 마침표를 빼먹는 실수가 잦으니 주의!)
1. 컨테이너 실행 및 포트 포워딩

```bash
docker run -p 8000:8000 my-fastapi-app
```

- -p 8000:8000
    - [내 컴퓨터(호스트) 포트]: [컨테이너 내부 포트]
    - 내 컴퓨터의 8000번 포트로 들어오는 접속을 컨테이너 내부의 8000번 포트로 연결(포워딩)해 주겠다는 뜻
    - 이것이 없으면 컨테이너가 아무리 돌아가도 외부(내 브라우저)에서 접속할 수 없음.
1. 접속 확인
- 웹 브라우저를 열고 주소창에 http://localhost:8000/hello를 입력하여 {”message”: “안녕하세요! 도커 안에서 돌아가는 FastAPI입니다.”}라는 메시지가 뜨면 1단계 완벽 성공.
