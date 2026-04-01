# SkeletonHunter: Diagnosing and Localizing Network Failures in Containerized Large Model Training

## 기본 정보
| 항목 | 내용 |
|------|------|
| **제목** | SkeletonHunter: Diagnosing and Localizing Network Failures in Containerized Large Model Training |
| **저자** | Wei Liu, Kun Qian, Zhenhua Li*, Tianyin Xu, Yunhao Liu, Weicheng Wang, Yun Zhang, Jiakang Li, Shuhong Zhu, Xue Li, Hongfei Xu, Fei Feng, Ennan Zhai* |
| **소속** | Tsinghua University / Alibaba Cloud / UIUC |
| **학회** | ACM SIGCOMM 2025 (September 8–11, 2025, Coimbra, Portugal) |
| **DOI** | 10.1145/3718958.3750513 |
| **키워드** | Containerized Large Model Training, AI Infrastructure, Serverless Computing, Large-scale Network Monitoring and Troubleshooting, Network Reliability, Traffic Pattern Analysis |

---

## 연구 배경 및 문제

### 컨테이너화된 대규모 모델 학습
- 대규모 LLM 학습(GPT-3, Llama 3, Qwen 3, DeepSeek V3 등)은 막대한 컴퓨팅·네트워킹 자원을 요구
- **컨테이너 기반 학습**이 경량성, 이식성, 격리성 덕분에 서버리스 컴퓨팅과 함께 대중화
  - 사용자가 학습 코드·데이터·의존성을 컨테이너로 번들링하여 공유 RNIC/GPU 위에서 실행
- Alibaba Cloud: **40K+ RNIC, 40K+ GPU** 규모의 컨테이너 학습 클라우드를 3년 이상 운영, **~5M 학습 작업** 처리

### 네트워크 인프라의 중요성
- RoCE(RDMA over Converged Ethernet) 네트워크: RTT < 20µs, 패킷 손실 0 기대
- 학습 작업은 고도로 동기적 → **RTT 10µs 증가만으로도 ~20% 학습 속도 저하**
- 연결 문제가 4초 이상 지속 시 → 집합 통신(collective communication) 타임아웃 → **전체 학습 실패**

### 인프라 구조
```
┌─ Host A ──────────────────────────────────────┐
│  Container Runtime                             │
│  ┌─ Tenant A ─┐  ┌─ Tenant B ─┐               │
│  │  GPU  GPU  │  │  GPU  GPU  │               │
│  └────────────┘  └────────────┘               │
│       ↕ vPorts                                 │
│  ┌─ Virtual Switch (OVS/VXLAN) ─┐  ← Overlay  │
│       ↕ vPorts                                 │
│  ┌─ NIC (RNIC × 8, SR-IOV) ────┐              │
└───────────┼────────────────────────────────────┘
            │  ← Underlay
   Physical Switches (Data Center Topology)
```
- **Overlay**: VXLAN 기반 가상 네트워크 → 테넌트 간 트래픽 격리
- **Underlay**: 물리 스위치 네트워크
- OVS(Open vSwitch)가 overlay↔underlay 간 패킷 포워딩 제어, 대부분의 캡슐화/포워딩은 RNIC에 오프로드

---

## 핵심 도전 과제 (3가지 + 곱셈 효과)

### Challenge 1: 컨테이너의 높은 동적성
- 컨테이너 수명이 짧음: **50% 이상이 60분 미만**, 70%가 100분 미만 (물리 호스트는 수개월~수년)
- 피크 시간에 **분당 ~2,000개 컨테이너 생성**, 비피크에도 수백 개
- 같은 학습 작업의 컨테이너들도 **상태 전환이 비동기적**: 초기화 시간이 수 분에서 최대 10분까지 차이
- 낮은 사양 컨테이너는 주로 디버깅/테스트 용도 → 수명이 더 짧음

### Challenge 2: 엔드포인트 유발 복잡성
- 컨테이너 1개가 최대 **8개 RNIC**에 바인딩 (GPU당 1개 RNIC 할당이 일반적)
- 83.1%의 컨테이너가 8개 RNIC, 8.9%가 4개 RNIC 사용
- 모니터링해야 할 소스-목적지 쌍이 급격히 증가

