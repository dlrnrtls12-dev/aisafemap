# DATABASE

## 목적

이 문서는 PostgreSQL + PostGIS를 사용해 사고, 위치, 상태, 채팅, 위험도, 장비 추천과 산악 지리정보를 저장하는 기준을 정의합니다.

## 기본 원칙

- 데이터베이스: PostgreSQL + PostGIS
- 기본 좌표계: WGS84, SRID 4326
- 기본 식별자: UUID
- 저장 시각: UTC, `TIMESTAMPTZ`
- 마이그레이션: Alembic
- 실제 개인정보를 시연 데이터에 사용하지 않습니다.

## PostGIS 초기화

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

`postgis_topology`는 1차 구현에 필요하지 않으며 실제 요구가 생길 때 추가합니다.

## 핵심 테이블

### incidents

사고방과 역할별 인증정보를 저장합니다.

- `id UUID PRIMARY KEY`
- `title TEXT NOT NULL`
- `status TEXT NOT NULL`: `open` 또는 `closed`
- `reporter_token_hash TEXT NOT NULL`
- `responder_token_hash TEXT NOT NULL`
- `commander_token_hash TEXT NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`
- `closed_at TIMESTAMPTZ NULL`
- `metadata JSONB NOT NULL DEFAULT '{}'`

권장 인덱스: `status`, `created_at`

### participants

사고 참여자의 역할과 현재 상태를 저장합니다.

- `id UUID PRIMARY KEY`
- `incident_id UUID NOT NULL REFERENCES incidents(id)`
- `role TEXT NOT NULL`: `reporter`, `responder`, `commander`
- `organization_type TEXT NULL`
- `current_location GEOMETRY(POINT, 4326) NULL`
- `location_accuracy_m DOUBLE PRECISION NULL`
- `battery_percent SMALLINT NULL`
- `last_seen_at TIMESTAMPTZ NULL`
- `extra JSONB NOT NULL DEFAULT '{}'`

권장 인덱스:

```sql
CREATE INDEX participants_incident_idx ON participants (incident_id);
CREATE INDEX participants_location_gist ON participants USING GIST (current_location);
```

### participant_locations

참여자의 위치 이동 이력을 저장합니다.

- `id UUID PRIMARY KEY`
- `incident_id UUID NOT NULL REFERENCES incidents(id)`
- `participant_id UUID NOT NULL REFERENCES participants(id)`
- `location GEOMETRY(POINT, 4326) NOT NULL`
- `accuracy_m DOUBLE PRECISION NOT NULL`
- `captured_at TIMESTAMPTZ NOT NULL`
- `received_at TIMESTAMPTZ NOT NULL`

권장 인덱스: `incident_id, captured_at`, 위치 GIST 인덱스

### reporter_statuses

신고자의 건강·부상 상태 이력을 저장합니다.

- `id UUID PRIMARY KEY`
- `incident_id UUID NOT NULL REFERENCES incidents(id)`
- `participant_id UUID NOT NULL REFERENCES participants(id)`
- `encrypted_data BYTEA NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`

건강정보 원문을 일반 JSONB나 로그에 저장하지 않습니다. 필드 암호화 방식과 키 관리는 `docs/SECURITY.md`를 따릅니다.

### messages

사고방 문자 채팅 기록을 저장합니다.

- `id UUID PRIMARY KEY`
- `incident_id UUID NOT NULL REFERENCES incidents(id)`
- `sender_participant_id UUID NOT NULL REFERENCES participants(id)`
- `message_type TEXT NOT NULL`
- `content TEXT NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`

권장 인덱스:

```sql
CREATE INDEX messages_incident_created_idx
ON messages (incident_id, created_at);
```

### risk_assessments

규칙 기반 위험도 평가 이력을 저장합니다.

- `id UUID PRIMARY KEY`
- `incident_id UUID NOT NULL REFERENCES incidents(id)`
- `score NUMERIC(5,2) NOT NULL`
- `level TEXT NOT NULL`
- `factors JSONB NOT NULL`
- `input_version TEXT NOT NULL`
- `assessed_at TIMESTAMPTZ NOT NULL`

### equipment_recommendations

구조장비 추천과 근거를 저장합니다.

