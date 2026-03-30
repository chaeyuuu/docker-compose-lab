# 🐋 Docker Compose HealthCheck 실습

> **Spring Boot + MySQL** 환경에서 Docker Compose 헬스체크 검증 실습

<br/>

## ⚙️ 실습 환경

| 항목 | 내용 |
|------|------|
| OS | Ubuntu |
| Docker Compose | v2 |
| Spring Boot | step04_empApp (app.jar) |
| DB | MySQL 8.0 |
| App 포트 | 8083 |
| 엔드포인트 | `/emp` |

<br/>

## 🖥️ 프로젝트 구조

```
healthcheck-lab/
├── app/
│   ├── app.jar        ← Spring Boot 실행 파일
│   └── Dockerfile     ← App HEALTHCHECK
└── docker-compose.yml ← DB HEALTHCHECK
```

<br/>

## ✒️ 설정 파일

### 📍 docker-compose.yml

```yaml
services:
  db:
    image: mysql:8.0
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - spring-mysql-net
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -p$$MYSQL_ROOT_PASSWORD || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  app:
    build: ./app
    ports:
      - "8083:8083"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/fisa
      SPRING_DATASOURCE_USERNAME: user01
      SPRING_DATASOURCE_PASSWORD: user01
    networks:
      - spring-mysql-net
    depends_on:
      db:
        condition: service_healthy

volumes:
  db_data:
networks:
  spring-mysql-net:
```

<br/>

#### DB healthcheck 파라미터

| 파라미터 | 값 | 의미 |
|---|---|---|
| `test` | `mysqladmin ping` | MySQL이 실제 쿼리를 수락할 수 있는지 확인 |
| `interval` | 10s | 10초마다 검사 |
| `timeout` | 5s | 5초 안에 응답 없으면 실패 |
| `start_period` | 30s | 컨테이너 기동 후 30초간 실패 무시 (InnoDB 초기화 여유) |
| `retries` | 10 | 10회 연속 실패 시 unhealthy |

<br/>

#### depends_on condition

```yaml
depends_on:
  db:
    condition: service_healthy
```

| condition | 의미 |
|---|---|
| `service_started` | DB 컨테이너가 시작만 되면 App 기동 |
| `service_healthy` | DB 헬스체크가 통과되어야 App 기동 |

<br/>

### 📍 Dockerfile

```dockerfile
FROM eclipse-temurin:17-jdk-alpine
RUN apk add --no-cache curl
WORKDIR /app
COPY app.jar app.jar

HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8083/emp || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

<br/>

#### App healthcheck 파라미터

| 파라미터 | 값 | 의미 |
|---|---|---|
| `--interval` | 10s | 10초마다 검사 |
| `--timeout` | 3s | 3초 안에 응답 없으면 실패 |
| `--retries` | 3 | 3회 연속 실패 시 unhealthy |
| `CMD` | `curl -f /emp` | HTTP 200 응답 여부로 App 생존 확인 |

<br/>

## 🔎 Health Check 테스트

### 1️⃣ Health Check O

```bash
docker compose up -d
docker ps
```

**결과**

<img width="1403" height="399" alt="스크린샷 2026-03-30 105714" src="https://github.com/user-attachments/assets/5f53e394-4199-4d7b-a422-7aed680242ff" />

<br/>

```
NAME                     STATUS
healthcheck-lab-db-1     Up 2 minutes (healthy)
healthcheck-lab-app-1    Up 14 seconds (healthy)
```

- DB가 `healthy` 상태가 된 이후 App 컨테이너가 기동됨
- App도 `/emp` 체크를 통과하여 `healthy` 전환
- `health: starting` 에서 `healthy` 전환 과정 `docker ps`을 통해 확인

**작동 순서**
```
DB 컨테이너 시작
    ↓ (mysqladmin ping 통과, ~30초)
DB: healthy
    ↓ (condition: service_healthy 충족)
App 컨테이너 시작
    ↓ (JVM 기동 ~8초, /emp 응답 확인)
App: healthy
```

<br/>


### 2️⃣ Health Check X

`docker-compose.yml`에서 `depends_on` 블록 전체 주석처리 후 재기동:

```bash
docker compose down
docker volume rm healthcheck-lab_db_data  # 볼륨 제거로 MySQL 초기화 시간 증가
docker compose up -d
docker logs healthcheck-lab-app-1
```

**결과 — Connection Error 발생**

<img width="1557" height="119" alt="스크린샷 2026-03-30 111851" src="https://github.com/user-attachments/assets/fa68a51a-2dee-4bd8-92b2-90ffc402e94b" />

<br/>

```
ERROR: Communications link failure
WARN:  Could not obtain connection to query metadata
ERROR: JDBCConnectionException: unable to obtain isolated JDBC connection
```

**원인**
- `depends_on` 없이 기동하면 DB와 App이 동시에 시작됨
- MySQL 초기화가 완료되기 전에 App이 DB 연결 시도
- app 컨테이너가 기동을 시도하지만 DB 연결 실패로 인해 애플리케이션 프로세스가 즉시 종료됨

<br/>

### 3️⃣ DB 강제 종료 후 App 동작 확인

```bash
docker stop healthcheck-lab-db-1
docker ps
curl http://localhost:8083/emp/
```

**결과**
- App 컨테이너는 `healthy` 상태 유지
- `/emp` 엔드포인트는 계속 302 응답 반환

**원인**
> `/emp`는 DB 연결 상태를 확인하지 않는 엔드포인트로 
> DB가 죽어도 Spring Boot 프로세스 자체는 살아있으므로 HTTP 응답이 가능.
> 따라서 App HEALTHCHECK가 DB 장애를 감지하지 못하는 한계가 존재하기에
> 실무에서는 `/actuator/health`처럼 DB 연결 상태를 포함하는 엔드포인트를 사용해야 함

<br/>


## 📖 결과 정리

| 항목 | 헬스체크 O | 헬스체크 X |
|------|----------------------|----------------------|
| DB healthcheck | `mysqladmin ping` | 없음 |
| App 시작 조건 | DB `healthy` 이후 | 즉시 (동시 가동) |
| 기동 시 오류 | 없음 | Connection Error |
| `docker ps` 상태 표시 | `(healthy)` 표시 | 상태 없음 |

- Docker는 컨테이너가 실행 중이어도 내부 서비스가 실제로 준비됐는지는 알 수 없음
- 헬스체크를 통해 프로세스 생존 여부와 처리 상태 여부를 검사함으로써 의존 서비스가 완전히 준비된 이후에 다음 컨테이너가 기동되도록 순서 보장

<br/>

## 🚀 트러블 슈팅

### 1. depends_on 제거 후에도 Connection Error가 재현되지 않음

**원인**

- docker compose down 후에도 볼륨(db_data)이 남아있으면 MySQL이 재기동 시 기존 데이터를 그대로 사용하여 초기화 과정을 건너뜀
- 이 경우 DB 기동 시간이 매우 짧아져 App보다 먼저 준비되면서 우연히 연결에 성공

**해결**

```bash
docker compose down
docker volume rm healthcheck-lab_db_data  # 볼륨 제거 → MySQL 최초 초기화 강제
docker compose up -d
docker logs healthcheck-lab-app-1
```
- 볼륨을 제거하면 MySQL이 시스템 테이블, InnoDB 엔진, 사용자/DB 생성 등 최초 초기화를 수행하느라 20~30초가 소요
- 이 사이에 App이 먼저 DB 연결을 시도하면서 Communications link failure 발생 확인 가능