### Challenge 3: Overlay-Underlay 상호작용
- 멀티 테넌트 환경에서 overlay 네트워크가 추가됨 → 가상 네트워크 컴포넌트(CNI, OVS, RNIC flow table 등)가 대폭 증가
- 호스트당 flow table 항목 수: 평균 40+, 최대 **9,300개**
- 전통적 물리 네트워크나 VM overlay 대비 장애 위치 파악이 훨씬 어려움

### 곱셈 효과 (Multiplicative Effect)
$$\text{검사 대상} = X \times Y \times Z$$
- $X$: 컨테이너 수 (예: 1K), $Y$: 컨테이너당 RNIC 수 (예: 8), $Z$: RNIC당 가상 네트워크 컴포넌트 수 (예: 16)
- **예: 1K × 8 × 16 = 128K** 네트워크 컴포넌트를 매 학습 라운드(~30초)마다 검사해야 함 → 실질적으로 불가능

---

## 기존 연구의 한계

| 접근 방식 | 대표 시스템 | 한계 |
|----------|----------|------|
| **포괄적 모니터링** (전체 패킷) | EverFlow, Confluo | 네트워크 인프라 대규모 수정 필요. 가상 스위치 등 소프트웨어 컴포넌트 모니터링은 성능 저하 유발 |
| **기회적 샘플링** | PINT, UnivMon | 모든 핵심 경로를 커버하는 것이 보장되지 않음 → false negative 발생 |
| **Pingmesh 계열** | Pingmesh, R-Pingmesh | 멀티 테넌트 컨테이너 환경의 동적 네트워크 토폴로지에 맞지 않음. 특수 하드웨어(IP-in-IP) 필요 |
| **토폴로지 인식** | deTector | 학습 워크로드의 희소성/대칭성 미고려 → 여전히 프로빙 수 과다 (15K+/라운드) |

---

## 핵심 인사이트: 학습 트래픽의 고유한 희소성

### 공간적 희소 분포 (Sparse Spatial Distribution)
- 대규모 모델 학습은 **집합 통신(collective communication)** 패러다임 사용 (NCCL, UCC, MSCCL, MPI)
- 병렬화 전략: **DP(Data Parallelism)**, **TP(Tensor Parallelism)**, **PP(Pipeline Parallelism)**
  - 각 GPU는 같은 병렬화 그룹 내 GPU와만 통신
  - 예: 512 GPU, TP=8, PP=8, DP=8 → RNIC 트래픽 매트릭스가 매우 희소 (Figure 9a)
- **Rail-optimized topology**: 같은 호스트의 서로 다른 RNIC은 서로 다른 ToR 스위치에 연결
  - 크로스-레일 통신은 자동으로 NVLink(인트라호스트) + 같은 레일 인터호스트 전송으로 변환
  - 결과: 네트워크 통신은 **같은 레일 내에서만** 발생

### 시간적 버스트 주기 (Temporal Burst Cycles)
- 학습 트래픽은 강한 **주기적·계절적 패턴** 보유 (Figure 7)
- 각 학습 이터레이션 동안:
  - 모델 레이어 간 전송은 미미 (idle 구간)
  - 이터레이션 종료 후 gradient 동기화 시 all-reduce → **트래픽 버스트** (최대 15 Gbps)
- 서로 다른 DP 그룹에서 **같은 위치의 RNIC은 동일한 버스트 주기**를 보임
  - → RNIC의 "역할"을 버스트 주기 비교로 식별 가능

---

## SkeletonHunter 시스템 설계

