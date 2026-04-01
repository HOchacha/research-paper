# Hawkeye: Diagnosing RDMA Network Performance Anomalies with PFC Provenance

## 기본 정보
| 항목 | 내용 |
|------|------|
| **제목** | Hawkeye: Diagnosing RDMA Network Performance Anomalies with PFC Provenance |
| **저자** | Shicheng Wang, Menghao Zhang, Xiao Li, Qiyang Peng, Haoyuan Yu, Zhiliang Wang, Mingwei Xu, Xiaohe Hu, Jiahai Yang, Xingang Shi |
| **소속** | Tsinghua University (BNRist) / Beihang University / Infrawaves / Quan Cheng Laboratory |
| **학회** | ACM SIGCOMM 2025 (September 8–11, 2025, Coimbra, Portugal) |
| **DOI** | 10.1145/3718958.3750490 |
| **키워드** | RDMA Networks, Performance Diagnosis, Programmable Networks, Network Provenance |
| **오픈소스** | https://github.com/wangshicheng1225/Hawkeye |

---

## 연구 배경 및 문제

### RDMA와 PFC의 보급
- RDMA(Remote Direct Memory Access)는 초저지연·초고처리량 네트워크 기술로, RoCEv2를 통해 대규모 데이터센터에서 점점 확대
- 활용 분야: 분산 딥러닝 학습, 클라우드 스토리지, 멀티 테넌트 퍼블릭 클라우드
- RDMA 네트워크는 무손실(lossless) 전송을 위해 **PFC(Priority Flow Control)** 배포
  - 스위치 ingress 큐가 임계값($X_{off}$) 초과 시 "PAUSE" 프레임 전송 → 상위 노드 전송 중지
  - 큐가 임계값($X_{on}$) 이하로 감소 시 "RESUME" 프레임 전송

### PFC가 초래하는 성능 이상(NPA)의 복잡성
PFC는 패킷 손실을 방지하지만, **연쇄적(cascading) 혼잡 확산** 특성으로 새로운 성능 이상을 유발:

| 이상 유형 | 설명 |
|----------|------|
| **PFC backpressure (flow contention)** | incast 마이크로버스트, ECMP 부하 불균형 등으로 혼잡 발생 → PFC가 다중 홉에 걸쳐 혼잡 확산. 혼잡 원인과 큐를 공유하지 않는 무관한 플로우(victim)도 차단됨 |
| **PFC storm (host injection)** | 호스트의 NIC 버그, 버퍼 고갈, PCIe 병목 등으로 PFC 프레임 지속 주입 → flow contention 없이도 광범위/네트워크 전체 트래픽 차단 |
| **PFC deadlock (in-loop)** | 순환 버퍼 의존성(CBD) 내에서 flow contention이 PFC를 순환 전파 → 데드락. 1ms 미만의 짧은 contention이 영구적 데드락 유발 가능 |
| **PFC deadlock (out-of-loop)** | CBD 외부에서 호스트 PFC 주입 또는 flow contention이 루프 내로 혼잡 전파 → 데드락 |

### RDMA NPA 진단의 두 가지 핵심 복잡성
1. **빈번하고 일시적인 큐 혼잡**: RDMA의 line-rate 시작, incast 트래픽, 얕은 스위치 버퍼 → 빈번한 마이크로버스트
2. **PFC 복잡성**: 전통적 TCP와 달리 혼잡이 로컬 큐 내 flow contention뿐 아니라 다운스트림 노드에서 독립적으로 확산 → 기존 로컬 분석 기반 진단 무력화

---

## 기존 연구의 한계

### 1. 세밀한 PFC 가시성 부족
- 호스트 기반(Pingmesh, Dapper 등) 또는 스위치 기반(Sonata, FlowRadar 등) 솔루션 모두 포트 PFC 상태, 플로우별 PFC 영향 등 PFC 관련 정보의 가시성 부재
- PFC watchdog: 폴링 주기가 수백 ms~초 단위 → 일시적 PFC 혼잡 놓침, 포트 수준 모니터링만으로 개별 플로우 영향 파악 불가