- `id UUID PRIMARY KEY`
- `incident_id UUID NOT NULL REFERENCES incidents(id)`
- `equipment_code TEXT NOT NULL`
- `equipment_name TEXT NOT NULL`
- `priority SMALLINT NOT NULL`
- `reason TEXT NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`

### mountains

산 기본정보를 저장합니다.

- `id UUID PRIMARY KEY`
- `name TEXT NOT NULL`
- `peak_location GEOMETRY(POINT, 4326) NULL`
- `boundary GEOMETRY(MULTIPOLYGON, 4326) NULL`
- `metadata JSONB NOT NULL`
- `source TEXT NOT NULL`
- `source_updated_at TIMESTAMPTZ NULL`

### trails

등산로 선형 데이터를 저장합니다.

- `id UUID PRIMARY KEY`
- `mountain_id UUID NULL REFERENCES mountains(id)`
- `name TEXT NULL`
- `geom GEOMETRY(LINESTRING, 4326) NOT NULL`
- `metadata JSONB NOT NULL`
- `source TEXT NOT NULL`

### facilities

산악위치표지판, 간이구조구급함 등 시설을 저장합니다.

- `id UUID PRIMARY KEY`
- `facility_type TEXT NOT NULL`
- `name TEXT NULL`
- `location GEOMETRY(POINT, 4326) NOT NULL`
- `public_metadata JSONB NOT NULL`
- `encrypted_restricted_data BYTEA NULL`
- `source TEXT NOT NULL`

간이구조구급함 비밀번호 같은 제한정보는 `encrypted_restricted_data`에 암호화해 저장하고 권한 있는 사용자에게만 제공합니다.

### weather_cache

외부 기상 API 응답을 캐시합니다.

- `id UUID PRIMARY KEY`
- `grid_key TEXT NOT NULL`
- `location GEOMETRY(POINT, 4326) NOT NULL`
- `data JSONB NOT NULL`
- `source TEXT NOT NULL`
- `observed_at TIMESTAMPTZ NOT NULL`
- `fetched_at TIMESTAMPTZ NOT NULL`
- `expires_at TIMESTAMPTZ NOT NULL`

### audit_logs

민감정보 접근과 중요 동작을 기록합니다.

- `id UUID PRIMARY KEY`
- `incident_id UUID NULL REFERENCES incidents(id)`
- `actor_role TEXT NOT NULL`
- `actor_id UUID NULL`
- `action TEXT NOT NULL`
- `target_type TEXT NOT NULL`
- `target_id UUID NULL`
- `request_id UUID NOT NULL`
- `created_at TIMESTAMPTZ NOT NULL`

감사로그에는 민감정보 원문과 전체 좌표를 저장하지 않습니다.

## 토큰 저장

- 접속 토큰은 충분한 난수로 생성합니다.
- 원문은 발급 응답에서 한 번만 표시합니다.
- 데이터베이스에는 서버 비밀키를 사용하는 HMAC-SHA-256 결과만 저장합니다.
- 토큰 검증은 수신 토큰의 HMAC과 저장값을 상수 시간 방식으로 비교합니다.
- WebSocket 단기 토큰은 일회용으로 처리하고 사용 여부와 만료 시각을 관리합니다.

## 마이그레이션

모든 스키마 변경은 Alembic으로 관리합니다.

```text
alembic revision --autogenerate -m "describe change"
alembic upgrade head
```

운영 데이터베이스를 직접 수동 변경하지 않습니다.

## 시연 데이터

- 파일 형식: GeoJSON 또는 CSV
- 모든 시연 데이터에 `source=demo`를 표시합니다.
- `DEMO_DATA_MODE=true`에서만 시연 데이터 적재를 허용합니다.
- 실제 이름, 전화번호, 건강정보와 실제 사고 위치를 포함하지 않습니다.

## 보존과 백업

- 사고기록은 사고 종료 후에도 보존합니다.
- 보존기간은 관련 법령과 소방기관 기록관리 방침에 따라 확정합니다.
- 위치 이동 이력과 건강정보의 장기 보존 필요성을 별도로 검토합니다.
- 백업은 암호화하고 접근권한을 제한합니다.
- AI 학습에는 익명화 또는 가명처리된 데이터만 사용합니다.