### 전체 아키텍처
```
┌──────────────────────────────────────────────────┐
│                 Controller                        │
│  ┌───────────────┐  ┌──────────────────────────┐ │
│  │ Skeleton-      │  │ Reactive Underlay        │ │
│  │ Optimized Ping │  │ Diagnosis                │ │
│  │ List Generation│  │                          │ │
│  └──────┬────────┘  └──────────────────────────┘ │
│         │ Instructions / Ping List                │
└─────────┼────────────────────────────────────────┘
          │  Frontend Network
┌─────────┼────────────────────────────────────────┐
│  Overlay Containers (Sidecar Agents)              │
│  ┌──────┐ ┌──────┐ ┌──────┐                      │
│  │Agent1│ │Agent2│ │Agent3│ ...                   │
│  └──┬───┘ └──┬───┘ └──┬───┘                      │
│     │ Probing/Telemetry Data                      │
└─────┼────────────────────────────────────────────┘
          │  Backend Network
┌─────────┼────────────────────────────────────────┐
│                  Analyzer                         │
│  ┌──────────────┐ ┌──────────────┐ ┌───────────┐│
│  │Traffic Skeleton│ │Anomaly       │ │Overlay-   ││
│  │Inference      │ │Detection     │ │Underlay   ││
│  │               │ │              │ │Disentangle││
│  └──────────────┘ └──────────────┘ └───────────┘│
└──────────────────────────────────────────────────┘
```

### 핵심 아이디어: Traffic Skeleton 추론
- CSP는 프라이버시상 테넌트의 모델 구성을 알 수 없음
- 대안: RNIC의 **처리량 버스트 주기**로부터 **traffic skeleton**(학습 트래픽이 일관되게 통과하는 핵심 네트워크 경로 집합)을 추론
- 추론된 skeleton 기반으로 최적 프로빙 매트릭스 구성 → 최소 리소스로 장애 탐지

---

## 구성요소 설계

### 5.1 Traffic Skeleton Inference (3단계)

#### Phase 1: Preload — 기본 Ping List 생성
- Rail-optimized topology 특성 활용: 네트워크 통신은 **같은 레일(같은 rank의 RNIC) 내**에서만 발생
- Full-mesh ping list에서 **다른 레일 간 쌍을 제거** → 호스트당 8 RNIC일 때 **8× 규모 축소**
- 사용자가 학습 작업 제출 시 즉시 수행 (컨테이너 초기화 이전)

#### Phase 2: Initialization — 점진적 Ping List 활성화
- 문제: 초기화 중 컨테이너가 네트워크 스택 미완료 → 오탐(false positive) 유발
- 해결: **데이터 플레인 자체에서** ping list 활성화 처리 (controller 병목 회피)
  - 컨테이너 생성 시 agent가 controller로부터 기본 ping list 수신, 단 즉시 프로빙 시작하지 않음
  - 다른 컨테이너가 ready 시 자신을 **등록(register)** → 대상 컨테이너의 ping list에서 해당 항목 활성화
  - $O(1)$ 시간 복잡도로 확장 가능

#### Phase 3: Runtime — Traffic Skeleton 기반 최적화
- 기본 ping list에도 비활성 쌍이 많음 (예: 512-GPU 작업에서 GPU당 64개 목적지 중 실제 연결은 9개 → 85.9% 추가 최적화 여지)
- **STFT(Short-Time Fourier Transform)** 로 RNIC 버스트 주기의 주파수 도메인 특성 추출
  - Wavelet Transform, DFT 대비 시변 특성을 가장 낮은 계산 복잡도로 포착
- **계층적 클러스터링(hierarchical clustering)** 으로 유사한 STFT 특성을 가진 RNIC 그룹화

클러스터링 제약 조건:

$$\min \quad \sigma^2 = \frac{1}{k} \sum_{i=1}^{k} (\|c_i\| - \bar{c})^2$$

$$\text{s.t.} \quad N \bmod \lfloor\bar{c}\rceil = 0$$

$$r_1, r_2, \cdots r_x \in H_r \Rightarrow \forall c_i, \|c_i \cap H_r\| \leq 1$$

- 식 (1): 그룹 내 RNIC 수의 분산 최소화 (각 파이프라인 동일 규모 보장)
- 식 (2): 그룹 수가 총 RNIC 수의 약수 (DP 균등 분할)
- 식 (3): 같은 호스트의 RNIC은 같은 그룹에 속하지 않음 (NVLink 가속을 위해 같은 DP에 배치되므로)

**DP 그룹 추론 → TP/PP 추론**:
- $TP \times PP = N / \lfloor\bar{c}\rceil$
- 버스트 주기의 **시간 시프트**를 이용하여 PP 레벨 구분 (앞단 PP가 먼저 버스트)
- 최종: **>95% 추가 ping list 축소** 달성

