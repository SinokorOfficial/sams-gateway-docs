# 라우팅 규칙

게이트웨이가 요청을 어디로 보낼지는 `conf.d/` 의 서버 블록과 location 이 정합니다. 이 페이지는 현재 규칙 전체와 각 규칙이 왜 그렇게 생겼는지 설명합니다.

!!! note "사이트 경로 표기"
    실제 사이트 경로는 공개하지 않습니다([ADR-003](decisions.md)). 이 문서에서는 경로로 붙은 사이트를 `/svc/` 로, 그 첫 화면을 `/svc/app` 으로 표기합니다.

## 서버 블록

| 포트 | server_name | 파일 | 역할 |
|---|---|---|---|
| 80 | `_` (`default_server`) | `<대표 도메인>.conf` | 인증서 검증 경로 응답, 나머지는 https 로 301 |
| 443 | `hub.example.com` (`default_server`) | `<대표 도메인>.conf` | 사이트 모음 (경로 구분) |
| 443 | `samsapi.example.com` | `samsapi.conf` | SamsApi 전용 |

443 의 기본 서버가 대표 도메인이므로, 등록되지 않은 이름으로 들어온 요청은 대표 도메인 인증서로 응답하고 404 를 받습니다.

## 80 포트

```nginx
location ^~ /.well-known/acme-challenge/ {
    root /var/www/certbot;          # certbot 이 검증 파일을 쓰는 곳
}
location / {
    return 301 https://$host$request_uri;
}
```

!!! warning
    `/.well-known/acme-challenge/` location 을 지우거나 80 포트를 닫으면 인증서 자동 갱신이 **조용히 실패**합니다. 만료 직전까지 아무 증상이 없습니다.

## 대표 도메인 (`hub.example.com`)

| location | 동작 |
|---|---|
| `= /robots.txt` | `Disallow: /` 로 전체 색인 차단 |
| `= /svc`, `= /svc/` | `301 /svc/app` (첫 화면으로) |
| `^~ /svc/` | 예시 서비스로 전달 (접두사 제거) |
| `/` | `404` |

### 접두사를 떼고 넘기기

```nginx
location ^~ /svc/ {
    proxy_pass https://svc/;                              # 끝의 "/" 가 "/svc/" 를 "/" 로 바꾼다
    proxy_redirect ~^(?:https?://[^/]+)?/(.*)$ /svc/$1;   # 서비스가 준 이동 주소에 /svc 를 다시 붙인다
    client_max_body_size 10m;                              # 인증서 파일 업로드
    proxy_read_timeout 120s;
}
```

| 사용자 요청 | 서비스가 받는 경로 |
|---|---|
| `/svc/app` | `/app` |
| `/svc/api/v1/items` | `/api/v1/items` |

| 서비스가 준 `Location` | 사용자에게 가는 `Location` |
|---|---|
| `/app/` | `/svc/app/` |
| `https://hub.example.com/app` | `/svc/app` |
| `http://<서비스>:<포트>/login` | `/svc/login` |

서비스는 자기가 하위 경로 아래에 있다는 사실을 몰라도 됩니다. 대신 **화면이 절대경로(`/static/...`, `fetch("/api/...")`)를 쓰면 깨집니다.** 화면은 상대경로를 써야 합니다 ([새 사이트 추가](add-site.md)).

### 서비스 재생성 따라가기 (upstream resolve)

```nginx
upstream svc {
    zone svc 64k;
    resolver 127.0.0.11 valid=5s ipv6=off;   # 도커 내장 DNS
    server <서비스>:<포트> resolve;
    keepalive 8;
}
```

nginx 는 보통 시작할 때 upstream 이름을 한 번만 해석합니다. 서비스가 재배포되어 컨테이너 IP 가 바뀌면 옛 IP 로 계속 보내 502 가 납니다. `resolve` 를 쓰면 5초마다 다시 해석하므로 게이트웨이를 건드리지 않아도 됩니다. 이 기능은 nginx 1.27.3 이상의 오픈소스 버전에 있습니다.

## SamsApi 도메인 (`samsapi.example.com`)

```nginx
upstream samsapi {
    server host.docker.internal:8400;
    keepalive 16;
}
```

| 설정 | 값 | 이유 |
|---|---|---|
| 전달 대상 | `host.docker.internal:8400` | SamsApi 의 도커 네트워크에 묶이지 않기 위해 호스트 포트 사용 |
| `client_max_body_size` | `60m` | SamsApi 업로드 상한(50MB)과 앱 판정(55MB)보다 크게 — 413 판단은 SamsApi 가 한다 |
| `proxy_read_timeout` 등 | `300s` | 느린 프로시저 대응, SamsApi nginx 와 같은 값 |
| `proxy_redirect` | `http://…:8400/x` → `https://$host/x` | SamsApi 가 만든 평문 주소를 https 로 |
| HSTS | 없음 | 같은 이름으로 `http://…:8400` 을 쓰는 사용자가 남아 있는 동안 브라우저가 https 로 강제해 깨지는 것을 막음 |
| 인증서 | 수동 사본 → 정식 갱신 대상으로 이전 예정 | [인증서](certificates.md) 참고 |

SamsApi 는 접두사 없이 도메인 전체를 쓰므로 `proxy_redirect` 외의 경로 변환은 없습니다.

## 공통 설정

대표 도메인 서버 블록은 다음을 모든 사이트에 적용합니다.

```nginx
ssl_protocols       TLSv1.2 TLSv1.3;
ssl_session_cache   shared:GW_SSL:10m;
add_header Strict-Transport-Security "max-age=31536000" always;
add_header X-Robots-Tag "noindex, nofollow" always;
proxy_http_version 1.1;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto https;
proxy_set_header Connection "";
```

!!! tip "location 안에서 add_header 를 쓰면"
    nginx 는 location 에 `add_header` 가 하나라도 있으면 상위 블록의 `add_header` 를 상속하지 않습니다. 사이트별로 헤더를 추가해야 하면 HSTS·X-Robots-Tag 도 그 location 에 다시 적어야 합니다.
