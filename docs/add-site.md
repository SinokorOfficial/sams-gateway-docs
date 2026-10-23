# 새 사이트 추가

대표 도메인(`hub.example.com`)에 새 사이트를 경로로 붙이는 절차입니다. 예시로 HR 사이트를 `/hr/` 에 붙입니다.

## 1. 서비스 준비

서비스는 평소처럼 자기 compose 프로젝트로 배포합니다. 게이트웨이를 위해 서비스가 지켜야 할 조건은 다음과 같습니다.

- **외부 포트를 열지 않는다.** 호스트 포트를 쓰더라도 VM 방화벽(NSG)에서는 닫아 둡니다.
- **하위 경로에서 동작해야 한다.** 게이트웨이는 `/hr` 을 떼고 넘기므로 서비스는 `/` 에 있다고 생각하지만, 브라우저 주소는 `/hr/...` 입니다.
    - 화면의 정적 파일·API 호출은 상대경로로 씁니다. `fetch("api/v1/...")`, `<script src="assets/app.js">`.
    - 프레임워크 기본 경로 설정을 쓰면 안전합니다. Vite `base: "./"` 또는 `"/hr/"`, FastAPI `root_path="/hr"`(문서·리다이렉트 주소용).
    - 쿠키를 쓴다면 `Path` 를 `/hr` 로 제한합니다.
- **브라우저 저장소 키를 고유하게 한다.** 같은 도메인의 사이트끼리 localStorage 를 공유합니다. 예: `token` 대신 `hrToken`.

!!! tip "접두사를 떼지 않는 방식"
    서비스가 이미 `/hr` 아래에서 동작하도록 만들어져 있으면(Spring `server.servlet.context-path=/hr` 등), 게이트웨이에서 접두사를 떼지 않습니다. `proxy_pass http://hr;` 처럼 **끝에 `/` 없이** 쓰고 `proxy_redirect` 는 생략합니다.

## 2. 게이트웨이에서 서비스로 가는 길

| 서비스 상태 | 방법 |
|---|---|
| 호스트 포트로 열려 있음 (권장) | `server host.docker.internal:<포트>;` — 게이트웨이 compose 수정 불필요 |
| 도커 네트워크로만 접근 가능 | 게이트웨이 `docker-compose.yml` 의 `networks` 에 그 서비스 네트워크를 `external` 로 추가하고 `server <서비스명>:<포트> resolve;` |

호스트 포트 방식은 서비스와 게이트웨이가 서로 독립적이라 권장합니다. 도커 네트워크 방식은 서비스 쪽 `docker compose down` 이 게이트웨이 연결을 끊는다는 점을 기억하세요 ([운영](operations.md)).

## 3. upstream 과 location 추가

`conf.d/<대표 도메인>.conf` 의 대표 도메인 블록에 `/svc/` 를 본떠 추가합니다.

```nginx
# 호스트 포트 방식
upstream hr {
    server host.docker.internal:8600;
    keepalive 8;
}

# (도커 네트워크 방식이면)
# upstream hr { zone hr 64k; resolver 127.0.0.11 valid=5s ipv6=off; server web:8000 resolve; }

location = /hr { return 301 /hr/; }
location ^~ /hr/ {
    proxy_pass http://hr/;                                  # "/hr" 를 떼고 넘김
    proxy_redirect ~^(?:https?://[^/]+)?/(.*)$ /hr/$1;      # 이동 주소에 /hr 다시 붙이기
}
```

필요에 따라 추가합니다.

| 상황 | 설정 |
|---|---|
| 큰 업로드 | `client_max_body_size 50m;` |
| 오래 걸리는 처리 | `proxy_read_timeout 300s;` |
| WebSocket | `proxy_set_header Upgrade $http_upgrade;` `proxy_set_header Connection "upgrade";` |
| 서비스가 자체 TLS | `proxy_pass https://hr/;` |

## 4. 배포와 확인

```bash
./deploy.sh
```

검사에 실패하면 운영 설정은 바뀌지 않습니다 ([배포](deployment.md)). 반영 후 확인합니다.

```bash
curl -sI https://hub.example.com/hr/ | head -5          # 200 또는 서비스의 302
curl -sI https://hub.example.com/hr  | grep -i location  # /hr/ 로 301
curl -sI https://hub.example.com/    | head -1           # 여전히 404 여야 함
```

브라우저에서 화면을 열어 개발자 도구 네트워크 탭에 404 가 없는지 봅니다. 절대경로를 쓴 자원은 `/hr` 없이 요청되어 404 가 납니다.

## 5. 기록

`README.md` 의 사이트 목록 표에 한 줄 추가합니다(경로, 서비스, 넘기는 곳, 저장소, 비고).

## 체크리스트

- [ ] 서비스 외부 포트 닫힘
- [ ] 화면이 상대경로 사용, 하위 경로에서 새로고침해도 동작
- [ ] 저장소 키·쿠키 Path 고유
- [ ] upstream·location 추가, 필요한 크기·시간 제한 설정
- [ ] `./deploy.sh` 통과
- [ ] 대표 도메인 루트가 여전히 404
- [ ] README 사이트 목록 갱신
