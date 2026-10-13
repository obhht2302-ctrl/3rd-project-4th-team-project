# chaos API 계약 — 2단계

확정일: 2026-10-08 (KST). 사용자가 제안한 레드팀 계약에 “제안 계약 사용”으로 답변했다. 기존 공유 `contracts/`와 PL 관리 `clients/chaos.py`는 변경하지 않았다. 입력·출력 스키마는 `src/chaos/app/schemas/redteam.py`와 `/openapi.json`에서 확인한다.

## 플래그 API

- 대상: backend, classifier, locator, verifier, writer, mailer.
- `PUT /flags/{target}`: 기존 `{incident_id, kind, injected_at, params}` 저장 후 저장한 객체 반환. 시간대가 있는 시각 필수. 소비 직후에도 저장 성공 응답을 반환한다.
- `GET /flags/{target}`: 소비하지 않고 조회, 미존재는 JSON null.
- `DELETE /flags/{target}`: 멱등 해제, JSON null 반환.
- 기존 PUT의 대상·종류 enum 검증은 보존한다. 아래 조합 제한은 새 레드팀 API에 적용한다.
- 소비자는 기존 GETDEL로 한 번만 소비한다. SET은 기존 대기 플래그를 덮어쓰며 TTL은 추가하지 않는다.

## 레드팀 API

`POST /run` 요청:

```json
{"candidates":[{"target":"classifier","kind":"llm_invalid"}]}
```

후보는 1~14개다. 대상·종류 외 추가 필드는 거부한다. AI가 후보 중 정확히 하나를 선택하고 서버가 스키마·허용 조합·요청 후보 포함 여부를 모두 검증한다. 검증 통과 후 서버가 UUID incident ID와 UTC 시각을 생성해 `{incident_id,kind,injected_at,params:{}}`를 저장한다. 성공 응답은 공통 봉투 없이 다음 객체다.

```json
{"target":"classifier","kind":"llm_invalid","reason":"응답 스키마 오류 복구를 확인합니다."}
```

reason은 공백만으로 구성할 수 없으며 1~500자다. AI가 명령을 실행하거나 자유로운 장애 대상을 추가하지 않는다. 자동 재시도·대체 모델·임의 후보 선택은 없다.

| 대상 | 허용 종류 |
|---|---|
| backend | injection, delay |
| classifier | llm_invalid, delay |
| locator | delay |
| verifier | link_dead, stale_info, pii_leak, not_official, delay |
| writer | llm_invalid, delay |
| mailer | mail_down, delay |

호출자가 현재 단계에 맞는 후보를 제공한다. verifier의 link_dead·stale_info·not_official은 channel, pii_leak은 draft 실행에 맞춰 사용한다. API는 job·mode를 추론하지 않는다. 생성 params는 빈 객체이므로 delay는 기존 래퍼 기본 1초이며 타임아웃 장애 시연에는 PUT의 `params.seconds`를 사용한다. 서비스 다운·DB 단절은 별도의 인프라 스크립트다.

## 실패 정책

| 조건 | HTTP | 동작 |
|---|---|---|
| 입력 누락·잘못된 대상/조합·추가 필드 | 422 | 모델 호출·저장 없음 |
| 모델 HTTP/연결 실패·손상/거절/잘림 응답·후보 밖 선택 | 502 | 저장 없음 |
| 모델 요청 전체 10초 초과·네트워크 타임아웃 | 504 | 저장 없음 |
| OpenAI 키 미설정 | 503 | 모델 호출·저장 없음 |
| Redis 연결·명령 실패 또는 설정 미비 | 503 | 저장 성공으로 표시하지 않음 |
| GET으로 조회한 저장 값 손상 | 500 | 손상 값을 노출·자동 삭제하지 않음 |

오류는 `{"detail":"고정 오류 설명"}`이며 모델/Redis 원문·키를 응답에 넣지 않는다. 422는 FastAPI 기본 검증 형식을 유지한다. Redis SET 이후 통신이 끊기는 경우 저장 여부가 불확실할 수 있으므로 GET으로 확인하고 필요하면 DELETE한다.

## 설정과 검증 경계

`REDIS_URL`, `OPENAI_API_KEY`, `OPENAI_MODEL` 사용. 기존 env 예시의 모델 기본값을 유지한다. OpenAI Chat Completions의 [Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) 형식으로 호출하고 로컬에서도 결과를 다시 검증한다. 환경 프록시를 사용하지 않으며 요청 전체 제한은 10초다. `/health`는 기존 연결 확인 응답을 보존하고 모델 키·Redis 준비 상태를 검사하지 않는다.

실제 OpenAI 호출은 키와 모델 권한이 있어야 검증 가능하다. 이번 결과는 대역 모델 + 실제 Redis/classifier 연동이다. 현재 backend DB 작업도 backend 플래그를 먼저 소비할 수 있어 injection이 원하는 위치에 적용된다고 보장하지 않는다. 공통 래퍼·backend 담당자와 통합할 때 확인한다. EC2·CI/CD는 이후 단계다.
