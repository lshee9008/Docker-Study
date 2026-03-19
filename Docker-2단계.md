# 1. Docker Compose 입문: 다중 컨테이너

- 지금까지 docker run 명령어로 컨테이너를 하나씩 실행했다면, 서비스가 복잡해질수록(웹 서버, DB, 캐시 서버 등) 매번 긴 명령어를 치는 것은 비효율적.
- Docker Compose는 여러 컨테이너의 실행 옵션을 하나의 파일(docker-compose.yml)에 적어두고 한 번에 실행 및 종료할 수 있게 해주는 도구
    - docker-compose.yml: 어떤 이미지를 사용할지, 포트는 어떻게 연결할지, 환경변수는 무엇인지 명시하는 ‘설계도’
    - docker-compose up -d: 설계도에 적힌 모든 컨테이너를 백그라운드(-d)에서 한 번에 실행
    - docker-compose down: 실행된 모든 컨테이너와 기본 네트워크를 깔끔하게 정리하고 종료

# 2. PostgreSQL 컨테이너 띄우기 & 3. 데이터 볼륨(Volume) 마운트

- 데이터베이스를 컨테이너로 띄울 때 가장 큰 문제는 “컨테이너가 삭제되면 안에 저장된 데이터도 함께 날아간다”는 것이다.
- 이를 방지하기 위해 컨테이너 내부의 데이터 저장 경로를 호스트 PC(내 컴퓨터)의 특정 공간과 연결하는 볼륨(Volume) 마운트가 필수적
- 아래는 DB와 볼륨을 설정하는 docker-compose.yml의 예시
    
    ```yaml
    version: '3.8'
    
    services:
    	db:
    		image: postgres:15
    		container_name: my_postgres
    		restart: always
    		environment:
    			POSTGRES_USER: myuser
    			POSTGRES_PASSWORD: mypassword
    			POSTGRES_DB: mydatabase
    		ports:
    			- "5432:5432"
    		volumes:
    			- postgres_data:/var/lib/postgresql/data # 볼륨 마운트 핵심 설정
    
    volumes:
    	postgres_data: # 도커가 관리하느 이름 있는 볼륨(Named Volume) 생성
    ```
    
- 환경변수(ENV): environment 항목을 통해 PostgreSQL 초기 세팅(유저명, 비밀번호, DB명)을 자동으로 진행
- 데이터 영속성(Persistence): 컨테이너 내부의 /var/lib/postgresql/data(PostgreSQL이 데이터를 저장하는 기본 경로)를 도커가 관리하는 postres_data라는 안전한 공간에 연결
- 이제 docker-compose down으로 DB 컨테이너를 지웠다 다시 띄어도 데이터는 그대로 유지

# 4. 컨테이너 간 통신 (Docker Network)

