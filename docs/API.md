# API 명세

## 문서 상태

이 문서는 MVP 1차 구현을 위한 설계 명세입니다. 아직 구현되지 않은 API를 포함하며, 실제 구현 시 Pydantic 스키마와 함께 갱신합니다.

## 공통 규칙

- REST 기본 경로: `/api/v1`
- WebSocket 기본 경로: `/ws`
- 요청·응답 본문: JSON
- 저장 시각: UTC, ISO 8601
- REST 인증: `Authorization: Bearer <token>`
- 오류 응답은 `error.code`, `error.message`, `request_id`를 포함합니다.
- API 키와 비밀정보는 서버 환경변수로만 관리합니다.

## 역할과 토큰

사고 생성 시 다음 접속 토큰을 한 번만 발급합니다.

- `reporter_token`: 신고자
- `responder_token`: 소방대원
- `commander_token`: 지휘관

관리자는 별도 관리자 인증을 사용합니다. 장기 접속 토큰은 URL이나 요청 본문에 넣지 않습니다.

## 공통 오류 형식

```json
{
  "error": {
    "code": "INVALID_LOCATION",
    "message": "위도 또는 경도 값이 올바르지 않습니다."
  },
  "request_id": "9a3a05c5-8c2a-45c4-8b29-752cf34a785a"
}
```

주요 상태 코드는 다음과 같습니다.

- `400`: 입력값 오류
- `401`: 토큰 없음 또는 유효하지 않음
- `403`: 역할 권한 부족
- `404`: 사고 또는 데이터 없음
- `409`: 종료된 사고에 대한 변경 요청
- `429`: 요청 한도 초과
- `502`: 외부 API 장애

## REST 엔드포인트

### 1. 사고 생성

`POST /api/v1/incidents`

새 사고를 만들고 역할별 접속 토큰을 발급합니다. 이 API는 승인된 관리자 또는 상황실 사용자만 호출할 수 있도록 설계합니다.

요청 예시:

```json
{
  "title": "산악 사고",
  "reported_at": "2030-01-01T06:04:05Z"
}
```

응답 예시 — `201 Created`:

```json
{
  "incident_id": "fb129c14-1bea-4da8-9fb3-aef82ea7f450",
  "reporter_token": "one_time_reporter_token",
  "responder_token": "one_time_responder_token",
  "commander_token": "one_time_commander_token",
  "created_at": "2030-01-01T06:04:05Z"
}
```

토큰 원문은 이 응답에서 한 번만 표시합니다.

### 2. 사고 상세 조회

`GET /api/v1/incidents/{incident_id}`

역할에 맞게 필드를 제한하여 사고의 현재 상태를 반환합니다.

- 신고자: 자신의 정보와 공개 가능한 구조 진행상황만 조회
- 소방대원·지휘관: 참여자 위치, 위험도, 장비 추천, 기상정보 조회
- 관리자: 업무상 필요한 범위에서 조회

### 3. 신고자 위치 전송

`POST /api/v1/incidents/{incident_id}/reporter/location`

인증: `reporter_token`

```json
{
  "latitude": 37.12345,
  "longitude": 127.12345,
  "accuracy_m": 15.0,
  "battery_percent": 42,
  "captured_at": "2030-01-01T06:04:05Z"
}
```

검증 규칙:

- 위도: -90 이상 90 이하
- 경도: -180 이상 180 이하
- 정확도: 0 이상
- 배터리: 0 이상 100 이하
- 사고 상태가 `open`일 때만 허용

### 4. 신고자 상태 전송

`POST /api/v1/incidents/{incident_id}/reporter/status`

인증: `reporter_token`

```json
{
  "consciousness": "alert",
  "breathing": "normal",
  "bleeding": "severe",
  "mobility": "unable",
  "injury_areas": ["left_leg"],
  "pain_level": 8,
  "additional_note": "통증이 심합니다.",
  "captured_at": "2030-01-01T06:04:05Z"
}
```

건강정보는 민감정보로 처리하며 로그에 원문을 남기지 않습니다.

### 5. 기상정보 조회

`GET /api/v1/incidents/{incident_id}/weather`

현재 기상, 단기예보, 특보, 일출·일몰을 반환합니다. 응답에는 데이터 출처와 기준 시각을 포함합니다.

### 6. 산 정보 조회

`GET /api/v1/incidents/{incident_id}/mountain`

