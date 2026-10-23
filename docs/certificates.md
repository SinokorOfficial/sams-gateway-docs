# 인증서

게이트웨이의 HTTPS 인증서는 Let's Encrypt 에서 받고, VM 의 certbot 컨테이너가 자동 갱신합니다. 발급·이름 추가·갱신 등록은 모두 `./cert.sh` 하나로 하며, 다시 실행해도 안전합니다.

## 구성

| 항목 | 값 |
|---|---|
| 발급 방식 | HTTP-01 (webroot) — 게이트웨이 80 의 `/.well-known/acme-challenge/` |
| 실행 도구 | `certbot/certbot` 컨테이너 (VM 에 certbot 설치 불필요) |
| 인증서 이름 | 대표 도메인 이름의 인증서 하나(`--cert-name`)에 여러 도메인을 SAN 으로 |
| 저장 위치 | VM `~/certbot/conf/live/<인증서 이름>/` (저장소에 넣지 않음) |
| 자동 갱신 | VM crontab 매일 03:17, 만료 30일 전부터 실제 갱신 |
| 갱신 후 | `sams-gateway` 와 인증서 관리 서비스의 nginx(`proxy`)에 HUP (설정 다시 읽기) |

## cert.sh

```bash
./cert.sh                                                       # 현재 이름 확인/발급 + 갱신 cron 등록
DOMAINS="hub.example.com samsapi.example.com" ./cert.sh         # 이름 추가
```

| 단계 | 하는 일 |
|---|---|
| 1 · DNS 확인 | `DOMAINS` 의 모든 이름이 VM 공인 IP 를 가리키는지 확인. 하나라도 아니면 중단 |
| 2 · 발급 / 이름 추가 | `certonly --webroot --expand --keep-until-expiring` — 같은 이름으로 유효하면 건너뜀, 이름이 늘면 확장 발급. 끝나면 게이트웨이 reload |
| 3 · cron 등록 | `# samsapi-certbot` 표시가 붙은 줄을 지우고 다시 넣음 (중복 없음) |

**전제 조건**

- 각 이름의 DNS 가 VM 공인 IP 를 가리킨다.
- VM 방화벽에서 80·443 이 열려 있다.
- `sams-gateway` 가 떠 있고 80 의 검증 location 이 살아 있다.

## 갱신 cron

```cron
17 3 * * * docker run --rm ... certbot/certbot renew -q ... \
  && docker kill -s HUP sams-gateway $(docker ps -qf label=com.docker.compose.project=<서비스 프로젝트> -f label=com.docker.compose.service=proxy) \
  # samsapi-certbot
```

- `renew` 는 만료 30일 이내인 인증서만 실제로 갱신합니다.
- 갱신이 성공했을 때만 HUP 을 보냅니다. HUP 은 nginx 의 무중단 설정 재적재입니다.
- 인증서 관리 서비스의 nginx 는 같은 인증서로 내부 TLS 를 하므로 함께 HUP 합니다. 그 서비스가 내부 TLS 를 그만두면 이 부분을 지웁니다.

## 상태 확인

```bash
# 만료일 (외부에서)
echo | openssl s_client -connect hub.example.com:443 -servername hub.example.com 2>/dev/null | openssl x509 -noout -enddate

# 갱신 시험 (VM 에서, 실제로 바꾸지 않음)
docker run --rm -v ~/certbot/conf:/etc/letsencrypt -v ~/certbot/www:/var/www/certbot certbot/certbot renew --dry-run

# cron 등록 확인 (VM 에서)
crontab -l | grep samsapi-certbot
```

!!! warning "만료 알림 메일은 오지 않습니다"
    인증서는 이메일 등록 없이 발급되어 Let's Encrypt 의 만료 경고 메일을 받지 않습니다. 갱신이 조용히 실패해도 알 수 없으므로, 위 만료일 확인을 정기 점검 항목에 넣으세요.

## SamsApi 전용 도메인 인증서

`samsapi.example.com` 은 아직 DNS 가 사내 서버를 가리키고 있어 이 VM 에서는 발급할 수 없습니다. 그래서 사내 서버에서 쓰던 인증서를 VM 의 `~/certbot/conf/manual/samsapi.../` 에 복사해 두고 `samsapi.conf` 가 그 경로를 씁니다. 이 사본은 자동 갱신되지 않습니다.

DNS 를 VM 으로 옮긴 뒤 정식 갱신 대상으로 옮깁니다.

1. DNS 를 VM 공인 IP 로 변경하고 반영을 확인합니다.
2. `DOMAINS="hub.example.com samsapi.example.com" ./cert.sh` 로 대표 인증서에 이름을 추가합니다.
3. `samsapi.conf` 의 `ssl_certificate` 경로를 `live/<대표 인증서 이름>/` 으로 바꾸고 `./deploy.sh` 합니다.
4. 수동 사본 폴더를 정리합니다.
