# 운영 · 트러블슈팅

게이트웨이는 모든 사이트의 입구라서, **정문이 멈추면 모든 사이트가 멈춥니다.** 이 페이지는 평소 작업과 증상별 점검 순서를 정리합니다.

## 평소 작업

| 작업 | 방법 |
|---|---|
| 설정 배포 | 개발 PC 에서 `./deploy.sh` |
| 인증서 발급·이름 추가 | `./cert.sh`, `DOMAINS="…" ./cert.sh` |
| 로그 | VM 에서 `docker logs --tail 200 -f sams-gateway` |
| 설정만 다시 읽기 | VM 에서 `docker exec sams-gateway nginx -t && docker exec sams-gateway nginx -s reload` |
| 상태 | VM 에서 `docker ps --filter name=sams-gateway` |
| 현재 설정 보기 | VM 에서 `docker exec sams-gateway nginx -T` |

## 증상별 점검

| 증상 | 흔한 원인 | 확인 · 조치 |
|---|---|---|
| 모든 사이트 접속 불가 | 게이트웨이 컨테이너 정지, 잘못된 설정으로 재시작 실패 | `docker ps -a`, `docker logs sams-gateway`. 설정 오류면 직전 커밋으로 되돌려 `deploy.sh` |
| 특정 경로만 404 | location 미등록, 끝 `/` 차이 | `nginx -T` 로 location 확인. `/hr` 과 `/hr/` 은 다른 규칙 |
| 화면은 뜨는데 그림·스크립트 404 | 서비스 화면이 절대경로 사용 | 브라우저 네트워크 탭에서 `/hr` 없이 나간 요청 확인 → 서비스를 상대경로로 ([새 사이트 추가](add-site.md)) |
| 로그인 후 엉뚱한 주소로 이동 | 서비스가 준 `Location` 에 접두사 누락 | `curl -sI` 로 `Location` 확인, `proxy_redirect` 규칙 점검 |
| 502 Bad Gateway | 서비스 정지, 도커 네트워크 끊김, 호스트 포트 미개방 | 아래 "서비스 쪽 compose down 이후" 참고. 호스트 포트 방식이면 VM 에서 `curl 127.0.0.1:<포트>` |
| 504 Gateway Timeout | 서비스 처리 시간이 `proxy_read_timeout` 초과 | 해당 location 의 timeout 조정 |
| 413 Request Entity Too Large | `client_max_body_size` 초과 | 해당 location 에서 상향 |
| 인증서 만료 경고 | 자동 갱신 실패 (80 차단, 검증 location 삭제, DNS 변경) | [인증서](certificates.md) 의 dry-run 으로 원인 확인 |
| 브라우저가 http 접속을 거부 | 대표 도메인 HSTS (1년) | 정상 동작. 대표 도메인으로는 평문 서비스를 두지 않음 |

## 서비스 쪽 `docker compose down` 이후

도커 네트워크 방식으로 붙은 서비스(인증서 관리 서비스)는 `docker compose down` 을 하면 자기 네트워크를 지웁니다. 서비스를 다시 올리면 같은 이름의 **새 네트워크**가 생기지만, 게이트웨이는 지워진 옛 네트워크에 붙어 있던 상태라 그 서비스로 가지 못합니다(502).

```bash
# 서비스를 다시 올린 뒤, VM 에서
cd ~/project/SamsGateway && docker compose up -d
```

평소 배포(`docker compose up -d`, 롤링 배포)는 네트워크를 지우지 않으므로 해당하지 않습니다. 호스트 포트 방식(SamsApi)은 이 문제가 없습니다.

!!! warning "외부 네트워크가 없으면 게이트웨이가 시작하지 못합니다"
    게이트웨이 compose 는 `<서비스 네트워크>` 를 external 로 참조합니다. 이 네트워크가 없는 상태에서 게이트웨이를 재생성하면 시작에 실패해 **모든 사이트가 멈춥니다.** 서비스를 영구히 내릴 때는 먼저 게이트웨이 설정에서 그 서비스와 네트워크를 빼고 배포하세요.

## VM 재부팅

게이트웨이와 서비스 모두 `restart: unless-stopped` 라 자동으로 올라옵니다. 도커가 네트워크를 함께 복원하므로 별도 조치가 필요 없습니다. 재부팅 후 다음을 확인합니다.

```bash
docker ps --format '{{.Names}}\t{{.Status}}'
curl -sI https://hub.example.com/svc/app | head -1
```

## 장애 영향 범위

| 멈춘 것 | 영향 |
|---|---|
| 게이트웨이 | 게이트웨이를 거치는 모든 사이트 (SamsApi 의 직접 포트 접속은 별개) |
| 서비스 하나 | 그 사이트만 502. 다른 사이트는 정상 |
| 서비스 재배포 | 게이트웨이 영향 없음. 실측: SamsApi 배포로 8400 이 1초 끊길 때 게이트웨이 경유 요청 실패 0 |