사고 위치 주변 산의 명칭, 고도, 등산로 요약과 데이터 출처를 반환합니다.

### 7. 산악안전시설 조회

`GET /api/v1/incidents/{incident_id}/facilities`

산악위치표지판, 간이구조구급함 등 인근 시설을 반환합니다. 구급함 비밀번호 같은 제한정보는 권한 있는 사용자에게만 반환합니다.

### 8. 위험도 평가 실행

`POST /api/v1/incidents/{incident_id}/risk-assessments`

규칙 기반 위험도 계산을 실행하고 결과를 저장합니다. 일반적으로 상태·위치·기상 변경 시 서버가 자동 실행하며, 수동 호출은 소방대원과 지휘관에게만 허용합니다.

응답 예시:

```json
{
  "score": 72,
  "level": "high",
  "factors": ["severe_bleeding", "sunset_soon", "strong_wind"],
  "assessed_at": "2030-01-01T06:04:05Z",
  "disclaimer": "AI 분석 결과는 참고자료이며, 최종 판단은 현장지휘관과 구조대원이 합니다."
}
```

### 9. 장비 추천 실행

`POST /api/v1/incidents/{incident_id}/equipment-recommendations`

위험도, 기상, 지형과 신고자 상태를 바탕으로 장비 추천을 생성하고 저장합니다. 수동 호출은 소방대원과 지휘관에게만 허용합니다.

### 10. 사고 기록 조회

`GET /api/v1/incidents/{incident_id}/records`

종료 여부와 관계없이 권한 있는 소방대원, 지휘관과 관리자에게 사고기록을 반환합니다. 신고자에게는 다른 참여자의 위치·건강정보를 반환하지 않습니다.

### 11. WebSocket 단기 토큰 발급

`POST /api/v1/incidents/{incident_id}/ws-token`

장기 역할 토큰을 Bearer 헤더로 검증한 뒤 짧은 유효기간의 일회용 WebSocket 토큰을 발급합니다.

```json
{
  "ws_token": "short_lived_one_time_token",
  "expires_in_seconds": 300
}
```

### 12. 사고 종료

`POST /api/v1/incidents/{incident_id}/close`

인증: `commander_token`

```json
{
  "result": "rescued",
  "closed_at": "2030-01-01T08:30:00Z"
}
```

종료 후 새 위치·상태·채팅 입력은 `409 Conflict`로 거부하고 기존 기록은 보존합니다.

## WebSocket

연결 주소:

`wss://HOST/ws/incidents/{incident_id}`

브라우저 클라이언트는 REST에서 발급받은 일회용 `ws_token`을 `Sec-WebSocket-Protocol`로 전달합니다. 장기 토큰은 쿼리 문자열에 넣지 않습니다.

모든 메시지는 다음 공통 필드를 포함합니다.

- `type`
- `message_id`
- `version`
- `timestamp`

### 채팅 메시지

```json
{
  "type": "chat",
  "message_id": "8eebba4d-9200-4d0d-a36c-6ed2636c7426",
  "version": "1.0",
  "timestamp": "2030-01-01T06:04:05Z",
  "text": "현재 위치를 확인했습니다."
}
```

서버는 발급된 토큰에서 발신자 역할을 결정합니다. 클라이언트가 보낸 `sender` 값을 신뢰하지 않습니다.

### 위치 메시지

```json
{
  "type": "location",
  "message_id": "d6b7e136-e9b3-4a77-bf76-fba72657c5ca",
  "version": "1.0",
  "timestamp": "2030-01-01T06:04:05Z",
  "latitude": 37.1,
  "longitude": 127.1,
  "accuracy_m": 12.0
}
```

### 위험도 갱신 메시지

```json
{
  "type": "risk_update",
  "message_id": "e48859d4-a607-4fe9-8943-aef41e4efae2",
  "version": "1.0",
  "timestamp": "2030-01-01T06:04:05Z",
  "level": "high",
  "score": 72,
  "factors": ["sunset_soon", "strong_wind"]
}
```

서버는 `message_id`로 중복 메시지를 식별합니다. 종료된 사고에서는 WebSocket 수신 연결을 읽기 전용으로 유지할 수 있지만 새 채팅과 위치 전송은 거부합니다.

## MVP 2차 API

다음 기능의 세부 API는 2차 구현 전에 이 문서에 추가합니다.

- 실제 등산로 기반 최적 구조 경로
- 신고자용 응급처치 챗봇
- 지휘관용 SOP 챗봇
