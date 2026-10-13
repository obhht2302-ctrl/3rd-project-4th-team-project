# 8개 서비스 배포 실행 절차

5단계는 backend·classifier·locator·verifier·writer·mailer·chaos·frontend의 기존 workflow에 test → deploy → notify를 연결한다. 실제 EC2 운영 인수는 main 실행 후 별도로 확인한다. [4단계 chaos 절차](chaos_deploy_runbook.md)의 SSH·env 보호·서버 준비 조건은 그대로 적용한다.

## 변경 영향표

사용자 확정: **기존 `database/**` 전체 서비스 트리거 유지**.

| 변경 경로 | 검사·main 배포 대상 |
| --- | --- |
| `src/<서비스>/**`, `.github/workflows/<서비스>.yml` | 해당 서비스만 |
| `contracts/**`, `database/**` | 8개 모두 |
| `docker-compose.yml`, `.dockerignore`, `.gitattributes` | 8개 모두 |
| `scripts/deploy_*.sh`, `scripts/tests/test_deploy_*.sh`, `scripts/tests/package*.json`, `scripts/tests/test_*workflow.mjs` | 8개 모두 |
| `.github/actions/**` | 8개 모두 |
| 문서·README만 변경 | 배포하지 않음 |

PR·main 외 브랜치는 test만 실행한다. test 성공은 기존 애플리케이션 테스트, 배포 스크립트·workflow 검사, 해당 서비스 이미지 빌드의 성공을 포함한다. frontend는 기존 `npm ci`, `npm run build`, `npm test`를 유지한다. 각 deploy는 동일 이름의 GitHub environment를 사용한다. 8개 environment 각각에 접속 시크릿 이름 4개가 존재함을 읽기 전용으로 확인했으며 값은 조회하지 않았다.

## 최초 서버 준비

앱 서버의 기존 `~/minwon/.env`를 유지한다. OpenAI 키·데이터 서버 DB/Redis 주소 등은 서버 관리자만 설정하며 CI에서 업로드하지 않는다. bash·rsync·flock·timeout·Docker Compose 및 SSH 사용자의 Docker 실행 권한이 필요하다. `AWS_SSH_KNOWN_HOSTS`는 신뢰한 서버 호스트 키를 사용한다.

초기 서버는 전체 코드와 locator seed를 준비하고 앱 8개 및 Mailpit을 먼저 기동한다. 데이터 서버 DB/Redis는 별도로 준비한다. 앱 서버에서 local profile은 비워야 한다. DB 초기 스키마·seed 준비는 데이터 담당 절차에 따르며 **소스 배포가 DB 마이그레이션을 실행하지 않는다**.

```bash
# 앱 서버 ~/minwon, 운영 .env가 준비된 뒤
# 데이터 서버 연결과 디스크·메모리 여유를 먼저 확인한다.
docker compose config --quiet
docker compose up -d --build backend classifier locator verifier writer mailer chaos frontend mailpit
```

서비스별 deploy는 `--no-deps`를 사용하므로 최초 의존 서비스 준비를 대신하지 않는다. Mailpit용 workflow는 만들지 않았고 mailer 단독 배포가 Mailpit을 재시작하지 않는다. Mailpit의 외부 SMTP relay 설정을 추가하지 않는다.

## 자동 배포 경로

`bash scripts/deploy_service.sh <서비스>`는 허용된 8개 이름만 받는다. 소스와 contracts·database·Compose·dockerignore·원격 스크립트를 release별 `.deploy/incoming/<SHA>-<run>-<attempt>/`에 업로드한다. SSH 임시 키는 제한 권한으로 만들고 종료 시 제거한다.

`deploy_service_remote.sh`는 서버 공통 `.deploy/server.lock`을 최대 300초 기다린다. 잠금 안에서 해당 서비스 폴더·contracts·schema·Compose만 반영하며 locator 배포 시 `database/seeds`도 동기화한다. `rsync --delete`는 대상 서비스·contracts·locator seeds의 좁은 범위에만 적용하고 `.env`, `.env.*`, 가상환경·캐시를 보호한다. 서버 디렉터리의 실제 경로를 사용해 bind mount 경로를 일치시킨다. 공유 파일 변경에서도 각 workflow가 선택 서비스만 기동한다.

서버에서 최대 600초 `docker compose -p minwon --env-file <기존 env> up -d --build --no-deps <서비스>`를 실행한다. 최대 90초 healthy를 기다린 뒤 Python 서비스는 JSON `/health`의 status·service를, frontend는 nginx `/health`의 서비스 식별 응답을 확인한다. 성공한 경우에만 `.deploy/<서비스>.json`을 갱신한다. Docker Hub나 추가 이미지 push 시크릿은 사용하지 않는다.

기존 chaos 진입점 `deploy_chaos.sh`, `deploy_chaos_remote.sh`는 공통 스크립트로 위임한다. 기존 알림 action은 유지하고 notify가 test·deploy를 모두 기다린다. 실패·취소·예상 밖 skipped를 성공으로 처리하지 않는다.

## 운영 확인·실패 복구

- 각 Actions run의 SHA와 environment 배포 기록, 서버 `.deploy/<서비스>.json`, 컨테이너 health를 대조한다. `/health`는 생존·식별 검사이며 Redis·LLM·업무 성공을 보증하지 않는다.
- 단일 서비스 변경은 다른 10개 로컬 컨테이너/앱 서버의 나머지 서비스 ID를 유지해야 한다. 공유 변경은 영향표의 8개 run과 최종 성공 SHA를 모두 확인한다.
- frontend `/api/health`로 backend proxy를 확인한다. locator `/seeds` 파일, mailer→Mailpit 연결과 실제 플래그 API도 별도로 확인한다.
- 잠금은 실행 도착 순서로 처리하며 커밋 순서를 보장하지 않는다. 겹친 push 후 최종 SHA를 확인하고 필요하면 최신 main run을 재실행한다. 향후 별도 배포 도구도 같은 서버 잠금을 사용한다.
- 서버 소스가 반영된 뒤 실패할 수 있고 자동 rollback은 없다. Actions 오류·컨테이너 로그를 확인한 뒤 정상 커밋을 다시 배포한다. 실패 시 마지막 성공 메타데이터는 유지된다.
- incoming release는 보존된다. 디스크 사용량을 관리하고 필요 없는 시험/배포 release만 관리자가 정리한다. 운영 env·볼륨을 삭제하지 않는다.

main 성공 및 실제 `#배포알림` 수신을 확인하고, 팀과 조율한 main 테스트 실패 시연에서 deploy skipped·기존 서비스 유지·실패 알림을 기록한다. 로컬 표현식 시험은 실제 GitHub 실행이나 디스코드 수신을 대신하지 않는다.

## 개발 검증

```bash
# 프로젝트 루트
for service in backend classifier locator verifier writer mailer chaos frontend; do
  bash scripts/tests/test_deploy_chaos.sh "$service" || exit
done
bash -n scripts/deploy_*.sh scripts/tests/test_deploy_chaos.sh
shellcheck scripts/deploy_*.sh scripts/tests/test_deploy_chaos.sh
docker compose build backend classifier locator verifier writer mailer chaos frontend
# scripts/tests 폴더
npm ci
npm test
```

[5단계 검증 결과](5단계_검증결과_2026-10-08.md)에서 로컬 실배포와 미수행 원격 인수를 구분한다.