### 2. 빠르고 효율적인 PFC 인과관계 추적 부족
- 단일 스위치 내 플로우만 분석하는 시스템(Sonata, *Flow 등) → 다중 홉 너머 PFC 근본 원인 찾기 불가
- 전체 텔레메트리 수집(NetSight) → 수집·진단 오버헤드 과대
- 피해 플로우 경로 스위치만 수집(SpiderMon) → PFC 확산 경로의 일부(특히 개시 노드) 누락 가능
- ITSY: PFC 데드락 루프만 탐지, 비루프 PFC backpressure 진단 불가

### 3. 정확한 이상 진단 부족
- 기존 flow-interaction 패러다임: 피해 플로우와 같은 큐를 공유하는 플로우만 분석
- PFC로 인한 혼잡과 로컬 flow contention 구분 불가
- 근본 원인이 피해 플로우 경로 밖(수 홉 떨어진 곳)에 있을 때 포착 불가
- 예: Figure 1(a)에서 F1의 성능 저하는 SW4.P1의 버스트가 원인이나, F1과 F2의 로컬 contention만 분석하면 진짜 원인 놓침

---

## Hawkeye 설계

### 설계 목표
1. **세밀한 PFC 가시성** 제공: PFC 유발 혼잡과 일반 flow contention 구분, 피해 정도 정량화
2. **효율적 텔레메트리 수집**: 피해 플로우 경로 + PFC 확산 경로의 인과 관계 스위치만 정확히 식별
3. **정확한 이상 식별 및 근본 원인 진단**: PFC 확산 경로, 이상 유형, 근본 원인(마이크로버스트/부하 불균형/호스트 PFC 주입) 종합 파악

### 전체 프레임워크 (3단계)

```
[호스트 감지 에이전트] → polling 패킷 전송
         ↓
[스위치 ASIC] ← 패시브 텔레메트리 기록 (PFC 가시성 + 인과관계)
  - PFC 상태 레지스터, 포트 트래픽 미터, 플로우/포트 텔레메트리
  - 데이터 플레인에서 line-rate PFC 인과관계 분석
  - CPU로 미러링 → 비동기 텔레메트리 수집
         ↓
[분석기] → 프로비넌스 그래프 구축 → 서명 기반 이상 진단
```

---

### 핵심 기법 1: 텔레메트리 기록 (§3.3)

#### PFC 가시성 및 인과관계 인식
- **PFC 상태 레지스터**: 각 포트의 egress 파이프라인에 설치. PFC 프레임 수신 시 포트 상태·잔여 pause 시간 갱신
- **포트 간 트래픽 인과관계 구조** (Figure 3):
  - 각 포트 쌍(InPort×OutPort)에 **트래픽 미터** 설치 → 트래픽 볼륨 기록
  - PFC 혼잡 시, 실제 패킷 축적에 기여하는 인과 포트만 식별 가능
  - 예: P3과 P4 모두 PFC 혼잡이지만, P1→P4 트래픽이 없으면 P4는 P1의 PFC에 무관 → P3만 인과 포트로 식별

#### 에포크 기반 다중 세분화 텔레메트리 (Figure 4)
- **에포크(epoch)**: 링 버퍼 방식으로 egress 파이프라인에서 유지
- 패킷 메타데이터의 48비트 타임스탬프에서 특정 비트 추출하여 에포크 인덱싱 (예: 1ms ≈ $2^{20}$ns → `timestamp[21:20]`)
- 에포크 ID (`timestamp[29:22]`) 로 순서 구분, wrap-around 시 리셋

