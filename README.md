# pfSense OpenVPN 원격접속 & 방화벽 ACL 실습 (SeSAC 2팀, 2026-09)

> 재택 PC가 OpenVPN 암호화 터널로 사내망에 접속 → 방화벽 ACL로 접근 통제 → **비대칭 라우팅 트러블슈팅**까지.
> 4인 팀, 1인 1노드. **내 담당: pfSense 방화벽 · VPN 서버.**
> 원본: 팀 발표자료 `2팀_pfSense_OpenVPN_방화벽ACL_실습.pptx` (2026-09-11, 21장). 팀원 실명은 이 레포에 넣지 않는다.

## 1. 구성

```
              WAN 192.168.0.0/24 (강의실·재택 물리망)
 재택 PC ─────────────┐
 (VPN 100.2)          │ OpenVPN UDP 1194
                ┌─────┴──────┐
                │  pfSense   │  방화벽 · 라우터 · VPN 서버   ← 내 담당
                └──┬──────┬──┘
     LAN 101.0/24  │      │  DMZ 102.0/24
   직원 Win 101.13 ┘      └ Rocky Linux 102.10 (Apache, bWAPP)
```

| 구간 | 대역 | pfSense GW | 주요 호스트 |
|---|---|---|---|
| WAN | 192.168.0.0/24 | 192.168.0.93 | 강의실 공유기, 재택 PC 물리망 |
| LAN | 192.168.101.0/24 | 192.168.101.1 | Windows 직원 PC 192.168.101.13 |
| DMZ | 192.168.102.0/24 | 192.168.102.1 | Rocky Linux 192.168.102.10 (Apache) |
| VPN 터널 | 192.168.100.0/24 | 192.168.100.1 | 재택 가상 IP 192.168.100.2 (자동 할당) |

원칙: 외부 접근은 pfSense 한 곳만 경유. DMZ·LAN 은 기본 차단 후 필요한 통신만 허용(default deny).

### 역할 (4파트 · 1인 1노드)
| 노드 | 담당 | 한 일 |
|---|---|---|
| **① pfSense 방화벽 · VPN 서버** | **나** | WAN·LAN·DMZ 인터페이스, 방화벽 룰, Block private 해제, OpenVPN 서버 + PKI 인증서 발급, `.ovpn` 내보내기 → 재택 배포, **Outbound NAT 로 비대칭 라우팅 해결** |
| ② 재택 PC (OpenVPN Client) | 팀원 | WAN 망에서 `.ovpn` 접속, 가상 IP 확인, 사내망 접근 검증 |
| ③ 내부 직원 PC (Windows, LAN) | 팀원 | 101.13, ICMP 인바운드 허용, (선택) RDP |
| ④ 내부 서버 (Rocky, DMZ) | 팀원 | Apache, firewalld 80·22, access.log 모니터링, bWAPP |

## 2. 구축 단계 (내 노드: pfSense)

### Phase 1 — 인터페이스
```
# Interfaces > Assign
WAN (em0) : DHCP  (강의실 공유기, 예: 192.168.0.93)
LAN (em1) : 192.168.101.1/24    # 직원망 GW
DMZ (em2) : 192.168.102.1/24    # 서버망 GW (OPT1 > Enable + Static)

# Interfaces > WAN  (강의실이 사설망이라 해제)
[ ] Block private networks and loopback
[ ] Block bogon networks
```
WAN 이 사설 대역(강의실)이라 기본 "사설망 차단"을 풀어야 재택 PC(192.168.0.x)가 들어온다.

### Phase 2 — 방화벽 기본 규칙
```
LAN     : PASS  LAN subnets -> any            (기본 허용)
DMZ     : PASS  DMZ subnets -> any            (DMZ 는 기본 차단이라 반드시 추가)
WAN     : PASS  UDP any -> WAN address:1194   (OpenVPN)
OpenVPN : PASS  any -> any                    (마법사 자동 → Phase 5 에서 세분화)
```
pfSense 룰은 **위에서부터 처음 맞는 하나만** 적용(Top-Down).

### Phase 3 — OpenVPN 서버 (Wizard + PKI)
```
CA          : VPN-CA              # 팀 자체 인증기관
Server Cert : Server-Cert
Interface   : WAN   Protocol : UDP / 1194
Server Mode : Remote Access (SSL/TLS + User Auth)
IPv4 Tunnel Network   : 192.168.100.0/24
IPv4 Local Network(s) : 192.168.101.0/24,192.168.102.0/24   # 쉼표 뒤 공백 X → 102 라우팅 누락 주의
# 배포: Package Manager > openvpn-client-export → VPN > Client Export > user01.ovpn
```
- PKI: VPN-CA 가 서버·클라이언트 인증서에 서명 → **양방향 인증** 후 터널 개설. 분실·퇴사 시 CRL 로 폐기.
- 실수 포인트: Local Network 를 쉼표 뒤 공백과 함께 쓰면 두 번째 대역(102)이 push 되지 않아 DMZ 라우팅이 빠짐.

