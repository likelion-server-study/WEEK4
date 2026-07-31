# 서버 스터디 4주차 — Nginx로 요청의 입구를 만들고 관찰하기

1주차에는 PetClinic이 실행 중인지와 로그를 확인했고, 2주차에는 Docker 컨테이너로 같은 앱을 실행했습니다. 3주차에는 PostgreSQL을 별도 컨테이너로 연결해 데이터를 보관했습니다.

4주차에는 그 앞에 Nginx를 둡니다. 사용자는 Nginx만 접속하고, Nginx가 내부의 PetClinic으로 요청을 전달합니다. 이 구조에서 정적 파일 캐시, 요청 제한, 로그 확인, 앱 장애를 직접 관찰합니다.

```text
브라우저
  ↓ http://localhost:80
Nginx
  ↓ http://app:8080
PetClinic
  ↓ postgres:5432
PostgreSQL
```

## 이번 주 목표

- Nginx만 호스트 포트를 공개하고 PetClinic·PostgreSQL은 Compose 내부에서만 통신하게 만든다.
- 정적 파일의 첫 요청(`MISS`)과 재요청(`HIT`)을 비교해, Nginx 캐시가 앱 요청을 줄이는 것을 확인한다.
- Nginx·PetClinic·PostgreSQL 로그를 분리해서 보고 `429`, `502`, 캐시 상태를 찾는다.
- 과도한 요청·큰 요청·불필요한 HTTP 메서드를 Nginx 앞단에서 어떻게 처리하는지 확인한다.
- Nginx 설정을 안전하게 검사하고, PetClinic을 멈췄을 때 발생하는 `502 Bad Gateway`와 복구 과정을 확인한다.

## 진행 순서

| 순서 | 자료 | 언제 |
| --- | --- | --- |
| 1 | [docs/01-환경세팅.md](docs/01-환경세팅.md) | 모임 전 준비 |
| 2 | [docs/02-이론.md](docs/02-이론.md) | 모임 전 예습 + 앞타임 설명 |
| 3 | [docs/03-실습.md](docs/03-실습.md) | 모임 중 실습 |
| 4 | [docs/04-트러블슈팅.md](docs/04-트러블슈팅.md) | 막혔을 때 |
| 5 | [records/_템플릿.md](records/_템플릿.md) | 실습 후 기록 |

## 결과물 기록

`records/_템플릿.md`를 복사해 자신의 이름으로 저장하고, 실행 결과·상태코드·로그·캐시 비교 결과를 기록합니다. 이 작업본은 아직 GitHub에 올리지 않습니다.

## 참고 자료

- [Spring PetClinic 공식 저장소](https://github.com/spring-projects/spring-petclinic)
- [Docker Compose 공식 문서](https://docs.docker.com/compose/)
- [Nginx 리버스 프록시·캐시 문서](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx 로그 문서](https://nginx.org/en/docs/http/ngx_http_log_module.html)
- [Nginx 요청 제한 문서](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)

> 이 자료의 명령은 macOS 터미널과 WSL2 Ubuntu 기준입니다. PowerShell이 아니라 Ubuntu 터미널에서 실행합니다.