- 이 부분이 로컬 개발 환경과 도커 환경의 가장 큰 차이점
- 로컬에서 FastAPI를 실행할 때는 DB 주소를 [localhost:5432](http://localhost:5432) 로 설정하지만, 도커 환경에서는 FastAPI 컨테이너와 DB 컨테이너가 각각 독립된 가상 환경(마치 다른 컴퓨터)에 존재
- 따라서 FastAPI에서 localhost를 호출하면 DB가 아닌 FastAPI 컨테이너 자기 자신을 가리키게 되어 연결에 실패
    - 해결책: Docker Compose로 묶인 컨테이너들은 자동으로 같은 가상 네트워크를 공유
    - 이 네트워크 안에서는 서비스 이름(위 예시의 경우 db)이 곧 호스트 주소(IP) 역할을 함

# 5. FastAPI + DB 연동 프로젝트(SQLAlchemy 적용)

```yaml
version: '3.8'

services:
	api:
		build: . # 현재 디렉토리의 Dockerfile을 기반으로 빌드
		container_name: fastapi_app
		ports:
			- "8000:8000"
		depends_on:
			- db # db 컨테이너가 먼저 실행된 후 api 컨테이너 실행
		environment:
			# DB 접속 주소 : localhost가 아니라 서비스명인 'db'를 사용
			- DATABASE_URL=postgresql://myuser:mypassword@db:5432/mydatabase
	db:
		image: postgres:15
		container_name: my_postgres
		environment:
			POSTGRES_USER: myuser
			POSTGRES_PASSWORD: mypassword
			POSTGRES_DB: mydatabase
		volumes:
			- postgres_data:/var/lib/postgresql/data

volumes:
	postgres_data:
```

### SQLAlchemy에서의 접속 설정

- FastAPI 내부의 파이썬 코드(SQLAlchemy)에서는 환경변수로 주입받은 DATABASE_URL을 사용하여 DB 엔진을 생성하게 됨

```python
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# docker-compose에서 넘겨준 환경변수 읽어옴
SQLALCEHMY_DATABASE_URL = os.getenv("DATABASE_URL", "postfresql://myuser:mypassword@localhost/mydatabase")

# engine 생성 (이후 세션 및 모델 연동에 사용)
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

- 이 구조를 완성하면, 코드 수정 후 docker-compose up —build -d 명령어 하나로 API 서버와 DB를 완벽하게 연동하여 띄울 수 있게 됨.

# 실습

### 프로젝트 구조

```
my-fastapi-project/
 ├── requirements.txt
 ├── main.py
 ├── Dockerfile
 └── docker-compose.yml
```

1. requirements.txt (파이썬 패키지 목록)
    
    ```
    fastapi
    uvicorn
    sqlalchemy
    psycopg2-binary
    ```
    
2. [main.py](http://main.py) (FastAPI 애플리케이션 및 DB 연동)
    - 유저가 보낸 텍스트를 DB에 저장하고, 저장된 텍스트 목록을 조회하는 간단한 API
    
    ```python
    import os
    from fastapi import FastAPI, Depends
    from sqlalchemy import create_engine, Colume, Integer, String
    from sqlalchemy.orm import declarative_base, sessionmaker, Session
    from pydantic import BaseModel
    
    # 1. DB 설정 (환경변수에서 주소 가져오기)
    DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://myuser:mypassword@db:5432/mydatabase")
    
    engine = create_engine(DATABASE_URL)
    SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
    Base = declarative_basE()
    
    # 2. 데이터베이스 모델 (SQLAlchemy)
    class UserText(Base):
    	__tablename__ = "user_texts"
    	id = Column(Integer, primary_key=True, index=True)
    	content = Column(String, index=True)
    	
    # 테이블 생성 (실무에서는 Alembic 같은 마이그레이션 툴을 쓰지만, 입문 단계에선 이걸로 충분)
    Base.metabase.create_all(bind=engine)
    
    # 3. Pydantic 스키마 (데이터 검증용)
    class TextCreate(BaseModel):
    	content: str
    	
    # 4. FastAPI 앱 초기화 및 의존성 주입
    app = FastAPI()
    
    def get_db():
    	db = SessionLocal()
    	try:
    		yield db
    	finally:
    		db.close()
    		
    # 5. API 엔드포인트 구현
    @app.post("/texts/")
    def create_text(text: TextCreate, db: session = Depends(get_db)):
    	db_text = UserText(content=text.content)
    	db.add(db_text)
    	db.commit()
    	db.refresh(db_text)
    	return {"message": "저장 완료!", "data": db_text}
    
    @app.get("texts/")
    def reaad_texts(db: Session = Depends(get_db)):
    	texts = db.query(UserText).all()
    	return texts
    ```
    
3. Dockerfile (FastAPI 이미지를 만들기 위한 레시피)
    - 파이선 환경을 세팅하고 위에서 만든 코드를 실행하는 역할
    
    ```docker
    # 파이썬 3.11 버전을 기반으로 생성
    FROM python:3.11-slim
    
    # 컨테이너 내 작업 디렉토리 설정
    WORKDIR /app
    
    # 패키지 목록을 복사하고 설치
    COPY requirements.txt
    RUN pip install --no-cache-dir -r requirements.txt
    
    # 나머지 모든 코드 복사
    COPY . .
    
    # FastAPI 서버 실행 명령어 (0.0.0.0으로 열어야 외부에서 접근 가능)
    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```
    
4. docker-compose.yml (다중 컨테이너 지휘)
    - 앞서 설명해 드린 대로, API 서버와 DB 서버를 하나의 네트워크로 묶고 실행
        
        ```yaml
        version: '3.8'
        
        services:
        	# 1. FastAPI 애플리케이션 서비스
        	api:
        		build: .
        		container_name: fastapi_app
        		ports:
        			- "8000:8000"
        		depends_on:
        			- db
        		environment:
        			# DB 서비스 이름인 'db'를 호스트로 사용
        			- DATABASE_URL=postgresql://myuser:mypassword@db:5432/mydatabase
        	
        	# 2. PostgreSQL 데이터베이스 서비스
        	db:
        		image: postgres:15
        		container_name: my_postgres
        		restart: always
        		environment:
        			POSTGRES_USER: myuser
        			POSTGRES_PASSWORD: mypassword
        			POSTGRES_DB: mydatabase
        		ports:
        			- "5432:5432"
        		volumes:
        			- postgres_data:/var/lib/postgresql/data
        
        # 데이터 유지를 위한 볼륨 선언
        volumes:
        	postgres_data:
        ```
        

### 실행

1. 터미널을 열고 프로젝트 경로로 이동
2. docker-compose up —bulld -d 로 실행
3. 브라우저 → http://localhost:8000/docs 로 접속
4. FastAPI가 자동으로 마들어주는 Swagger UI 창
5. POST /texts/를 클릭하고 Try it out 버튼을 눌러 텍스트를 입력한 뒤 Excute 해보면 DB에 데이터가 들어감
6. GET /texts/ 를 실행해서 방금 넣은 데이터가 잘 나오는지 확이
7. docekr-compose down으로 종료
