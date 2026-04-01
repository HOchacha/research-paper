# Mitigating IoT Botnet DDoS Attacks through MUD and eBPF based Traffic Filtering

## 기본 정보
- **저자**: Angelo Feraudo, Diana Andreea Popescu, Poonam Yadav, Richard Mortier, Paolo Bellavista
- **소속**: University of Bologna / University of Cambridge / University of York
- **게재**: ICDCN '24 (25th International Conference on Distributed Computing and Networking), January 2024
- **DOI**: 10.1145/3631461.3631549
- **키워드**: DDoS, IoT Botnet, MUD (Manufacturer Usage Description), eBPF, XDP, iptables, Traffic Filtering

---

## 1. 연구 배경 및 문제 정의

### IoT 봇넷 DDoS 문제
- IoT 기기는 인터넷에 상시 연결되어 있으면서 보안이 취약 → 봇넷에 모집되어 DDoS 공격에 악용
- 기기가 해킹되어도 정상 기능은 유지되면서 대량의 악성 트래픽을 생성
- 대표 사례: Mirai 봇넷 — 기본/약한 로그인 자격 증명을 무차별 대입으로 공격

### MUD (Manufacturer Usage Description) 표준 — RFC 8520
- IETF가 제안한 IoT 보안 표준
- 제조업체가 기기의 **허용된 통신 패턴**을 기계 판독 가능한 파일로 정의
- 홈 라우터에서 해당 규칙을 시행하여 기기가 임의의 인터넷 트래픽을 생성하지 못하게 차단

### MUD의 한계
- 기존 MUD는 **주소 기반 필터링**만 지원 (출발지/목적지 IP, 포트, 프로토콜)
- **트래픽 속도(rate) 제한 기능 부재** → DDoS 공격 시 허용된 목적지로의 대량 트래픽을 막을 수 없음
- 자원 제한적인 홈 라우터에서 효율적인 규칙 시행 방법 필요

---

## 2. 핵심 기여

| # | 기여 | 설명 |
|---|------|------|
| 1 | **MUD 표준 확장** | packet-rate / byte-rate 제한을 ACE(Access Control Entry) 액션에 추가 |
| 2 | **osMUD 확장** | VM 환경 및 실제 Linux 기반 라우터에서 배포 가능하도록 개선 |
| 3 | **eBPF/iptables 백엔드** | XDP 기반 eBPF-IoT-MUD 어댑터 + iptables 어댑터 개발 |
| 4 | **봇넷 공격 대응 평가** | SYN Flood 공격 트래픽에 대한 정량적 효과 검증 |

---

## 3. MUD 확장 설계

### Rate-Limiting 확장
- ACE의 **action 그룹**에 두 필드 추가:
  - `packet-rate`: 시간 단위당 최대 패킷 수
  - `byte-rate`: 시간 단위당 최대 바이트 수
- 메트릭 단위: second, minute, hour, day
- **허용된 통신(목적지)별로 개별 설정** 가능 → 세밀한 정책

### 임계값 학습 (81개 IoT 기기, 6개 카테고리 분석)
- Ren et al.의 데이터셋 (81개 기기) 기반, 60초 윈도우로 분석

| 카테고리 | TCP 평균(pkts) | TCP 피크(pkts) | UDP 평균(pkts) | UDP 피크(pkts) |
|--------|-------------|-------------|-------------|-------------|
| appliances | 36.7 | 223.2 | 5.0 | 140.3 |
| smart-hubs | 21.3 | 1716.8 | 9.6 | 152.4 |
| cameras | 62.2 | 1471.3 | 94.4 | 7863.0 |
| audio | 52.0 | 6687.5 | 293.6 | 1837.6 |
| home-aut | 5.2 | 702.3 | 14.5 | 127.5 |

### 두 가지 정책
- **Average 정책**: 평균 트래픽 기준 → 정상 트래픽도 상당량 차단 (smart-hubs 80%+ 드롭)
- **Peak 정책**: 피크 트래픽 기준 → 정상 트래픽 영향 최소 (<1.5% 드롭) ← **채택**

