# 연구 분야 관심사

> 논문을 읽으며 발견한 흥미로운 주제, 기술, 아이디어를 정리하는 문서

---

## 1. RDMA 기반 제로 오버헤드 모니터링

**출처**: ZERO+ (TNET 2024) — Song et al., Shanghai Jiao Tong Univ. / Alibaba Group

### 관심 포인트
- **One-sided RDMA READ**를 활용하면 모니터링 대상 호스트의 CPU를 전혀 사용하지 않고 원격에서 메트릭을 수집할 수 있음
- 기존 모니터링(Netdata, Prometheus)은 1~5% CPU만 점유해도 서비스에 심각한 간섭 유발 → RDMA로 완전 해결
- **수신자 주도(Receiver-Driven) 모델**이 대규모 모니터링의 incast 문제 해결에 효과적
- 대역폭 공유 흐름 제어($BW_i = BW/Num$)로 네트워크 환경 변화에 자동 적응
- 알리바바 실제 프로덕션 환경 4개 이종 클러스터에서 검증됨

### 핵심 기술 키워드
`One-sided RDMA` `Zero-overhead Monitoring` `Receiver-driven Flow Control` `Cloud-native` `Incast Mitigation` `Priority Queue QoS` `Shared Memory` `eBPF`

### 떠오르는 생각 / 후속 질문
- RDMA가 없는 환경(일반 클라우드 VM)에서도 유사한 제로 오버헤드 접근이 가능할까?
- Smart NIC/DPU 기반으로 확장하면 더 범용적인 모니터링이 될 수 있을까?
- 보안 측면: RDMA READ로 메모리를 원격 노출하는 것의 보안 위험은?

