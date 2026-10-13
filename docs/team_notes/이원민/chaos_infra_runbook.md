# 인프라 장애 실행·복원 절차

3단계 구현 기준: 2026-10-08. 대상 스크립트는 [chaos_infra.sh](../../../scripts/chaos_infra.sh), 회귀 테스트는 [test_chaos_infra.sh](../../../scripts/tests/test_chaos_infra.sh)다.

## 환경 준비

로컬은 `.env.example`을 기준으로 `.env`를 준비하고 `COMPOSE_PROFILES=local`을 사용한다. 기존 `.env`를 덮어쓰지 않는다. 앱 8개 + Mailpit + PostgreSQL + Redis, 총 11개다. 처음 기동은 `docker compose --profile local up -d --build`로 하고 `docker compose ps`의 health와 서비스 응답을 확인한다.

앱 서버는 `~/minwon/.env`를 보존하고 `COMPOSE_PROFILES`를 비운다. 앱 8개 + Mailpit, 총 9개이며 DATABASE_URL·REDIS_URL은 데이터 서버를 가리킨다. 계정은 기존 `deploy/data/.env.example`을 따른다. 키·비밀번호를 실행 로그나 보고서에 복사하지 않는다. Mailpit은 외부 SMTP 릴레이를 설정하지 않는다. 이번 9개 구성은 예시 주소로 config를 검사했으며 실제 EC2 기동 결과가 아니다.

기본 환경 파일은 저장소 루트 `.env`다. 별도 환경은 `CHAOS_ENV_FILE`로 지정한다. 파일 경로는 스크립트가 이동하는 저장소 루트 기준이다. 파일 전체를 source로 실행하지 않고 DATA_HOST 값만 읽는다.

```bash
# 이미 실행 중인 로컬 서비스 장애
bash scripts/chaos_infra.sh classifier 5

# 격리 QA env 사용. 명시적 빈 DATA_HOST는 파일의 원격 주소 대신 로컬 시험 선택
CHAOS_ENV_FILE=.venv/qa.env DATA_HOST= bash scripts/chaos_infra.sh postgres 5

# 원격 데이터 서버 연결 차단: Linux 앱 서버의 기존 .env에 DATA_HOST 설정 후 실행
bash scripts/chaos_infra.sh postgres 5
```

대상은 backend·classifier·locator·verifier·writer·chaos·frontend·postgres·redis·mailer·mailpit 11개, 시간은 0~60초(기본 5초)다. DATA_HOST는 선행 0이 없는 IPv4 주소만 받는다. postgres·redis 외 대상에는 DATA_HOST를 적용하지 않는다.

## 동작·종료 코드

| 조건 | 동작 |
|---|---|
| 잘못된 대상·시간·IPv4·명시 env 누락 | 종료 2, 인프라 변경 없음 |
| 대상 컨테이너 없음·여러 개·다른 체크아웃·paused 등 | 비정상 종료, 컨테이너 생성/변경 없음 |
| 대상이 원래 exited/created | 종료 0, 중지 상태 유지 |
| 대상이 running | 해당 프로젝트의 서비스만 stop → 대기 → start |
| 같은 대상의 실험 중이거나 미해결 복원 잠금 존재 | 종료 1, 두 번째 실험 거부 |
| INT / TERM | 대기를 중단하고 복원, 각각 130 / 143 |
| 중단·대기·삽입 명령 실패 | 복원 시도 후 원래 실패 코드 유지 |
| 복원 실패 | 종료 1, RESTORE FAILED 표시, 복원 파일·잠금 유지 |

로컬은 start 명령 성공 뒤 최대 30번(1초 간격) 상태를 확인한다. healthcheck가 있으면 healthy, 없으면 running을 요구한다. Compose 명령 자체에 걸리는 시간은 별도다. 컨테이너 복원과 같은 job의 자동 재개는 다르다. DB 중단 중 기록·HTTP 오류·작업 재개는 backend 담당자의 계약을 따른다.

잠금은 `${TMPDIR:-/tmp}/minwon-chaos-<대상>.lock`이며 mkdir로 원자적으로 획득한다. 동일 호스트의 같은 대상 실험만 직렬화한다. 다른 TMPDIR를 사용하면 잠금이 공유되지 않으므로 팀의 실험은 같은 TMPDIR·운영 계정에서 실행한다. 배포·수동 stop·다른 장애 도구와 동시 조작하지 않는다.

## 원격 방화벽 전제

이 경로는 Linux 앱 서버의 Docker bridge 트래픽과 iptables `DOCKER-USER` 체인을 전제로 한다. 해당 체인은 Docker 규칙보다 먼저 처리된다. nftables backend는 새 DOCKER-USER 체인을 제공하지 않으므로 이 스크립트의 지원 대상이 아니다. [Docker iptables 문서](https://docs.docker.com/engine/network/firewall-iptables/), [nftables 문서](https://docs.docker.com/engine/network/firewall-nftables/).

`sudo -n iptables` 권한과 DOCKER-USER 체인을 먼저 검사한다. 자동으로 권한·방화벽 정책·daemon backend를 변경하지 않는다. 체인이 남아 있어도 실제 FORWARD 경로에 연결되지 않은 환경은 지원하지 않는다. host network나 호스트에서 직접 실행한 앱은 FORWARD 규칙의 차단 범위가 아니므로 실제 앱 요청으로 단절을 확인해야 한다.

postgres는 목적지 5432, redis는 6379를 차단한다. 사용자 정의 데이터 서버 포트는 현재 지원하지 않는다. 추가한 규칙은 `minwon-chaos:<대상>:<PID>:<난수>` comment를 갖고, 복원 시 같은 규칙만 확인·삭제한다. 기존 사용자 규칙·데이터 서버 컨테이너·볼륨을 지우지 않는다.

## 수동 복원

복원 실패·SIGKILL·호스트 종료처럼 trap이 완료되지 않는 상황은 자동 복원을 보장하지 못한다. 잠금 디렉터리는 복원이 필요한 상태를 표시하므로 곧바로 지우지 않는다.

1. 대상·프로젝트와 실행 중인 실험/배포가 없는지 확인한다. 저장소 루트에서 잠금의 `recovery.txt`를 읽는다.
2. 내용을 확인한 후 거기에 기록된 정확한 명령을 직접 실행한다. 로컬은 기존 환경 파일로 해당 서비스 start, 원격은 이번 실험 comment가 붙은 규칙 한 개만 삭제한다. 파일을 무조건 source로 실행하지 않는다.
3. 로컬은 `docker compose ... ps`와 health·실제 요청을 확인한다. 원격은 연결·실제 DB 요청 복구와 해당 comment 규칙 제거를 확인한다.
4. 성공을 확인한 뒤 해당 잠금의 recovery.txt만 제거하고 빈 디렉터리를 rmdir한다. 전체 /tmp·전체 DOCKER-USER를 삭제하지 않는다.
5. recovery.txt 없이 잠금만 남았다면 실제 상태를 먼저 확인하고, 실행 중인 소유자가 없는 경우에만 빈 잠금을 제거한다.

서버가 종료되는 상황에 외부 감시자·자동 복구 타이머는 이번 단계에서 도입하지 않았다. [3단계 검증 결과](3단계_검증결과_2026-10-08.md)와 [관찰 데이터](3단계_검증관찰_2026-10-08.json)를 참조한다.