---

### 5.2 Connectivity Anomaly Detection

#### 단기 지연 이상 탐지 (30초 윈도우)
- 각 RNIC 엔드포인트 쌍의 지연 데이터를 30초 단위로 집계
- 지연 분포를 7개 통계량으로 기술: P25, P50, P75, min, mean, std, max
- **LOF(Local Outlier Factor)** 적용: 밀도 기반 이상 탐지
  - 5분 look-back 윈도우로 LOF 계산 → 버스트 주기 커버
  - 새 5분 윈도우가 이전 윈도우와 클러스터링 불가 시 → 이상 판정

#### 장기 지연 이상 탐지 (30분 윈도우)
- **목적**: 단기 분석에서 놓칠 수 있는 점진적 성능 저하 탐지
- 정상 작동 시 RNIC 쌍의 지연 데이터는 **로그정규 분포(log-normal distribution)** 따름

$$Y = \ln(X) \sim N(\mu, \sigma^2)$$

- 시점 $T$에서 파라미터 추정 → $T+0.5h$, $T+1h$, $T+1.5h$에서 **Z-test** 적용
  - 추정 분포에서 벗어나면 이상 판정

---

### 5.3 Optimistic Overlay-Underlay Disentanglement

장애를 탐지한 후, 문제가 되는 네트워크 컴포넌트를 특정하기 위한 **낙관적 분리 진단**:

#### 핵심 가정
> Overlay(소프트웨어)와 Underlay(하드웨어) 장애의 근본 원인은 서로 전파되지 않는다

#### Algorithm 1: 장애 분리 및 위치 특정

```
1. 경로 분리: 두 컨테이너 간 경로(P_AB)를 overlay 링크(L_O)와 underlay 링크(L_U)로 분리
   - 패킷이 캡슐화되어 있으면 → Overlay
   - 아니면 → Underlay

2. Overlay 논리적 도달 가능성 분석:
   - 포워딩 체인을 순차적으로 재현
   - 도달 불가(null) → 해당 링크가 overlay 장애 지점
   - 이미 방문한 노드에 도달 → 루프 발생 (잘못된 포워딩 규칙)

3. Underlay 물리적 교차 분석:
   - ECMP 경로 다중화로 인해 network tomography 기법 사용
   - 다수의 장애 경로에서 공통으로 나타나는 물리 링크에 투표(vote)
   - 가장 많이 등장하는 링크 → underlay 장애 지점

4. RNIC 검증 (두 레이어 모두 장애 불발견 시):
   - OVS에서 RNIC으로 오프로드된 flow table 덤프
   - Overlay-Underlay 간 flow table 불일치 탐지
   - 불일치 미발견 시 → 수동 점검
```

#### 낙관적 접근의 한계
- Underlay RNIC의 비정상 동작이 overlay 가상 스위치 오설정을 유발하는 드문 사례 존재
- 이 경우 수동 해결 필요하나, 실제 운영 환경에서 매우 드묾

---

## 구현

| 구성요소 | 구현 상세 |
|---------|---------|
| **Controller** | Java 6,350 LoC. 프론트엔드 네트워크의 서버 2대 (로드밸런싱 + 장애 허용). 에이전트와의 모든 통신 암호화 |
| **Agent (Overlay)** | Go 5,400 LoC. 학습 컨테이너와 함께 **사이드카 컨테이너**로 배포. 네트워크 네임스페이스 공유, 프로세스 격리 |
| **Agent (Underlay)** | Go. 호스트에 독립 컨테이너로 배포. 호스트 전체 리소스 접근 가능 |
| **Analyzer** | Alibaba Cloud 로그 서비스 + 실시간 컴퓨팅 서비스(Apache Flink). 피드백 루프로 controller에 결과 전달 및 알람 트리거 |

---

## 평가

### 배포 환경
- 대규모 프로덕션 클러스터: **4K+ 물리 호스트**, 호스트당 8 RNIC (200/400 Gbps)
- RNIC: SR-IOV 모드, 128 VF(Virtual Function)/RNIC
- 평가 기간: 6개월 (2024.3–8), **2M+ 학습 작업**