### 관련 논문
- ZERO (USENIX ATC'22) — 본 논문의 이전 버전
- Pilaf (USENIX ATC'13) — One-sided RDMA를 KV store에 적용
- FaRM (USENIX NSDI'14) — RDMA 기반 분산 메모리
- Homa (SIGCOMM'18) — Receiver-driven 저지연 전송 프로토콜


## 2. MUD + eBPF 기반 IoT DDoS 방어

**출처**: Feraudo et al. (ICDCN 2024) — Univ. of Bologna / Cambridge / York

### 관심 포인트
- **MUD(RFC 8520) 표준을 rate-limiting으로 확장**하여 IoT 기기의 허용된 통신에도 트래픽 속도 제한을 걸 수 있음
- **XDP(eBPF) 기반 방화벽**으로 Linux 커널 최저 레이어에서 패킷 필터링 → 평균 556ns 처리, 기존 방화벽 대비 추가 오버헤드 없음
- 81개 IoT 기기 트래픽 분석으로 카테고리별 peak/average 임계값 도출 → Peak 정책 시 정상 트래픽 드롭 <1.5%
- 프로그래머블 스위치(P4) 없이도 **일반 Linux 라우터/Raspberry Pi에서 배포 가능**
- MUD는 제조업체가 기기별 통신 프로파일을 정의하는 표준 → IoT 보안의 확장 가능한 접근법

### 핵심 기술 키워드
`MUD (RFC 8520)` `eBPF` `XDP` `IoT Botnet` `DDoS Mitigation` `Rate-limiting` `iptables` `osMUD` `Smart Home Security`

### 떠오르는 생각 / 후속 질문
- ML 기반으로 기기별 rate-limit 임계값을 자동 학습/적응할 수 있을까?
- SwitchSketch 같은 sketch 기반 heavy hitter 탐지와 결합하면 저속 DDoS도 탐지 가능할까?
- P4 스위치에서의 MUD 시행과 eBPF 기반 시행의 규모 확장성 비교가 흥미로울 듯

### 관련 논문
- Mirai Botnet (USENIX Security'17) — IoT 봇넷의 대표 사례
- XDP (CoNEXT'18) — eXpress Data Path 원본 논문
- Cloudflare XDP DDoS Mitigation — 실제 프로덕션 eBPF/XDP 활용 사례

---

## 3. 트랜잭션 기반 비침투적 분산 트레이싱

**출처**: DeepTrace (SIGCOMM 2025) — Geng, Zhang et al., Tsinghua University

### 관심 포인트
- **트랜잭션 의미론(API 엔드포인트 관계 + 영속적 필드)을 활용**하여 비침투적 트레이싱의 정확도를 95% 이상으로 유지 — FIFO 기반(~40%)이나 지연 기반(~63%)과 완전히 다른 접근
- **eBPF 기반 20개 이상 프로토콜 파싱**: RFC/오픈소스에서 파생된 프로토콜 템플릿 + 점프 파싱으로 고효율 스팬 구축
- **TF-IDF + 피어슨 상관 + Shannon 엔트로피** 조합으로 확률 함수 가중치를 자동 조정 — 트랜잭션 필드 부재 시에도 graceful degradation
- **쿼리 기반 트레이스 조립**: 이중 인덱싱(역인덱스 + 히스토그램)으로 전송 오버헤드 94% 감소, 전체 스팬 수집 불필요
- 성능 오버헤드 단 **4.6%** (Jaeger 100%: 24%, DeepFlow: 유사 수준)
- 수십 개 기업 프로덕션 환경에서 실제 배포 → 15분 내 장애 근본 원인 식별, DDoS 공격 탐지

### 핵심 기술 키워드
`eBPF` `Non-intrusive Tracing` `Transaction-based Correlation` `TF-IDF` `Pearson Correlation` `Shannon Entropy` `Span Association` `Protocol Template` `Query-driven Assembly` `Microservice APM`

### 떠오르는 생각 / 후속 질문
- ZERO+의 RDMA 기반 메트릭 수집 + DeepTrace의 트랜잭션 기반 트레이싱을 결합하면 "제로 오버헤드 분산 트레이싱"이 가능할까?
- 암호화 트래픽(TLS)에서의 트랜잭션 필드 추출 — uprobe 활용 가능하지만 범용성 한계가 있을 듯
- 글로벌 추론(논문의 향후 계획)이 실현되면 50개 이상 컴포넌트 트레이스에서도 높은 정확도 가능할까?
- MUD+eBPF의 IoT DDoS 방어와 DeepTrace의 Application DDoS 분석을 결합하면 더 포괄적인 DDoS 대응 가능

### 관련 논문
- DeepFlow (SIGCOMM'23) — FIFO 기반 비침투적 트레이싱, DeepTrace의 선행 연구
- TraceWeaver (SIGCOMM'24) — 지연 기반 비침투적 트레이싱
- Jaeger — 침투적 분산 트레이싱의 표준
- OpenTelemetry — 관측 가능성 표준화 프레임워크
- Dapper (Google, 2010) — 분산 트레이싱의 기초

---

## 4. PFC 프로비넌스 기반 RDMA 네트워크 이상 진단

**출처**: Hawkeye (SIGCOMM 2025) — Shicheng Wang, Menghao Zhang et al., Tsinghua University / Beihang University

### 관심 포인트
- **PFC(Priority Flow Control)의 연쇄적 혼잡 확산**이 RDMA 네트워크에서 기존 진단 도구가 대응 못하는 복잡한 성능 이상(backpressure, storm, deadlock)을 유발 — 근본 원인이 피해 플로우 경로에서 수 홉 떨어진 곳에 위치할 수 있음
- **포트 쌍별 트래픽 미터**로 PFC 인과관계를 세밀하게 기록: 같은 스위치에서 혼잡한 포트가 여러 개여도 실제 패킷 축적에 기여하는 인과 포트만 식별 가능
- **데이터 플레인 내 라인레이트 PFC 인과관계 추적**: polling 패킷이 피해 플로우 경로를 따라가다 PFC를 감지하면 자동으로 PFC 확산 경로까지 재귀적 추적 — 스위치 CPU는 비동기 텔레메트리 수집만 담당
- **이종 wait-for 프로비넌스 그래프**: 포트↔플로우 간 관계를 3종 에지(포트→포트, 플로우→포트, 포트→플로우)로 표현 → 서명 매칭으로 이상 유형과 근본 원인을 동시에 진단
- 90% 이상 정밀도, ~100% 재현율이면서 기존 대비 **1~4 자릿수 낮은 오버헤드** (CPU 필터링으로 텔레메트리 80%+ 감소, 패킷 수 95% 감소)
- Tofino 하드웨어 리소스 30% 이하 사용, 차세대 ASIC에도 확장 가능

### 핵심 기술 키워드
`RDMA` `PFC (Priority Flow Control)` `Network Provenance` `Programmable Switches (P4/Tofino)` `In-network Causality Analysis` `Wait-for Graph` `Signature-based Diagnosis` `Epoch-based Telemetry` `PFC Deadlock` `PFC Storm`

### 떠오르는 생각 / 후속 질문
- ZERO+의 RDMA 모니터링은 호스트 메트릭 수집에 집중하고, Hawkeye는 네트워크 내부 PFC 진단에 집중 → 두 시스템을 결합하면 호스트~네트워크 전체를 아우르는 RDMA 관측 가능성(observability) 스택이 될 수 있을까?
- PFC 프로비넌스 그래프의 서명 매칭 방식은 사전 정의된 이상에만 대응 — ML 기반으로 미지의 이상 패턴을 자동 발견하는 확장이 가능할까?
- 멀티 테넌트 환경에서 PFC 기반 공격(LoRDMA 등)의 탐지와 진단을 Hawkeye가 동시에 수행할 수 있을까? 보안 활용 가능성이 흥미로움
- 부분 배포(ToR 스위치만 flow telemetry) 시 진단 효과 저하 — DeepTrace의 트랜잭션 기반 상관관계처럼 불완전한 데이터에서도 추론하는 방법을 적용할 수 있을까?
- 에포크 크기와 감지 임계값의 자동 튜닝 — 워크로드 패턴에 따라 적응적으로 조정하는 메커니즘이 필요해 보임

### 관련 논문
- ZERO+ (TNET'24) — RDMA 기반 제로 오버헤드 모니터링 (관심사 #1)
- SpiderMon (NSDI'22) — 피해 플로우 경로 기반 프로비넌스 진단 (Hawkeye의 주요 비교 대상)
- ITSY (INFOCOM'22) — 데이터 플레인 내 PFC 데드락 탐지 (루프만 대응)
- LoRDMA (NDSS'24) — PFC backpressure를 악용한 저속 DoS 공격 (같은 저자)
- R-Pingmesh (SIGCOMM'24) — RoCE 네트워크 모니터링 진단 시스템
- PrintQueue (SIGCOMM'22) — 패킷 수준 큐 측정 프로비넌스

---

## 5. SDN 서비스 우선순위 기반 Link Flooding Attack 방어

**출처**: SPM (ISCCN 2025) — Jiancheng Wang et al., Information Engineering University, China

### 관심 포인트
- **Link Flooding Attack(LFA)**는 서비스가 아닌 **링크를 대상**으로 하는 DDoS 공격 — 단일 공격 호스트의 트래픽이 정상과 거의 동일하여 기존 DDoS 탐지 무력화
- **서비스 우선순위 기반 핵심 링크 선택**: 네트워크 전체가 아닌 고가치 서비스에 도달하는 경로에서 빈도 높은 링크만 모니터링 → 방어 비용 대폭 감소
- **SDN flow table에서 LFA 고유 행동 피처 추출**: 고빈도 목적지 IP 집중도, 혼잡 기간 출현 패턴 등 11개 — 전통 DDoS 피처(대역폭, 패킷 손실률 등)와 완전히 다른 접근
- **빠른 완화 + 느린 식별 전략**: 의심 호스트에 지수적으로 증가하는 폐기 시간 적용(2→4→8→...초), 이중 임계값(신뢰 0.95 / 식별 5회)으로 정상 호스트 영향 최소화
- Random Forest 기반으로 재현율 97.52%, 오탐률 1.36% 달성, 공격 주기·악성 호스트 수 변화에도 안정적

### 핵심 기술 키워드
`Link Flooding Attack (LFA)` `SDN` `Crossfire Attack` `Service Priority` `Flow Table Feature Extraction` `Random Forest` `LLDP Link Delay` `Quick Mitigation Slow Detection` `Rolling Attack` `Decoy Service`

### 떠오르는 생각 / 후속 질문
- MUD+eBPF의 IoT DDoS 방어(관심사 #2)와 결합: IoT 봇넷이 LFA 공격을 수행할 때, MUD 프로파일로 디코이 서비스 접근 패턴을 사전 차단하는 것이 가능할까?
- SPM의 "고빈도 목적지 IP 집중도" 피처는 SwitchSketch(관심사 #1 관련) 같은 sketch 기반 heavy hitter 탐지와 결합하면 데이터 플레인에서 실시간 추출 가능할 듯
- Mininet 20Mbps 실험은 규모가 작음 — 실제 데이터센터급(100Gbps+)에서 LLDP 기반 5초 주기 지연 측정이 충분히 빠른가? Hawkeye(관심사 #4)의 에포크 기반 텔레메트리와 결합하면 더 민감한 탐지 가능
- 적응형 공격자가 60초 윈도우 내에 행동 패턴을 변형하면? — 온라인 학습이나 윈도우 크기 적응적 조정이 필요해 보임
- SDN 부분 배포 시 flow table 수집 불가 — P4 프로그래머블 스위치에서 피처 추출을 데이터 플레인에 오프로드하는 확장이 흥미로움

### 관련 논문
- Crossfire Attack (IEEE S&P'13) — LFA의 대표적 공격 모델
- CyberPulse++ (Int J Intell Syst'21) — 전통 DDoS 피처 기반 ML 탐지 (SPM의 비교 대상)
- CFADefense (HPCC'19) — 재라우팅 기반 Crossfire 방어
- EqualNet (NDSS'22) — 토폴로지 난독화 기반 LFA 능동 방어
- SpiderNet (TNET'24) — 허니팟 기반 봇넷 식별 + 토폴로지 은닉

---

## 6. BGP 이상 탐지를 위한 다중 수준 피처 추출

**출처**: BGPFlow (NGNO '25 @ SIGCOMM 2025) — Yanxu Fu, Pei Zhang, Han Zhang et al., BUPT / Tsinghua University

### 관심 포인트
- **Original AS를 키로 피처를 집계**하면 collector/VP 수준 대비 노이즈가 대폭 줄고, Facebook 장애 사례에서 **약 20분 조기 탐지** 가능 — 기존 collector 관점의 한계를 극복하는 간단하면서 효과적인 아이디어
- **국가 수준 집계**: 글로벌 80,000+ AS 토폴로지를 행정적·토폴로지적으로 자연스러운 국가 단위로 분할 → 계산 복잡도 감소 + 근본 원인 지역화 (KT 장애에서 한국 토폴로지 재구축으로 정확 지목)
- **GAE(Graph Autoencoder) + K-Means로 그래프 압축**: 미국 18,000+ AS 토폴로지를 구조적 의미 보존(PCC >0.9)하면서 계산 시간 568초→61초로 대폭 감소 — 실시간 BGP 모니터링에 필수적
- **MOAS(Multi-Origin AS) 탐지**: 인메모리 프리픽스 도달성 테이블로 신규 MOAS만 식별 → 프리픽스 하이재킹 의심 이벤트 조기 발견
- 아직 ML 모델 통합 전 단계(피처 추출 프레임워크)이지만, 다중 세분화 관측 가능성(observability) 뷰를 제공하는 데이터 플랫폼으로서 가치가 높음

### 핵심 기술 키워드
`BGP` `Routing Anomaly` `Original AS` `Country-level Aggregation` `Graph Autoencoder (GAE)` `GCN` `K-Means Clustering` `MOAS Detection` `Prefix Hijacking` `Network Observability`

### 떠오르는 생각 / 후속 질문
- DeepTrace(관심사 #3)의 트랜잭션 기반 상관관계처럼, BGPFlow의 Original AS 키도 "의미 있는 집계 단위를 찾는 것"이 성능의 핵심 — 다른 네트워크 모니터링에서도 적용 가능한 범용 원칙일까?
- SPM(관심사 #5)의 Link Flooding Attack 방어 시 핵심 링크 선택에 BGPFlow의 국가 수준 토폴로지 정보를 활용하면 더 정교한 링크 중요도 계산이 가능할 듯
- GAE 기반 그래프 압축이 Hawkeye(관심사 #4)의 프로비넌스 그래프에도 적용 가능할까? 대규모 데이터센터 토폴로지의 PFC 인과관계 그래프 압축에 활용 가능성
- AS 지리적 위치 부정확성 문제 — 실시간 액티브 측정(traceroute 등)과 결합하면 보정 가능할까?
- 향후 LLM/에이전트 기반 BGP 이상 분석 계획이 흥미로움 — 프로비넌스 그래프 + LLM 추론의 결합은 네트워크 진단의 새로운 패러다임이 될 수 있을 듯

### 관련 논문
- BML (ICC Workshops'21) — BGP 데이터셋 생성 및 기존 피처 추출 도구
- Themis (USENIX Security'22) — MOAS 기반 프리픽스 하이재킹 탐지 가속
- NSDI'24 (Holterbach et al.) — forged-origin BGP 하이재킹 탐지 시스템
- Xu et al. (JCN'10) — Origin AS 관점 BGP 라우팅 분석의 선행 연구
- Variational GAE (Kipf & Welling, 2016) — BGPFlow 그래프 압축 기반 기술

---

## 7. Alert Flooding 분석을 통한 대규모 네트워크 장애 대응

**출처**: SkyNet (SIGCOMM 2025) — Bo Yang, Huanwu Hu, Yifan Li et al., Alibaba Cloud

### 관심 포인트
- **심각한 네트워크 장애 시 $O(10^4)$개 알림 폭주(Alert Flooding)**를 소수의 인시던트로 압축하는 실용적 시스템 — 단일 모니터링 도구의 커버리지 한계(3%~84%)를 12종 데이터 소스 통합으로 해결
- **계층적 Alert Tree**: 네트워크 계층(Region → City → Logic Site → Site → Cluster)에 따라 알림을 시간·위치·타입 기반 클러스터링 → 토폴로지 인접성을 활용하여 같은 근본 원인의 알림을 자동 그룹핑
- **알림 3단계 분류(Failure / Abnormal / Root Cause)**: Failure 알림이 인시던트 탐지에 가장 중요(거의 모든 장애에 동반)하고, Root Cause 알림이 완화에 핵심 — 유형 기반 카운팅으로 알림 빈도 차이 해소 및 오경보 억제
- **정량적 심각도 평가**: Impact Factor(회선 단절·고객 중요도) × Time Factor(패킷 손실률·지속 시간)로 동시 다발 인시던트 우선순위 결정 — 시그모이드 함수로 소수 핵심 고객 영향 시 민감하게 반응
- **시계열 인과 분석을 의도적으로 배제**: 실제로는 동작 이상 알림이 먼저, 근본 원인 알림이 나중에 도착 → 타임아웃 윈도우 기반 연관이 더 신뢰할 수 있다는 프로덕션 교훈이 인상적
- 1.5년간 프로덕션 운영, **false negative 0건**, 완화 시간 중간값 80% 감소(736초→147초)

### 핵심 기술 키워드
`Alert Flooding` `Multi-source Monitoring Integration` `Hierarchical Alert Tree` `Incident Discovery` `Severity Scoring` `Alert Classification` `Location Zoom-in` `Reachability Matrix` `SOP Automation` `Cloud Network Reliability`

### 떠오르는 생각 / 후속 질문
- ZERO+(관심사 #1)의 RDMA 기반 메트릭 수집을 SkyNet의 데이터 소스로 추가하면, 호스트 측 메트릭까지 제로 오버헤드로 통합하여 더 포괄적인 장애 탐지가 가능할까?
- Hawkeye(관심사 #4)의 PFC 프로비넌스 그래프와 SkyNet의 Alert Tree를 결합하면: SkyNet이 인시던트의 위치·범위를 파악하고, Hawkeye가 해당 범위 내에서 정밀 근본 원인을 추적하는 계층적 진단이 가능할 듯
- DeepTrace(관심사 #3)의 트랜잭션 기반 상관관계 아이디어를 SkyNet에 적용: 알림 간 "트랜잭션 의미론"(같은 장애에서 파생된 알림들의 의미적 관계)을 학습하면 타임아웃 윈도우보다 더 정확한 알림 그룹핑이 가능할까?
- SkyNet이 LLM 통합을 계획 중인 것이 흥미로움 — 인시던트로 압축된 결과를 LLM 입력으로 제공하면 입력 길이 제한 문제를 자연스럽게 해결하면서 진단 자동화 가능
- BGPFlow(관심사 #6)의 "의미 있는 집계 단위를 찾는 것"과 SkyNet의 "네트워크 계층 기반 위치 집계"는 같은 원칙 — 관측 가능성에서 집계 키 선택이 시스템 효과의 핵심
- 경험치 기반 임계값(인시던트 생성 2/1+2/5, 심각도 10) → 워크로드/네트워크 규모에 따라 자동 적응하려면 어떤 접근이 필요할까?

### 관련 논문
- Scouts (SIGCOMM'20) — 도메인 맞춤 인시던트 라우팅
- Gandalf (NSDI'20) — 대규모 클라우드 안전 배포 분석
- Terminator (INFOCOM WKSHPS'20) — 경량 장애 위치 추적
- NetPilot (SIGCOMM'12) — 데이터센터 장애 자동 완화
- DeepIP (ASE'20) — 딥러닝 기반 인시던트 심각도 우선순위 예측
- ZERO+ (TNET'24) — RDMA 기반 제로 오버헤드 모니터링 (관심사 #1)
- Hawkeye (SIGCOMM'25) — PFC 프로비넌스 기반 RDMA 진단 (관심사 #4)

---

## 8. Two-level 협업 측정으로 스케치 정확도와 부하 균형 동시 달성

**출처**: LTCM (TOIT 2025) — Zhongjun Qiu, Yang Du, He Huang et al., Soochow University

### 관심 포인트
- **Flow-level 로드 밸런싱 = 정확도 최적화, Packet-level 로드 밸런싱 = 오버헤드 최적화**라는 핵심 관찰이 인상적 — 기존 협업 측정의 양자택일 구도를 heavy-tail 분포 특성을 활용하여 양립시킴
- **Interval-matching 기법**: 토폴로지 대칭성을 활용하여 스위치당 유지할 구간을 $\sigma_j$개(k=48 FatTree에서 65만 개)에서 최대 $2h$개(~96 바이트)로 축소 → $\mathcal{O}(h^4)$ 복잡도로 네트워크 규모와 무관한 확장성. 문제를 선형 방정식으로 변환하는 과정이 우아함
- **BSS(Balancing Strategy Selector)**: 5KB 메모리 + 샘플링 확률 0.001로 대규모 플로우를 실시간 탐지하여 전략 전환 — 1비트 플래그로 하류 스위치에 전략 전달하는 설계가 매우 실용적
- **TC Sketch의 대/소 플로우 분리 저장**: LTCM이 자연스럽게 제공하는 플로우 크기 정보를 활용하여 해시 충돌을 구조적으로 감소 — SwitchSketch의 모드 전환과 상호 보완적인 아이디어
- **Tofino P4 구현**: Read-Write Phase 분리 + resubmit 패턴, 리소스 사용 대부분 10% 미만. Hawkeye(관심사 #4)의 Tofino 구현 패턴과 비교할 만함
- AAE 최대 96.6% 감소, packet-level 표준편차 최대 80.3% 감소

### 핵심 기술 키워드
`Collaborative Measurement` `Two-level Load Balancing` `Sketch` `Interval-matching` `Heavy-tail Distribution` `Clos Topology` `P4/Tofino` `Resubmit` `Flow-level` `Packet-level` `Per-flow Size Estimation` `Heavy Hitter Detection`

### 떠오르는 생각 / 후속 질문
- SwitchSketch(같은 저자 그룹)의 가변 길이 셀 + 모드 전환이 TC Sketch의 $B_S$/$B_L$ 내부에 적용되면 더 정밀한 대/소 플로우 분리와 메모리 활용이 가능할 듯 — 두 기법의 결합이 자연스러움
- Hawkeye(관심사 #4)의 PFC 프로비넌스 그래프 구축 시, LTCM의 interval-matching 기법을 활용하면 텔레메트리 수집 부하를 스위치 간 균등 분배할 수 있을까? 측정 부하 균형 원리의 다른 도메인 적용
- SPM(관심사 #5)의 Link Flooding Attack 탐지에서 핵심 링크 트래픽 모니터링 → LTCM의 협업 스케치를 배포하면 각 스위치의 heavy hitter 탐지 정확도를 높이면서도 오버헤드 균형 가능
- BSS의 임계값 $\mathcal{T}$와 TC Sketch의 $B_L:B_S$ 비율이 모두 고정 — 트래픽 패턴 변화에 적응적으로 조정하는 메커니즘이 있다면 DCN의 동적 워크로드(burst, 마이크로서비스 스케일링 등)에 더 강건할 것
- ECMP 멀티패스 환경에서 쿼리 시 모든 경로 순회 필요 — DeepTrace(관심사 #3)의 쿼리 기반 트레이스 조립(이중 인덱싱)과 유사한 효율적 쿼리 메커니즘을 적용할 수 있을까?
- eBPF/XDP(관심사 #2) 환경에서 LTCM의 interval-matching + BSS를 소프트웨어로 구현하면 P4 스위치 없이도 협업 측정이 가능할까? 범용 Linux 서버에서의 배포 가능성

### 관련 논문
- SwitchSketch (SIGMOD'23) — 같은 저자 그룹의 단일 스위치 heavy hitter 탐지 스케치 (TC Sketch와 상호 보완)
- NSPA (INFOCOM'19) — Flow-level 협업 측정, LP 기반 $\mathcal{O}(n^{3.5})$ (LTCM의 주요 비교 대상)
- CountMax (TNET'18) — Ingress/Egress 한정 flow-level 측정
- DS (TNET'23) — 분산 스케치, 경로별 논리 스케치 배분
- UWRA (TNET'21) — Packet-level 협업 측정, min-heap 기반 글로벌 샘플링
- cSamp (NSDI'08) — LP 기반 network-wide flow 모니터링
- Hawkeye (SIGCOMM'25) — Tofino P4 구현 패턴 비교 (관심사 #4)

---

## 8. 에이전트리스 실시간 경로 인식 네트워크 프로빙

**출처**: ByteTracker (SIGCOMM 2025) — Shixian Guo, Kefei Liu et al., ByteDance / BUPT

### 관심 포인트
- **에이전트리스 프로빙**: 수백만 서버 DC에서 엔드호스트에 에이전트·설정 변경 없이 소수의 중앙집중형 Prober만으로 전체 네트워크 프로빙 — TCP SYN(무효 포트)을 보내면 타겟 **커널이 RST로 자동 응답**하므로 user-space 프로세스 불필요. 배포·업데이트·버그 리스크가 근본적으로 제거됨
- **동시 프로빙으로 호스트/네트워크 타임아웃 구분**: 타겟당 3개 프로브를 서로 다른 ECMP 경로로 동시 발사 → 1~2개 타임아웃이면 네트워크, 3개 모두면 호스트로 판정. 추가 메트릭/이벤트/다른 NIC 없이 **단일 타겟 프로브 데이터만으로 밀리초 수준 판별** — R-Pingmesh의 ToR-mesh(100ms 주기, 다수 NIC 의존)보다 훨씬 간결하고 정확
- **ERSPAN 실시간 경로 추적**: 패킷 미러링이 포워딩 **전에** 발생하므로 드롭된 패킷도 미러링됨 → 마지막 미러링 위치 = 장애 스위치. 5개 이상 독립 프로브가 같은 스위치를 마킹해야 이상으로 판정 → 오탐 확률 $\ll (0.01)^5 \approx 0$
- **N-S + E-W 2방향 프로빙의 배경이 흥미로움**: 멀티칩 스위치의 칩 내부/칩 간 포워딩 차이를 실제 운영 경험에서 발견하여 설계에 반영 — Pod당 ToR 1대만 IP-in-IP 활성화하여 수평 방향 커버. 512B 페이로드로 모든 스위치의 패킷 분할·재조립 이상까지 탐지
- **홉별 MD5 비교로 silent bit flipping 탐지**: 기존 프로빙 시스템이 전혀 대응하지 못하던 영역을 해결. CRC+TCP checksum을 통과하는 극단 사례가 실제로 서비스 크래시를 유발했다는 것이 놀라움
- **실적**: 수백만 서버 전체 DC에 1년 내 배포, 6개월간 276건 이상 탐지(Pingmesh 262건), **5초 이내 100% 장애 위치 정확도**

### 핵심 기술 키워드
`Agentless Probing` `ERSPAN` `Packet Mirroring` `TCP SYN/RST` `Concurrent Probing` `North-South / East-West` `Multi-chip Switch` `IP-in-IP` `Bit Flipping Detection` `Fault Localization`

### 떠오르는 생각 / 후속 질문
- BiAn(관심사 #7)의 LLM 기반 장애 위치 파악과 ByteTracker의 ERSPAN 기반 실시간 위치 파악은 **완벽한 보완 관계** — ByteTracker는 5초 이내에 정확한 장애 스위치를 식별하고, BiAn은 복잡한 인시던트에서 다중 데이터 소스를 LLM으로 추론. ByteTracker의 위치 결과를 BiAn의 입력(모니터링 도구의 하나)으로 활용하면 시너지가 클 듯
- Hawkeye(관심사 #4)의 PFC 프로비넌스와 ByteTracker의 ERSPAN 경로 추적을 결합하면 — ByteTracker가 패킷 드롭 위치를 식별하고, Hawkeye가 PFC 연쇄 확산의 인과관계를 분석하는 **2계층 진단**이 가능
- **RoCE 적응**이 향후 과제로 언급됨 — ByteTracker의 TCP 프로브는 PFC 데드락 등 RoCE 큐 고유 문제를 탐지 못함. Hawkeye의 polling 패킷과 유사한 RoCE 큐 전용 프로브가 필요
- ZERO+(관심사 #1)의 RDMA 기반 제로 오버헤드 모니터링과 ByteTracker의 에이전트리스 프로빙은 모두 **호스트 부담 제로**를 추구하되 접근이 다름(RDMA READ vs 커널 자동 응답) — 두 시스템을 결합하면 메트릭 수집 + 네트워크 장애 탐지를 모두 호스트 개입 없이 달성 가능
- 동시 프로빙의 "3개 병렬 경로"가 DLB/AR(패킷/flowlet 수준 부하 분산) 환경에서도 유효한지? — 모든 프로브가 동일 경로로 해시될 가능성은 낮지만 검증이 필요
- 단방향 패킷 드롭이 대부분이라는 발견이 인상적 → 기존 양방향 가정의 진단 시스템들이 재검토 필요

### 관련 논문
- Pingmesh (SIGCOMM'15) — ByteTracker의 주요 비교·대체 대상
- R-Pingmesh (SIGCOMM'24) — RoCE 네트워크 확장판, ToR-mesh 프로빙 (관심사 #4 관련)
- NetBouncer (NSDI'19) — IP-in-IP 기반 능동 장애 위치 파악 (에이전트 필요)
- Everflow (SIGCOMM'15) — 패킷 매칭·미러링 기반 네트워크 분석
- RD-Probe (SIGCOMM'24) — 스위치 미러링으로 프로브 커버리지 분석 + 결정론적 프로브 보완
- 007 (NSDI'18) — 패킷 드롭 원인의 "민주적" 탐지 (BiAn의 Hot Device 베이스라인)

---

<!-- 새 관심사 항목은 아래에 추가 -->