---

## 4. 시스템 아키텍처

### 전체 흐름

```
IoT 기기 ─DHCP(MUD-URL)─▶ dnsmasq ─▶ osMUD Manager
                                           │
                                     MUD File Server에서
                                     MUD 파일 다운로드 & 파싱
                                           │
                                    ┌──────┴──────┐
                                    │  Adapter    │
                                    ├─────────────┤
                                    │ eBPF (XDP)  │  ← 신규
                                    │ iptables    │  ← 신규
                                    │ OpenWRT     │  기존
                                    └─────────────┘
                                           │
                                    라우터 방화벽에서
                                    트래픽 필터링/속도 제한
```

### osMUD 확장
- 기존: OpenWRT 전용 → **일반 Linux 기반 라우터, RaspberryPi까지 확장**
- MUD 파서가 `action` 그룹에서 rate-limit 필드를 인식하도록 수정
- 어댑터 패턴으로 방화벽 백엔드와 분리

---

## 5. eBPF-IoT-MUD 어댑터

### XDP 기반 구현
- **두 프로그램**:
  - `xdpfw_from_device`: LAN 인터페이스에 부착 → IoT → 인터넷 트래픽 필터링 (DDoS 방지)
  - `xdpfw_to_device`: WAN 인터페이스에 부착 → 인터넷 → IoT 트래픽 필터링 (침입 방지)

### eBPF Map 구조
```c
struct flow_key_ipv4 {
    __u8  ip_address_src[4];
    __u8  ip_address_dst[4];
    enum  addr_type type;       // from_device / to_device
    enum  port_protocol proto;  // TCP / UDP
    __u16 port;
};

struct counters_rate {
    __u64 packets;           // 현재 윈도우 내 패킷 수
    __u64 bytes;             // 현재 윈도우 내 바이트 수
    __u64 max_pkt_rate;      // MUD 파일의 최대 패킷 속도
    __u64 max_bytes_rate;    // MUD 파일의 최대 바이트 속도
};
```

### 패킷 처리 로직
1. Ethernet → IP → Transport 헤더 파싱
2. 헤더 정보로 key 생성 → allowlist(BPF_MAP_TYPE_HASH) 조회
3. **key 없음** → `XDP_DROP` (MUD에 없는 통신)
4. **key 있음** →
   - 현재 시간 윈도우 확인 (만료 시 통계 리셋)
   - 패킷/바이트 카운터 업데이트
   - max_pkt_rate 또는 max_bytes_rate 초과 시 → `XDP_DROP`
   - 정상 → `XDP_PASS`

### 추가 Map
- `BPF_MAP_TYPE_ARRAY`: 현재 시간 저장 (윈도우 관리)
- `BPF_MAP_TYPE_HASH`: 윈도우 크기 설정 (기본 1분)

---

## 6. iptables 어댑터

- FORWARD 체인에서 MUD 규칙 시행
- **커스텀 체인** 사용 → MUD 정책만 독립 관리 (시스템 규칙과 분리)
- Rate-limiting: **hashlimit 모듈** 사용 (연결 그룹별 속도 제한)
- `--limit-burst` 파라미터로 소규모 버스트 허용 (기본 5패킷)

---

## 7. 평가 결과

### 실험 환경
- VirtualBox 위 Linux VM: VMMUD(라우터) + IoT-1(IoT 기기) + SERVER(인터넷)
- 정상 트래픽: Sivanathan et al. 데이터셋 (28개 IoT 기기, 6개월)
- 비정상 트래픽: Network TON_IoT 데이터셋 (SYN Flood 공격)

### eBPF 마이크로벤치마크