### 7.1 효율성 및 효과성

#### 프로빙 규모 축소

| RNIC 수 | Full-mesh 프로빙 | SkeletonHunter-Basic | SkeletonHunter (최종) |
|---------|---------------|---------------------|---------------------|
| 2,048 | 60,430.32 | ~7,500 (8× 축소) | **2,593.03** |

- 모든 RNIC 구성에서 full-mesh 대비 **1자릿수 이상** 축소
- 기본 ping list 대비 최종 skeleton 기반 ping list: **>95% 추가 축소**

#### 프로빙 시간

| RNIC 수 | Full-mesh | Basic | SkeletonHunter |
|---------|-----------|-------|----------------|
| 512 | 560.25s | 64.85s | **8.23s** |
| 1,024 | 1,123.43s | 122.54s | **16.91s** |
| 2,048 | 2,034.12s | 240.54s | **25.09s** |

- Basic 대비 최종: **87.3~89.6% 추가 시간 절감**
- 학습 라운드(~30초) 이내에 프로빙 완료 가능

#### Agent 오버헤드
- CPU: **~1%**, 메모리: **~35MB** (컨테이너 전체 수명 동안 안정적)
- 학습 컨테이너에 미치는 영향 무시 가능

#### 탐지 정확도

| 지표 | 수치 |
|------|------|
| **Precision** | 98.2% |
| **Recall** | 99.3% |
| **장애 탐지 수** | 4,816건 (6개월) |
| **문제 컴포넌트 특정** | 1,302개 (정확도 95.7%) |
| **평균 탐지 시간** | 8초 |
| **수정 후 월간 장애율 감소** | **99.1%** (98% 컴포넌트 수정 후, 2024.10~12) |

---

### 7.2 네트워크 장애 분류 (Table 1)

#### 링크/스위치 이상 (Issue 1–4)

| No. | 이슈 | 컴포넌트 | 증상 | 원인 |
|-----|------|---------|------|------|
| 1 | CRC error | 호스트 간 네트워크 | Packet Loss | 물리 패브릭의 패킷 손상 |
| 2 | Switch port down | 호스트 간 네트워크 | 연결 불가 | 스위치 포트 도달 불가 |
| 3 | Switch port flapping | 호스트 간 네트워크 | Packet Loss | 스위치 포트 플래핑 |
| 4 | Switch offline | 호스트 간 네트워크 | 연결 불가 | 스위치 크래시 또는 업그레이드 오프라인 |

#### 호스트 관련 이상 (Issue 5–13)

| No. | 이슈 | 컴포넌트 | 증상 | 원인 |
|-----|------|---------|------|------|
| 5 | RNIC hardware failure | RNIC | 연결 불가 | RNIC 하드웨어 고장 |
| 6 | RNIC firmware not responding | RNIC | High Latency | RNIC 펌웨어 버그로 특정 플로우 고지연 |
| 7 | RNIC port down | RNIC | 연결 불가 | RNIC 포트 지속적 다운 |
| 8 | RNIC port flapping | RNIC | Packet Loss | RNIC 포트 주기적 다운 |
| 9 | Offloading failure | RNIC | High Latency | 패킷 캡슐화/역캡슐화를 RNIC에 오프로드 실패 |
| 10 | Bond error | Kernel/RNIC | 연결 불가 | RNIC 포트 본딩 실패 |
| 11 | RNIC GID change | Kernel | 연결 불가 | OS 네트워크 서비스 예기치 않은 재시작 |
| 12 | PCIe-NIC error | Host Board | High Latency | 같은 호스트 내 RNIC 간 통신 불가 |
| 13 | GPU direct RDMA error | Host Board | High Latency | GPU-RNIC 간 직접 통신 불가 |

#### 가상 스위치/컨테이너 이상 (Issue 14–19)