| 텔레메트리 수준 | 기록 항목 |
|--------------|---------|
| **플로우 수준** | 5-tuple, 패킷 수, egress 큐 깊이, **PFC에 의해 pause된 패킷 수** |
| **포트 수준** | PFC에 의해 pause된 패킷 수, egress 큐 깊이 |

- 플로우 텔레메트리: 5-tuple 해시로 슬롯 인덱싱, XOR로 기존 플로우 여부 확인
- 포트 텔레메트리: 포트 번호로 인덱싱 → 데이터 플레인에서 비용이 큰 플로우→포트 집계 연산 회피

---

### 핵심 기법 2: 텔레메트리 수집 (§3.4)

#### 이상 기반 호스트 감지 에이전트
- **호스트 측 트리거링** (스위치 트리거링 대비 장점):
  1. PFC 연쇄 특성으로 여러 스위치가 동시 감지 → 중복 추적 제거
  2. RTT, FCT, JCT 등 호스트 측 고수준 성능 메트릭 활용 가능
- NVIDIA BlueField-3 DPU 기반 프로토타입: DOCA PCC API로 플로우별 RTT 모니터링
- 성능 저하 감지 시 → **polling 패킷** 생성, PFC에 의해 pause되지 않는 별도 큐로 전송

#### Polling 패킷 설계 (Figure 5)
```
[DST MAC | SRC MAC | ETH TYPE | POLLING FLAG | EVENT ID | VIC FLOW 5-TUPLE]
```
- **Polling Flag** (2비트):
  - `01` (기본): 피해 플로우 경로만 추적
  - `10`: PFC 인과관계만 추적
  - `11`: 양쪽 모두 추적
  - `00`: 추적 불필요

#### 데이터 플레인 내 인과관계 분석 (Figure 6)
라인레이트로 동작하는 재귀적 PFC 인과관계 추적:

1. **피해 플로우 경로 스위치** (flag `01`): polling 패킷을 피해 플로우와 동일 포트로 유니캐스트
   - egress에서 pause된 패킷 수 확인 → PFC 감지 시 flag 상위 비트 설정
2. **PFC 인과 스위치** (flag `1*`): ingress에서 모든 egress 포트로 멀티캐스트
   - 각 포트가 인과관계 구조(Figure 3) 확인 → 인과 관련 포트만 polling 패킷 전송
3. **종료 조건**: egress 포트가 호스트 연결이거나 pause된 패킷 없음 → 각각 호스트 PFC 주입 또는 로컬 flow contention 의미
4. 각 스위치는 polling 패킷을 CPU 포트로 미러링 → 비동기 텔레메트리 수집 트리거

#### 컨트롤러 지원 데이터 수집
- 데이터 플레인 패킷 생성의 한계: 스테이지당 1회 메모리 접근, 제한된 PHV 크기 → 많은 재순환·패킷 필요
- **스위치 CPU**가 비동기적으로 레지스터 읽기 → 무의미한 데이터(0 값 등) 필터링 → MTU 크기 패킷으로 집약 보고
- Tofino의 `REGISTER_SYNC` 테이블 연산(DMA 전송)으로 레지스터 배열 고속 동기화

---

### 핵심 기법 3: 프로비넌스 기반 진단 (§3.5)

#### 3.5.1 이종 Wait-for 프로비넌스 그래프 구축

플로우와 포트를 노드로, 방향성 가중 에지로 wait-for 관계를 표현하는 이종 그래프:

**① 포트 수준 에지 (PFC 인과관계)**
- PFC에 의해 pause된 포트 $P_i$ → 다운스트림 혼잡 포트 $P_j$로의 wait-for 에지
- 에지 가중치: $w_{ij} = paused\_num[P_i] \times \frac{meter[P_i][P_j]}{\sum_k meter[P_i][P_k]} \times qdepth[P_j]$
- 의미: $P_j$의 큐 깊이와 $P_i \to P_j$ 트래픽 비율에 비례한 PFC 혼잡 기여도