| 연산 | Min (ns) | 평균 (ns) | 90th (ns) | Max (ns) |
|------|---------|---------|----------|---------|
| 규칙 삽입 (255개) | 3,740 | 4,533 | 5,087 | 15,037 |
| 규칙 삭제 (255개) | 3,426 | 4,577 | 4,504 | 49,581 |
| 패킷 처리 경로 | 140 | 556 | 1,787 | 24,915 |

### 패킷 지연 비교 (netperf TCP_RR)

| 실험 | Min 지연(μs) | 평균 지연(μs) | Txns/s |
|------|-----------|-----------|--------|
| No firewall | 186.4 | 368.2 | 2,714 |
| iptables | 187.0 | 345.5 | 2,893 |
| **eBPF-IoT-MUD** | **188.0** | **342.7** | **2,916** |

→ **eBPF 사용 시 패킷 처리에 추가 오버헤드가 거의 없음**

### 정상 트래픽에 대한 영향 (Peak 정책)

| 카테고리 | 패킷 드롭률 | 바이트 드롭률 |
|--------|----------|----------|
| appliances | 0.2% | 1.18% |
| smart-hubs | 0.01% | 0.03% |

→ **정상 트래픽에 대한 영향 무시 가능 수준**

### 비정상 트래픽(SYN Flood) 차단
- 공격 트래픽이 MUD 파일에 없는 서비스 포트 사용 → **완전 차단**
- 허용된 목적지로의 과도한 트래픽도 rate-limit으로 제한
- eBPF: rate-limit 도달 시 즉시 드롭 → 평탄한 트래픽 그래프
- iptables: limit-burst 허용으로 약간의 변동, but 여전히 제한 내

---

## 8. 기존 솔루션 비교

| 솔루션 | 네트워크 | Rate-limiting | MUD 준수 | 이상 탐지 | 규칙 시행 기술 |
|--------|---------|-------------|---------|---------|------------|
| **본 논문** | **스마트홈** | **Yes (MUD 확장)** | **Yes** | No | **eBPF/iptables** |
| [16] | 산업 | No | Yes (MUD manager 없음) | No | P4 |
| [14] | 스마트홈 | No | Yes (MUD manager 없음) | Yes (ML) | OpenFlow |
| [34] | 산업 | No | Yes (MUD manager 없음) | No | OpenFlow |
| [13] | 스마트홈 | No | No | Yes (VirusTotal) | OpenWRT |
| IoTrim [28] | 스마트홈 | No | No | No (whitelist) | iptables |

→ **최초로 MUD 준수 네트워크에서 rate-limiting을 eBPF/XDP로 시행하고 평가**

---

## 9. 한계 및 향후 연구

| 항목 | 설명 |
|------|------|
| **트래픽 셰이핑 공격** | rate-limit만으로는 속도를 조절하는 저속 DDoS를 탐지 불가 → 다른 봇넷 탐지 기법 필요 |
| **임계값 설정** | 사용자/제조업체가 적절한 rate-limit 값을 설정해야 하는 별도 문제 |
| **lateral movement** | 현재 osMUD는 same-manufacturer/controller 규칙 미지원 → 내부 이동 공격 취약 |
| **향후 계획** | Raspberry Pi 등 실제 하드웨어에서 배포 + ML 기반 자동 rate-limit 값 학습 |

---

## 10. 핵심 기여 요약

1. **MUD 표준에 rate-limiting 확장**: packet-rate/byte-rate를 ACE 액션에 추가, 허용된 목적지별 개별 설정
2. **eBPF-IoT-MUD**: XDP 기반 방화벽으로 Linux 커널 최저 레이어에서 패킷 필터링 + 속도 제한, 패킷 처리 평균 556ns
3. **양쪽 백엔드 구현**: eBPF와 iptables 모두 지원, 범용 Linux 라우터/Raspberry Pi에서 배포 가능
4. **정상 트래픽 최소 영향**: Peak 정책 시 드롭률 <1.5%, 패킷 지연 추가 오버헤드 없음
5. **SYN Flood 공격 효과적 차단**: 실제 공격 데이터셋으로 검증