### Phase 4 — 접속 검증 (재택 PC)
```
ipconfig                      # 192.168.100.2 (터널)
ping 192.168.101.13           # Reply, TTL 127
curl http://192.168.102.10    # Apache 페이지
```
주의: 재택 PC 어댑터를 LAN(101)에 꽂으면 대역 충돌 — 101.0/24 가 온링크와 터널에 중복.

### Phase 5 — 보안 ACL (OpenVPN 인터페이스)
| 순서 | 동작 | 규칙 | 의미 |
|---|---|---|---|
| ① | PASS | TCP any → 192.168.102.10:80 | 웹서버만 허용 |
| ② | BLOCK | any → LAN subnets | 직원망 전면 차단 |
| ③ | DISABLED | 기존 any → any | 비활성화 |

검증: `curl http://192.168.102.10` → 응답(①), `ping 192.168.101.13` → 요청 만료(②).

## 3. 트러블슈팅 — 비대칭 라우팅 (내 노드에서 해결)
상세 분석: [docs/트러블슈팅_비대칭라우팅.md](docs/트러블슈팅_비대칭라우팅.md)

| | 내용 |
|---|---|
| 증상 | 방화벽 룰은 Pass, States 에 트래픽도 보이는데 재택 PC → `http://192.168.102.10` **타임아웃** |
| 원인 | 요청(100.2 → pfSense → Rocky)은 도착, **Rocky 의 응답이 VPN 대역으로 돌아가는 경로가 달라 유실** (방화벽 룰 문제 아님) |
| 해결 (실제로 한 것) | `Firewall > NAT > Outbound` 를 **Hybrid** 로, VPN 대역 100.0/24 → DMZ 주소 102.1 로 마스커레이드 |
| 대가 | Rocky 로그에 진짜 출발지(100.2)가 안 남고 102.1 만 남음 → 추적·탐지 불리 |
| 정석 | 라우팅으로 응답 경로를 맞춤 → 로그에 100.2 보존 |
| 당시 결론 | **실습·시연 = NAT(간편), 정석·로그 보존 = 라우팅. 로그 분석(SOC)이 목적이면 라우팅이 유리** |

```
# Firewall > NAT > Outbound
Mode        : Hybrid Outbound NAT rule generation
[Add] (맨 위)
Interface   : DMZ        Protocol : any
Source      : 192.168.100.0/24   # VPN 터널
Destination : any
Translation : DMZ address (192.168.102.1)
```

### 함께 겪은 문제: 호스트 PC 의 Windows 프록시 → ping 응답 없음
VM 을 돌리던 호스트 PC(Windows)에 설정 프록시가 켜져 있었고 ping 이 나가기만 하고 돌아오지 않았는데, **프록시를 끄니 해결**. 단 Windows 설정 프록시는 HTTP·HTTPS 에만 적용되고 ping(ICMP)과는 무관해서, 같은 시점의 다른 변경이 실제 원인이었을 가능성이 있다(미확인). 정리: [docs/트러블슈팅_비대칭라우팅.md §3](docs/트러블슈팅_비대칭라우팅.md).

pfSense 최신 UI 명칭 차이: `Single host or alias` → `Address or Alias`, `LAN net` → `LAN subnets`. 웹이 안 열리면 `Services > Snort > Blocked` 도 확인.

## 4. 2일차 계획 (발표 시점 기준 "예정")
- ACL 확장: RDP(3389)만 허용 시나리오
- 암호화 검증: WAN 구간 Wireshark 로 OpenVPN(UDP 1194) 암호화 vs 내부 평문 대조
- 라우팅 증빙: 재택 `route print`, Rocky `ip route`
- Snort 탐지 룰: [configs/snort_local.rules](configs/snort_local.rules) (SYN 스캔, SSH 브루트포스, ICMP Flood, 외부 DNS 직접 질의)
- 애드온 설계: pfSense **Squid 투명 프록시**(80 → 3128, access.log 로 URL 가시화, 도메인 블랙리스트) + DMZ **bWAPP** 공격(SQLi·XSS·브루트포스) → Squid·Apache 로그 상관분석
- 결과 캡처 체크리스트 12칸은 발표자료에서 "캡쳐 예정" 상태 — 캡처가 있으면 `docs/screenshots/` 에 추가

## 5. 배운 것
1. **방화벽 룰이 통과인데 안 되면 경로를 의심한다.** States 에 세션이 있는데 응답이 없으면 비대칭 라우팅.
2. **한 번에 하나만 바꾸고 재현으로 확정한다.** 프록시를 끄니 됐다고 프록시가 원인이라는 보장은 없다(ICMP 는 프록시를 안 탐). 끄고 켜서 재현돼야 원인.
3. **NAT 로 덮으면 빨리 되지만 출발지 IP 를 잃는다.** 관제에서는 그 IP 가 탐지·추적의 핵심.
4. 같은 원리를 2026-11 팀 프로젝트에서 다시 만남 — 웹 앱 로그의 client_ip 가 전부 127.0.0.1(중간 장비가 출발지를 바꿈) → mod_remoteip 로 복원.
5. Top-Down 룰 순서, 사설망 차단 해제, OpenVPN Local Network 쉼표 표기처럼 **작은 설정 하나가 연결 전체를 막는다.**
