# tls-handshake-lab

TLS Handshake의 실제 비용을 TLS 1.2, 1.3, 0-RTT로 비교 검증한다.

## 배경

"HTTPS는 느리지만 보안을 위해 당연히 써야지"로 넘기지만:

- TLS 1.2: 2-RTT handshake (ClientHello → ServerHello+Cert → Finished)
- TLS 1.3: 1-RTT handshake (키 교환이 한 단계 줄어듦)
- TLS 1.3 0-RTT: 이전 세션 정보로 첫 패킷에 데이터 포함 가능
- 인증서 체인이 길면 (Root → Intermediate × 2 → Leaf) 검증 비용 증가
- TLS session resumption / session ticket의 실제 효과는?

## 가설

| # | 가설 | 근거 |
|---|------|------|
| H1 | TLS 1.3이 1.2보다 첫 요청 latency가 1 RTT분 (LAN ~1ms, WAN ~50ms) 빠를 것 | handshake RTT 감소 |
| H2 | Session resumption 시 TLS 1.2도 1-RTT로 줄어들어 차이가 줄 것 | session ticket 재사용 |
| H3 | 인증서 체인 3단계가 1단계보다 검증에 1-5ms 추가할 것 | RSA/ECDSA 서명 검증 비용 |
| H4 | Keep-Alive가 켜져 있으면 TLS overhead가 첫 연결에만 발생하여 전체 영향 미미할 것 | 이후 요청은 기존 세션 사용 |
| H5 | HTTP(평문) vs HTTPS(TLS 1.3) throughput 차이가 5% 이내일 것 | 현대 CPU의 AES-NI 하드웨어 가속 |

## 실험 설계

### 서버
- nginx 또는 Caddy (TLS 1.2 / 1.3 개별 설정 가능)
- 자체 서명 인증서 (1단계) vs 긴 체인 (3단계)

### 시나리오
1. HTTP (평문) — baseline
2. HTTPS TLS 1.2
3. HTTPS TLS 1.3
4. HTTPS TLS 1.3 + 0-RTT
5. 각각 Keep-Alive ON/OFF

### 측정
- curl `--write-out '%{time_appconnect}'`로 TLS handshake 시간 분리
- k6 `http_req_tls_handshaking` 메트릭
- openssl s_client로 handshake 상세 타이밍
- 인증서 체인 길이별 검증 시간
