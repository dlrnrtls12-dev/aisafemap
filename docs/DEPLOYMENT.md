# DEPLOYMENT

## 문서 상태

이 문서는 MVP 1차 구현을 위한 배포 설계입니다. Dockerfile과 Compose 파일이 생성되기 전에는 아래 명령이 동작하지 않을 수 있으며, 구현 진행에 맞춰 갱신합니다.

## 목표 구성

Docker Compose로 다음 서비스를 실행합니다.

- `frontend`: React + TypeScript + Vite
- `backend`: FastAPI
- `db`: PostgreSQL + PostGIS

Redis는 1차 구현의 필수 구성요소가 아닙니다. 여러 백엔드 인스턴스에서 WebSocket을 운영할 필요가 생길 때 별도 검토합니다.

## 필요한 도구

- Git
- Docker
- Docker Compose

Docker를 사용하지 않고 직접 실행할 때만 Python과 Node.js를 별도로 설치합니다.

## 환경변수

저장소에는 실제 값이 없는 `.env.example`만 커밋합니다. 실제 `.env`는 Git에 추가하지 않습니다.

### Backend

```dotenv
DATABASE_URL=postgresql+psycopg://aisafemap:change-me@db:5432/aisafemap
TOKEN_HMAC_KEY=replace-with-secret
WS_TOKEN_SECRET_KEY=replace-with-secret
DATA_ENCRYPTION_KEY=replace-with-secret
WEATHER_API_KEY=
FIRE_SAFETY_DATA_API_KEY=
ALLOWED_ORIGINS=http://localhost:5173
DEMO_DATA_MODE=true
LOG_LEVEL=INFO
```

### Frontend

```dotenv
VITE_API_BASE_URL=http://localhost:8000/api/v1
VITE_WS_BASE_URL=ws://localhost:8000/ws
```

`VITE_`로 시작하는 값은 브라우저에 공개됩니다. 비밀키를 넣지 않습니다.

## 로컬 실행 계획

Compose 파일이 구현된 후 저장소 루트에서 실행합니다.

```text
docker compose up --build
```

예정된 기본 주소:

- Frontend: `http://localhost:5173`
- Backend API: `http://localhost:8000`
- API 문서: `http://localhost:8000/docs`

로컬 개발에서만 HTTP와 WS를 허용합니다.

## 데이터베이스 초기화

PostGIS 컨테이너 이미지 또는 초기화 SQL로 확장을 활성화합니다.

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

마이그레이션 실행 계획:

```text
docker compose exec backend alembic upgrade head
```

시연 데이터 적재 계획:

```text
docker compose exec backend python -m app.utils.seed_demo
```

시연 데이터 적재는 `DEMO_DATA_MODE=true`에서만 허용합니다.

## WebSocket 연결 흐름

1. 클라이언트가 역할 토큰을 Bearer 헤더에 넣어 `POST /api/v1/incidents/{id}/ws-token`을 요청합니다.
2. 서버가 사고 상태와 권한을 확인합니다.
3. 서버가 짧은 유효기간의 일회용 `ws_token`을 발급합니다.
4. 브라우저가 `wss://HOST/ws/incidents/{id}`에 연결하면서 `Sec-WebSocket-Protocol`로 단기 토큰을 전달합니다.
5. 서버가 토큰을 한 번 사용한 것으로 처리합니다.

장기 역할 토큰을 WebSocket URL에 넣지 않습니다.

## 외부 공공데이터 연동 순서

1. 소방안전 빅데이터 플랫폼 API
2. 기상청 단기예보·초단기실황·기상특보 API
3. 공공데이터 또는 검증된 방식의 일출·일몰 정보
4. 산림청·공공데이터포털의 산·등산로 정보
5. 국내 데이터가 부족할 때만 보조 외국 API

초기 구현은 `DEMO_DATA_MODE=true`로 시작하고, API 명세와 이용조건을 확인한 뒤 서버 연동 모듈을 하나씩 추가합니다.

## 배포 환경 보안

- HTTPS와 WSS를 필수로 사용합니다.
- TLS 종료는 Nginx, Traefik 또는 클라우드 로드밸런서에서 수행할 수 있습니다.
- 실제 비밀값은 GitHub Secrets 또는 배포 환경의 Secret Manager로 주입합니다.
- CORS는 실제 프론트엔드 주소만 허용합니다.
- 데이터베이스 포트를 인터넷에 직접 공개하지 않습니다.
- 백업을 암호화하고 접근권한을 제한합니다.

## CI 계획

GitHub Actions에서 다음을 실행합니다.

### Backend

- 코드 스타일과 정적 검사
- Pytest
- 마이그레이션 검증

### Frontend

- 의존성 설치
- TypeScript 검사
- 기본 컴포넌트 테스트
- 프로덕션 빌드

실제 API 키를 CI 로그에 출력하지 않습니다.

## 시연 전 점검

1. `DEMO_DATA_MODE=true` 확인
2. Docker Compose 전체 서비스 실행 성공
3. 마이그레이션과 시연 데이터 적재 성공
4. 사고 생성과 세 역할 링크 발급
5. 신고자 위치·상태 전송
6. 소방대원 지도에서 신고자 위치 확인
7. 기상·산·시설 시연 데이터 표시
8. 위험도와 장비 추천 표시
9. 두 브라우저 간 문자 채팅
10. 지휘관의 사고 종료
11. 종료 후 새 위치·상태·채팅 차단
12. 종료된 사고기록 조회

## 운영 전 추가 결정사항

- 역할 토큰의 최대 유효기간과 갱신 정책
- WebSocket 단기 토큰 유효기간
- 관리자 인증과 2단계 인증 적용 여부
- 건강정보·정밀위치 암호화 키 관리 방식
- 감사로그와 사고기록의 법정 보존기간
- 외부 API 호출 한도와 장애 대응 기준
