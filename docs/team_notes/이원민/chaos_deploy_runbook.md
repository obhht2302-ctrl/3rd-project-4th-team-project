# chaos 배포 실행 절차

4단계 대상은 chaos 한 서비스다. 실행 정의는 `.github/workflows/chaos.yml`, 전송은 `scripts/deploy_chaos.sh`, 서버 반영은 `scripts/deploy_chaos_remote.sh`다. 나머지 7개 workflow 확장은 5단계다.

## 실행 조건

1. GitHub environment `chaos`에 `AWS_HOST`, `AWS_USER`, `AWS_SSH_PRIVATE_KEY`, `AWS_SSH_KNOWN_HOSTS`가 있어야 한다. 값은 출력하거나 문서에 기록하지 않는다. known_hosts는 서버 관리자가 신뢰한 호스트 키를 사용한다.
2. 기존 알림 action은 저장소 시크릿 `DISCORD_WEBHOOK_URL`을 사용한다. 새 알림 action이나 웹훅을 만들지 않는다.
3. 앱 서버의 `~/minwon/.env`가 이미 있어야 한다. Docker Compose, bash, rsync, flock, timeout이 설치되고 SSH 사용자가 Docker를 실행할 수 있어야 한다. 데이터 서버 DB·Redis 및 기존 앱 네트워크는 먼저 준비한다. `--no-deps` 배포이므로 의존 서비스를 자동 기동하지 않는다.
4. main에 관련 경로 변경을 push하면 test가 실행된다. PR·다른 브랜치는 test만 실행한다. test에는 pytest, 담당 Python 린트, workflow 조건 테스트, 셸 검사·테스트, actionlint, chaos 이미지 빌드가 포함된다.

`deploy`는 `needs: [test]`, main push 조건, `environment: chaos`를 갖는다. [GitHub needs 규칙](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)과 [environment 시크릿 범위](https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments)에 따라 테스트가 성공해야 서버 접속 단계가 실행된다.

## 서버 반영 순서

- checkout의 chaos·contracts·schema·Compose·dockerignore·원격 스크립트를 `~/minwon/.deploy/incoming/<SHA>-<run>-<attempt>/`에 전송한다. 전체 프로젝트 루트에 `rsync --delete`를 실행하지 않는다.
- `.env`, `.env.*`, 가상환경과 캐시는 전송·삭제 대상에서 제외한다. Docker context에서도 중첩 env와 `.deploy`를 제외한다.
- 서버 `.deploy/server.lock`의 flock을 최대 300초 기다린 뒤 chaos·contracts 소스와 필요한 공통 파일을 반영한다. 향후 다른 서비스 배포 스크립트도 같은 서버 잠금을 사용해야 한다.
- `docker compose -p minwon --env-file ~/minwon/.env config --quiet` 후 `up -d --build --no-deps chaos`를 최대 600초 실행한다. Docker Hub 없이 서버에서 빌드한다.
- 최대 90초 동안 컨테이너 healthy를 기다리고 컨테이너 안 `/health`의 `status=ok`, `service=chaos`를 확인한다. 성공한 경우에만 `.deploy/chaos.json`을 원자적으로 갱신한다. 기록에는 SHA·release·이미지 ID·UTC 시각만 들어간다.
- 기존 notify는 test·deploy 결과를 함께 받아 성공·실패·취소를 전송한다. skipped를 성공으로 표시하지 않는다.

이 health 검사는 프로세스 생존과 서비스 식별 검사다. 실제 Redis 연결·레드팀 LLM 호출·전체 민원 처리는 별도로 검증한다. 소스 및 공통 파일은 실패 전에 이미 반영될 수 있으며 자동 rollback은 없다. incoming 디렉터리는 증거 보존을 위해 남는다. 디스크 사용량을 확인하고 필요 없는 release만 서버 관리자가 정리한다. 동시 실행은 잠금 획득 순서로 처리하며 최신 커밋 순서를 보장하지 않으므로 겹친 실행 뒤 최종 SHA를 확인하고 필요하면 최신 main 실행을 재실행한다.

## 배포 후 인수

1. Actions의 test·deploy 성공과 environment 배포 기록을 확인한다.
2. 앱 서버에서 chaos healthy, `.deploy/chaos.json`의 SHA와 해당 Actions SHA 일치, 다른 서비스 컨테이너 ID 유지, 기존 `.env` 유지 여부를 확인한다. env 내용은 출력하지 않는다.
3. 실제 플래그 API를 실행하고 데이터 서버 연결을 확인한다. 팀 디스코드 `#배포알림`에서 해당 run의 성공 알림 수신을 확인한다.
4. 팀과 시간을 조율한 별도 main 커밋에서 의도적으로 테스트 단언을 실패시킨다. test 실패·deploy skipped·기존 chaos ID/성공 기록 유지·실패 알림을 기록하고 정상 커밋으로 복구한다. PR 실패와 로컬 조건 테스트만으로 이 인수를 완료 처리하지 않는다.

## 실패 진단

시크릿 누락은 이름만 표시하고 종료한다. SSH 호스트 키 불일치·인증 실패는 전송 전에 막힌다. 서버 `.env` 누락은 업로드 전에 막힌다. 서버 잠금 획득 실패는 소스 반영 전에 끝난다. 빌드·health 실패는 deploy 실패와 실패 알림으로 이어지고 마지막 성공 메타데이터는 유지된다. 정상 커밋을 다시 배포하기 전에 Actions 오류, 컨테이너 상태 및 로그를 확인한다. known_hosts 검증을 끄거나 `.env`를 덮어써서 우회하지 않는다.

## 로컬 재검증

```bash
# 프로젝트 루트; Python 의존성은 기존 격리 환경에서 설치
bash scripts/tests/test_deploy_chaos.sh
bash -n scripts/deploy_chaos.sh scripts/deploy_chaos_remote.sh scripts/tests/test_deploy_chaos.sh
shellcheck scripts/deploy_chaos.sh scripts/deploy_chaos_remote.sh scripts/tests/test_deploy_chaos.sh
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.7 -shellcheck= -pyflakes= .github/workflows/chaos.yml
docker compose build chaos
cd scripts/tests
npm ci
npm test
```

실제 SSH 격리 시험 결과와 원격 미검증 항목은 [4단계 검증 결과](4단계_검증결과_2026-10-08.md)를 참조한다.

5단계부터 chaos 진입점은 공통 서비스 배포 스크립트에 위임합니다. 현재 영향표·8개 서비스 운영 절차는 [서비스 배포 절차](services_deploy_runbook.md)를 참조하세요.