**② 플로우→포트 에지 (PFC pause 영향)**
- PFC pause된 포트 $P_j$를 통과하는 플로우 $f_i$ → $P_j$로의 에지
- 가중치: $paused\_num(f_i, P_j)$ (해당 포트에서 $f_i$의 pause된 패킷 수)

**③ 포트→플로우 에지 (flow contention 기여도)**
- 포트 내 flow contention에 대한 각 플로우의 기여도
- 플로우 $f_j$의 기여도: $\sum_i w(f_i, f_j) - \sum_k w(f_j, f_k)$
  - **양수** → contention 기여자 (contributor)
  - **음수** → 피해자 (victim)
- PFC pause와 flow contention 동시 발생 시, pause된 패킷은 port-flow 에지 구축에서 제외

#### 3.5.2 서명 기반 이상 진단 (Table 2)

프로비넌스 그래프를 순회하며 미리 정의된 **이상 서명(signature)**과 매칭:

| 이상 유형 | 근본 원인 | 그래프 서명 특징 |
|----------|---------|--------------|
| **마이크로버스트 incast** | Flow contention | PFC 경로 존재, 끝 노드(out-degree=0)에서 양수 가중치 port→flow 에지 + burst-flow 판별 |
| **In-loop deadlock** | Flow contention | 포트 수준 서브그래프에 **루프** 존재, 루프 내 모든 포트 out-degree=1, 루프 내 한 포트에서 flow contention |
| **Out-of-loop deadlock** | Flow contention | 루프 존재, 루프 내 한 포트 out-degree>1, 루프 외부 경로 끝에서 flow contention |
| **Out-of-loop deadlock** | Host PFC injection | 루프 존재, 루프 내 한 포트 out-degree>1, 루프 외부 경로 끝에서 flow contention 없음 (모든 port→flow 가중치 ≤ 0) |
| **PFC storm** | Host PFC injection | PFC 경로 존재, 끝 노드에서 flow contention 없음 |
| **일반 flow contention** | Flow contention | 포트 간 에지 없음 (PFC 경로 없음), 해당 포트에서 양수 port→flow 에지 존재 |

진단 절차 (Algorithm 2):
1. 피해 플로우 경로의 각 포트 확인
2. PFC pause 발견 시 → `CheckPortNode()`: DFS로 포트 수준 서브그래프 탐색
3. 루프 발견 → `DeadlockDiagnose()`: in-loop vs out-of-loop 분류, 근본 원인 분석
4. PFC 경로 끝(out-degree=0) → `AnalyzeFlowContention()`: flow contention vs host PFC injection 판별
5. PFC 없는 포트 → 전통적 flow contention 진단

---

## 구현

| 구성요소 | 구현 상세 |
|---------|---------|
| **Tofino 하드웨어** | ~2,500 lines P4 + ~3,000 lines C |
| **NS-3 시뮬레이션** | HPCC 시뮬레이터 기반 |
| **근본 원인 분석기** | Python |
| **호스트 감지 에이전트** | NVIDIA BlueField-3 DPU, DOCA PCC 기반 |

### Tofino에서 PFC 인식 활성화
- 기본적으로 PFC 프레임은 MAC에서 필터링됨 → P4 파이프라인에 도달 불가
- `rxconfig` 레지스터의 `filterpf` 비트를 0으로 설정 → PFC 프레임을 P4 애플리케이션 로직으로 전달
- 차세대 RDMA 네이티브 스위치는 PFC 모니터링 인터페이스를 기본 제공하여 구현 용이

---

## 평가

### 실험 환경
- **시뮬레이션**: Fat-Tree (K=4) 토폴로지, 20개 스위치, 100Gbps 링크, 2μs 링크 지연
- **하드웨어 테스트베드**: Tofino 스위치 2 파이프라인을 2개 논리 스위치로 분할, Dell PowerEdge R7525 서버 2대
- **워크로드**: 실제 데이터센터 RoCEv2 트래픽 분포 (long-tailed), Poisson 프로세스 기반

