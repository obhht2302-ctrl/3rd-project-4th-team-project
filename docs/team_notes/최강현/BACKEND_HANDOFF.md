# 백엔드·통합 민원 AI 에이전트 인계

최강현 담당 구현은 `src/locator`, `database/seeds`의 운영/참고 CSV, `scripts/check_channels.py`에 한정했습니다. backend, classifier, writer, verifier, mailer, contracts, DB 스키마, Compose는 수정하지 않았습니다.

## 전체 저장소와 연동 관계

| 위치 | 담당 역할·현재 연결 |
|---|---|
| `src/frontend` | 현재 건강 확인 화면. 민원 입력·질문·타임라인·결과 화면은 후속 구현 대상 |
| `src/backend` | Supervisor, 작업/타임라인/질문 저장, agent HTTP 호출, 가상 발송 라우트 |
| `src/classifier` | 유형·지역 분류 계약. 현재 샘플 기반 스텁 |
| `src/locator` | 이번 구현: CSV→DB 적재 및 실제 관할 창구 조회 |
| `src/verifier` | 공식 웹 링크·관할·작성 초안 검증. 전화 링크 지원은 후속 작업 |
| `src/writer` | 초안 작성 계약. 현재 샘플 기반 스텁 |
| `src/mailer` | Mailpit으로만 가상 발송. `.example` 수신자 사용 |
| `src/chaos` | Redis 장애 플래그 주입 |
| `database/schema.sql` | backend 소유 영속 테이블 계약 |
| `database/seeds` | 추천 창구, 참고자료, PL 소유 mock 수신자 |
| `contracts` | 입출력 JSON Schema·기존 샘플, 변경 없음 |
| `docker-compose.yml`, `deploy` | 실행·데이터 서버 설정. 기존 locator `/seeds:ro` 마운트 활용 |

## backend가 호출할 API

컨테이너 내부는 `http://locator:8000/run`, 기본 로컬 포트는 `http://127.0.0.1:8020/run`입니다. 신규 backend API를 추가할 필요 없이 기존 `clients/agents.py` 호출 계약을 사용합니다.

```json
{
  "task_id": "job-example",
  "from": "supervisor",
  "to": "locator",
  "objective": "공식 신고 창구 탐색",
  "inputs": {
    "category": "streetlight",
    "region": {"sido": "서울특별시", "sigungu": "동작구", "dong": "상도동", "place_text": "사용자가 제공한 장소"},
    "exclude_channel_ids": []
  },
  "constraints": ["공식 출처만 사용할 것"],
  "attempt": 1,
  "last_failure": null,
  "memory_refs": []
}
```

`category`: `streetlight`, `road_damage`, `illegal_dumping`, `out_of_scope`. CSV 내부의 `all`을 API 입력으로 보내지 않습니다. `region`의 네 키는 유지하고 모르는 값은 null로 보냅니다. 동명 하나만 보고 임의로 동작구라고 보정하지 않습니다.

성공 시 `status=success`, `failure=null`, `data={channels:[...],fallback_level:district|city|national}`입니다. channel은 기존 계약의 **channel_id, name, org, method, url, jurisdiction, verified_at, source_url** 8필드만 포함합니다. CSV의 지역·유형·active 및 참고자료 메타데이터를 응답에 추가하면 기존 계약을 위반합니다.

빈 결과는 HTTP 200 / `status=failed`, `failure=NO_CHANNEL`, `channels=[]`, `fallback_level=national`입니다. DB 장애는 HTTP 503입니다. backend의 기존 HTTP 오류 변환이 `author=backend`, `failure=SERVICE_UNAVAILABLE`로 처리해야 하며 locator가 이 실패 코드를 직접 생성하지 않습니다. 잘못된 Handoff는 HTTP 422입니다.

## 데이터 활용