| No. | 이슈 | 컴포넌트 | 증상 | 원인 |
|-----|------|---------|------|------|
| 14 | Not using RDMA | Virtual Switch | High Latency | RDMA로 전송해야 할 플로우가 TCP/UDP 사용 |
| 15 | Repetitive flow offloading | Virtual Switch | High Latency | 오프로드된 플로우가 RNIC에서 빈번히 무효화 |
| 16 | Suboptimal flow offloading | Virtual Switch | High Latency | 잘못된 순서로 플로우 오프로드 → 일부 플로우 고지연 |
| 17 | Container crash | Container Runtime | 연결 불가 | 런타임 결함으로 컨테이너 생성 직후 크래시 |
| 18 | Hugepage misconfiguration | Configuration | High Latency | 호스트의 hugepage 설정과 RNIC 불일치 |
| 19 | Congestion control issue | Configuration | High Latency | 스위치의 특정 큐에 혼잡 제어 미활성화 |

#### 사례 연구: Flow Table 불일치 (Figure 18)
- 90초 전: 두 RNIC 컨테이너 간 지연 ~16µs로 안정
- 90초 후: 지연 120µs 급등 + 0.1% 미만 패킷 손실
- SkeletonHunter가 통계 테스트로 이상 판정 → overlay/underlay 모두 정상 → RNIC flow table 덤프
- **RNIC이 flow counter를 적시에 갱신하지 않아** control plane이 해당 flow를 비활성으로 판단하여 무효화
- 결과: 패킷이 소프트웨어 스택에서 처리되어 지연 급증
- RNIC 격리 후 60초 내 복구

---

## 한계

### 불확실한 사용자 워크로드
- 집합 통신 프리미티브를 사용하지 않는 학습(디버깅, 라이브러리 테스트 등)에서는 skeleton 추론이 부정확할 수 있음
- 새로운 병렬화 전략(예: Expert Parallelism)의 등장에 따른 미지의 트래픽 패턴
- 대응: skeleton 충실도 사전 검증, 사용자 인터페이스에서 수동 비활성화 옵션, 범용 추론 알고리즘 개발 중

### 오탐(False Detection)
- **인트라호스트 연결 이슈**(GPU-to-GPU, GPU-to-PCIe)는 네트워크가 아닌 하드웨어 문제 → SkeletonHunter 범위 밖
- Agent 크래시 시 프로빙 미응답 → 정상 링크를 장애로 오판
- PTP(Precision Time Protocol)를 사용한 클록 드리프트 제거에 의존 → agent 안정성이 전제

---

## 운영 경험

### Pingmesh 계열 기존 솔루션 시도
- 멀티 테넌트 컨테이너 학습의 동적 네트워크 토폴로지에 부적합
- IP-in-IP 등 특수 하드웨어 요구 → 운영 비용 및 안정성 리스크

### 장애 처리 통합
- 이상 탐지 시 네트워크 운영팀 알림 + 해당 호스트/RNIC **블랙리스트** 자동 추가
- 문제 해결 전까지 새 학습 작업 스케줄링 차단
- 학습 컨테이너 라이브 마이그레이션 메커니즘 개발 중

### 에이전트 빠른 업데이트
- 사이드카 컨테이너 활용 → 학습 작업과 독립적으로 배포/업데이트
- 월간 정기 릴리스 + 주간 긴급 릴리스
- 10개월간 **20회 이상** 온라인 업데이트 수행

### SR-IOV 환경의 도전
- VF 활성화 시 PF(Physical Function)에서 RDMA 사용 불가 → 호스트에서 직접 RDMA 모니터링 불가
- 사이드카 메커니즘으로 학습 컨테이너와 동일 네트워크 환경 공유하여 해결

---

## 핵심 기여 요약

1. **최초**: 대규모 컨테이너 학습의 네트워크 신뢰성 과제와 곱셈 효과를 실 운영 환경에서 규명
2. **Traffic Skeleton 기반 모니터링**: 학습 트래픽의 고유한 희소·주기적 패턴을 활용하여 프로빙 규모를 **full-mesh 대비 ~23× 축소**, 학습 라운드 내(8초) 탐지 달성
3. **Overlay-Underlay 분리 진단**: 낙관적 분리 가정 하에 19종 장애 유형을 6개 컴포넌트 범주로 정확히 위치 특정 (95.7%)
4. **프로덕션 검증**: 6개월간 4,816건 장애 탐지 → 1,302개 컴포넌트 수정 후 월간 장애율 **99.1% 감소**
