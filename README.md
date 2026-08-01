# 서버 스터디 4주차 — Nginx로 요청의 입구를 만들고 관찰하기

1주차에는 PetClinic이 실행 중인지와 로그를 확인했고, 2주차에는 Docker 컨테이너로 같은 앱을 실행했습니다. 3주차에는 PostgreSQL을 별도 컨테이너로 연결해 데이터를 보관했습니다.

4주차에는 그 앞에 Nginx를 둡니다. 사용자는 Nginx만 접속하고, Nginx가 내부의 PetClinic으로 요청을 전달합니다. 이 구조에서 정적 파일 캐시, 요청 제한, 로그 확인, 앱 장애를 직접 관찰합니다.

현재 3주차 Compose에는 `mysql`과 `postgres`만 있으므로 PetClinic `app`과 `nginx` 서비스를 새로 추가합니다. MySQL 설정은 삭제하지 않고, 4주차 실행 명령에서 `postgres app nginx`만 지정해 MySQL 컨테이너는 실행하지 않습니다. PostgreSQL Named Volume은 유지합니다.

이번 자료에서는 Nginx의 호스트 입구를 `8080`으로 통일합니다. Compose의 `8080:80`은 호스트 8080으로 받은 요청을 Nginx 컨테이너 내부 80으로 전달한다는 뜻입니다.

```text
브라우저
  ↓ http://localhost:8080
Nginx ───────────── proxy-net(앞단 네트워크)
  ↓ http://app:8080
PetClinic(app) ──── proxy-net + db-net
  ↓ postgres:5432
PostgreSQL ──────── db-net(뒷단 네트워크)
```

호스트에 게시되는 입구는 Nginx 하나다. `proxy-net`은 Nginx와 app을 연결하는 앞단 네트워크이고, `db-net`은 app과 PostgreSQL을 연결하는 뒷단 네트워크다. Nginx는 `db-net`에 참여하지 않으므로 DB와 직접 통신하지 않는다.

## 이번 주 목표

- Nginx만 호스트 포트를 공개하고 PetClinic·PostgreSQL은 Compose 내부에서만 통신하게 만든다.
- 같은 Wi-Fi의 다른 장치에서도 Nginx만 접근되고 app·DB 포트는 접근되지 않는지 확인한다.
- 정적 파일의 첫 요청(`MISS`)과 재요청(`HIT`)을 비교해, Nginx 캐시가 앱 요청을 줄이는 것을 확인한다.
- 각 실습 단계에서 Nginx·PetClinic·PostgreSQL 로그를 필요한 서비스·시간·상태 코드로 좁혀 결과를 판정한다.
- 학습용 access log는 한 요청이 중복 집계되지 않도록 Nginx의 대상 `server`에 한 번만 적용한다.
- 과도한 요청(`429`)·큰 요청(`413`)·허용하지 않은 메서드(`405`)를 Nginx 앞단에서 처리하는지 확인한다.
- 경로 차단(`403`)·버전 숨김·보안 헤더까지 여러 앞단 보호 기능을 각각 시험한다.
- Nginx 설정을 반영하기 전에 `nginx -t`로 문법을 검사하고, 설정 파일 변경과 Compose 변경의 적용 방법을 구분한다.
- PetClinic을 멈췄을 때 상태 코드 `502`는 유지하면서 Nginx가 사용자 친화적인 오류 화면을 반환하고 복구되는 과정을 확인한다.

실습은 **1 Nginx 연결 → 2 뒷단 비공개 → 3 캐시 → 4 앞단 보호 → 5 앱 장애** 순서로 진행한다. 로그는 별도 단계로 떼지 않고 각 단계의 성공 여부를 확인하는 증거로 사용한다.

| 단계 | 핵심 판정 |
| --- | --- |
| 1 | 세 컨테이너 기동, Nginx 문법 성공, Nginx 경유 `200` |
| 2 | Nginx만 호스트에 공개되고 app·DB는 실행 중이지만 직접 접근 불가 |
| 3 | 실제 이미지의 `BYPASS/MISS/HIT`, 시간, app 전달 횟수 비교 |
| 4 | `200/429/413/405/403`과 보안 헤더를 응답·로그로 구분 |
| 5 | 사용자 안내 화면과 `502` 유지, access/error log, app 재시작 후 복구 |

## 진행 순서

| 순서 | 자료 | 언제 |
| --- | --- | --- |
| 1 | [docs/01-환경세팅.md](docs/01-환경세팅.md) | 모임 전 준비 |
| 2 | [docs/02-이론.md](docs/02-이론.md) | 모임 전 예습 + 실습 전 이론 설명 |
| 3 | [docs/03-실습.md](docs/03-실습.md) | 모임 중 실습 |
| 4 | [docs/04-트러블슈팅.md](docs/04-트러블슈팅.md) | 막혔을 때 |
| 5 | [records/_템플릿.md](records/_템플릿.md) | 실습 후 기록 |

## 결과물 기록

`records/_템플릿.md`를 복사해 자신의 이름으로 저장하고, 실행 결과·상태 코드·로그·캐시 비교 결과를 기록합니다.

실습이 끝난 뒤에는 `docker compose down`으로 세 컨테이너를 함께 정리합니다. 단, `-v`는 PostgreSQL 데이터 볼륨까지 지울 수 있으므로 사용하지 않습니다.

## 참고 자료

- [Spring PetClinic 공식 저장소](https://github.com/spring-projects/spring-petclinic)
- [Docker Compose 공식 문서](https://docs.docker.com/compose/)
- [Nginx 리버스 프록시·캐시 문서](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx 로그 문서](https://nginx.org/en/docs/http/ngx_http_log_module.html)
- [Nginx 요청 제한 문서](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)

> 이 자료의 명령은 macOS Terminal과 WSL2 Ubuntu의 셸을 기준으로 작성했습니다. Windows 사용자는 PowerShell이 아니라 WSL2 Ubuntu 터미널에서 실행합니다.