- 운영 2개 CSV 14행만 `channels`에 적재합니다. 기본 추천 가능한 웹 창구는 8행, 전화 참고는 6행입니다. 시작 시 DB가 비어 있지 않으면 새 CSV는 반영되지 않으므로 수동 upsert 절차를 사용합니다.
- `database/seeds/seed_manifest.csv`가 전체 CSV 경로·컬럼·행수·해시·소비 서비스의 기준입니다. `reference`는 data_sources/complaint_routing/agent_guidance/security/operations로 분류했습니다. 출처·확인일·검증범위·용도·제약을 함께 가져가세요. 참고자료의 별도 DB/RAG/MCP 연결은 후속 구현입니다.
- 작성 가이드는 `project_recommendation`이며 기관의 공식 문구가 아닙니다. 요구사항의 `channel_scope`, `applicability`, `requirement_level`을 필터링하여 **포상금 신고 요건을 일반 쓰레기 수거 요청에 강제하지 마세요**.
- 행정동 목록은 법정동·좌표 경계 데이터가 아닙니다. 공공데이터 카탈로그는 실제 원자료나 현재 고장 증거가 아닙니다. `research_backlog`는 조사할 질문이며 검증된 답변으로 검색 결과에 제시하지 않습니다.
- `dj-*` ID와 기존 mock 수신자의 `dongjak-*` ID가 다르지만 mailer는 미등록 ID에 `{channel_id}@minwon.example`을 사용합니다. 시연용 이름/담당자 맞춤은 PL에게 요청하면 됩니다. 실제 기관 이메일로 변경하지 않습니다.

## 타 담당 후속 작업

| 담당 | 작업 | 현재 근거 |
|---|---|---|
| backend | verifier 실패 ID를 누적하여 locator의 `exclude_channel_ids`로 재호출; 단계 재시도/재계획 제한 구현 | `src/backend/app/services/supervisor.py`의 TODO, 현재 제외목록은 항상 빈 배열 |
| backend | `OUT_OF_SCOPE`일 때 전국 통합 창구 안내까지 연결 | 현재 classifier 성공 여부에서 먼저 종료되는 흐름 확인 필요 |
| backend | 장애 incident·복구 지표 기록, HTTP 제한시간 팀 합의 | agents.py는 현재 15초, 작업배분서 기준은 10초 |
| verifier/PL | 전화 형식과 `source_url` 공식성/접근성 검증 후 전화 추천 활성화 | 현재 `channel.py`는 http(s) URL만 인정, source_url 별도 검증 없음 |
| classifier/writer | 실제 분류·초안 생성 및 reference 활용 시 출처·제안 구분 | 현재 스텁 응답이므로 이번 locator 완성만으로 전체 AI 흐름이 완성되지는 않음 |

공공기관 사이트 URL은 자동 접수 API를 의미하지 않습니다. 인증·실제 제출 API는 확인되지 않았으며 이번 구현은 창구 탐색과 사용자 접수 안내까지입니다. 추가조사 우선순위는 `reference/operations/research_backlog.csv`에 기록했습니다.

정기 점검은 `scripts/check_channels.py --schedule-csv database/seeds/integration/channel_check_schedule.csv`로 실행합니다. 기본은 초회 즉시·24시간 주기·회차 실패 후 1시간 재시도이며, 운영 DB 주소는 환경변수로 제공합니다. 실행 프로세스를 OS/서버 작업에 등록하는 일은 아직 수행하지 않았습니다. 자세한 명령과 주의사항은 `database/seeds/README.md`에 있습니다.

설계서의 창구 데이터 요구사항에 맞춰 DB를 원본으로 유지합니다. 정기 점검이 커밋한 비활성 값은 다음 locator 조회에 반영되고, 모든 후보가 사라지면 구→시→전국→NO_CHANNEL 순서로 처리됩니다. 404/410 이외 일시적인 접속 실패로 창구를 삭제하지 않습니다. 링크 점검과 API 사다리를 연결한 실제 PostgreSQL 통합 테스트도 통과했습니다.