### 진단 정확도

| 비교 대상 | Burst-PFC | PFC storm | In-loop DL | Out-loop DL | Flow contention |
|----------|-----------|-----------|------------|-------------|-----------------|
| **Hawkeye** | >90% P, ~100% R | >90% P, ~100% R | >90% P, ~100% R | >90% P, ~100% R | >90% P, ~100% R |
| Full polling | 동일 수준 | 동일 수준 | 동일 수준 | 동일 수준 | 동일 수준 |
| Victim-only | 근접 | 근접 | **낮음** | **낮음** | 근접 |
| SpiderMon | **낮음** | **낮음** | **낮음** | **낮음** | 높음 |
| NetSight | **낮음** | **낮음** | **낮음** | **낮음** | 높음 |

- 정밀도는 주로 **에포크 크기**에 영향받음: 에포크 증가 → flow contention 식별 정확도 감소
- Recall은 RTT 임계값 초과 시 진단 트리거 → 거의 모든 이상 보고 가능

### 효율성

| 메트릭 | Hawkeye vs 기존 |
|--------|---------------|
| **처리 오버헤드** | Full polling 대비 1~4 자릿수 낮음 |
| **대역폭 오버헤드** | 패시브 모니터링 + anomaly 시에만 polling 패킷 → 매우 낮음 |
| **수집 스위치 수** | Full polling 대비 훨씬 적으면서 인과 스위치 100% 커버리지 |
| **CPU 폴링 시간** | 2~4 에포크 기준 80~120ms (병렬 수집, 스케일 무관) |
| **텔레메트리 크기 감소** | CPU 필터링으로 >80% 감소 |
| **텔레메트리 패킷 수 감소** | CPU 배칭으로 ~95% 감소 |

### 하드웨어 리소스 사용
- Tofino에 적합하게 맞춰짐 (Gateway, Table, ALU, PHV, SRAM, TCAM 모두 30% 이하)
- PFC 인과관계 구조 + 포트 텔레메트리: 포트 수에 바운드된 상수 크기
- 플로우 텔레메트리: $O(\#flow)$로 확장

---

## 핵심 기여 요약

1. **PFC 인식 세밀 텔레메트리**: 패킷 수준 PFC 상태 가시성 + 포트 간 트래픽 인과관계 기록 → PFC 유발 혼잡과 로컬 flow contention 구분 가능
2. **데이터 플레인 내 PFC 인과관계 추적**: 피해 플로우 경로 + PFC 확산 경로를 라인레이트로 재귀적 추적, CPU 비동기 수집 → 필요한 스위치만 정확히 커버
3. **이종 프로비넌스 그래프 + 서명 기반 진단**: 플로우·포트 간 wait-for 관계를 그래프로 구축, 사전 정의된 이상 서명 매칭으로 이상 유형 + 근본 원인 정확 진단

---

## 논의 및 한계

- **부분 배포**: 일부 스위치만 Hawkeye 배포 시 PFC 추적 중단 가능. ToR 스위치에만 flow telemetry 배포하는 절충안 제시
- **커버 이상 범위**: PFC 존재 여부, 근본 원인 유형(flow contention/host injection), PFC 경로 형태(루프 유무)를 커버하는 대표적 이상에 집중. 더 포괄적인 NPA 커버리지는 향후 과제
- **파라미터 설정**: 에포크 크기(짧을수록 정확하나 메모리 증가)와 감지 임계값(보통 최대 RTT의 2~4배) 간 트레이드오프
- **확장성**: 텔레메트리 설계는 기존 flow-level 모니터링 연구와 직교적으로 보완 가능
- **적용성**: Tofino에 한정되지 않으며, 차세대 스위칭 ASIC에 구현 가능 (PFC 상태 모니터링 기본 지원 + 상수 크기 추가 레지스터)
