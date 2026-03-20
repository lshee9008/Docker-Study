# 1. .dockerignore활용

- .gitignore와 완벽히 똑같은 역할을 한다.
- docker build 명령어를 실행할 때, 도커 데몬(엔진)으로 파일들을 전송(Build Context)하는데, 이때 불필요한 파일이 복사되는 것을 막아줌
- 왜 필요한가?
    - 빌드 속도 향상: 불필요한 파일을 도커 엔진으로 전송하지 않아 빌드가 빨라짐
    - 보안 유지: 실수로 .env 같은 비밀키나 .git 폴더의 히스토리가 이미지에 포함되는 것을 막아줌
    - 캐시 효율: 소스 코드가 아닌 파일(예: 로컬 로그 파일)이 변경되었다고 해서 도커가 전체 이미지를 다시 빌드하는 낭비를 막는다.
- .dockerignore 예시
    
    ```
    # Git
    .git
    .gitignore
    
    # Python
    __pycache__/
    *.py[cod]
    *$py.class
    .venv/
    venv/
    env/
    
    # 환경변수 및 로컬 파일
    .env
    *.log
    Dockerfile
    docker-compose.yml
    ```
    

# 2. 이미지 경량화 (Multi-stage Build)

- 컨테이너를 빌드할 때 쓰이는 도구들(예: C 컴파일러, gcc 등)은 막상 프로그램이 실행될 때는 필요가 없다.
- 멀티 스테이지 빌드는 “요리하는 주방(빌드 환경)”과 “음식이 서빙되는 식탁(실행 환경)”을 분리하는 기술
    - 동작 방식: 하나의 Dockerfile 안에 FROM 명령어를 여러 개 사용
    - 이전 단계에서 생성된 핵심 결과물(빌드된 파일이나 패키지)만 다음 단계로 복사(COPY —from- …)해 옴
- Multi-stage Build 예시(FastAPI)
    
    ```docker
    # Stage 1: Builder (요리하는 주방)
    FROM python:3.11-slim AS builder
    
    WORKDIR /app
    
    # 컴파일에 필요한 필수 패키지 설치 (예: 빌드 도구)
    RUN python -m venv /opt/venv
    ENV PATH="/opt/venv/bin:$PATH"
    
    COPY requirements.txt
    RUN pip install --no-cache-dir -r requirements.txt
    
    # Stage 2: Runner (음식이 서빙되는 식탁 - 최종 이미지)
    FROM python:3.11-slim
    
    WORKDIR /app
    
    # Builder 단계에서 완성된 가상환경 폴더만 쏙 빼온다. (gcc 같은 무거운 도구는 버려짐)
    COPY --from=builder /opt/venv /opt/venv
    ENV PATH="/opt/venv/bin:$PATH"
    
    # 실제 애플리케이션 소스 코드 복사
    COPY . .
    
    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```
    
    - 이렇게 하면 수백 MB를 차지하던 빌드 도구들이 싹 빠져서 최종 이미지 용량이 획기적으로 줄어듬

# 3. 보안 최적화 (Non-root user)

- 도커 컨테이너는 기본적으로 root (최고 관리자) 권한으로 실행
- 만약 해커가 앱의 취약점을 뚫고 컨테이너 내부로 들어온다면? 컨테이너 안에서 무소불위의 권력을 휘두를 수 있다.
- 이를 막기 위해 제한된 권한을 가진 일반 사용자(Non-root)를 만들어 앱을 실행
- Runner Stage에 Non-root 적용하기
    
    ```docker
    # ... (위의 Stage 2 코드에 이어짐) ...
    
    # 1. 'appuser'라는 이름의 시스템 그룹과 유저 생성 (비밀번호 없음)
    RUN addgroup --system appuser && adduser --system --group appuser
    
    # 2. 작업 디렉토리의 소유권을 appuser로 변경
    RUN chown -R appuser:appuser /app
    
    # 3. 이제부터 실행되는 명령어는 모두 'appuser' 권한으로 실행됨
    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```
    
    - 이제 컨테이너가 털리더라도 해커는 appuser 권한밖에 없으므로 치명적인 시스템 파일 수정 등을 할 수 없게 됨.

# Healthcheck 및 Depends_on

- docker-compose 로 FastAPI, DB, Ollama를 띄울 때 가장 흔히 겪는 에러가 “FastAPI가 켜지면서 DB에 연결하려고 했는데, DB가 아직 덜 켜져서 연결 실패로 FastAPI가 죽어버리는 현상”
- 단순히 depends_on만 쓰면 “DB 컨테이너가 시작되었는지”만 확인(컨테이너는 커졌지만, DB 시스템 자체가 부팅되려면 몇 초 더 걸림)
- 그래서 healthcheck로 “DB나 Ollama가 실제로 요청을 받을 준비가 되었는지” 검사하고, 준비가 완료되었을 때 FastAPI를 켜야 함
    
    ```yaml
    version: '3.8'
    
    services:
    	ollama:
    		image: ollama/ollama
    		ports:
    			- "11434:11434"
    		# 핵심 1: Ollama가 제대로 응답하는지 5초마다 찔러봄
    		healthcheck:
    			test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
    			interval: 5s
    			timeout: 3s
    			retries: 5
    			
    	postgres-db:
    		image: postgres:15
    		environment:
    			POSTGRES_USER: user
    			POSTGRES_PASSWORD: password
    			
    		# 핵심 2: DB가 연결을 수락할 준비가 되었는지 검사
    		healthcheck:
    			test: ["CMD-SHELL", "pg_isready -U user"]
    			interval: 5s
    			timeout: 5s
    			retries: 5
    	
    	fastapi-app:
    		build: .
    		ports:
    			- "8000:8000"
    			
    		# 핵심 3: 그냥 켜지는게 아니라, 위 서비스들이 'healthy' 상태가 되면 켜짐
    		depends_on:
    			ollama:
    				condition: service_healthy
    			postgres-db:
    				condition: service_healthy
    ```
