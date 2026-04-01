# BGPFlow: Flow-based Feature Extraction for BGP Anomaly Detection

## 기본 정보
| 항목 | 내용 |
|------|------|
| **제목** | BGPFlow: Flow-based Feature Extraction for BGP Anomaly Detection |
| **저자** | Yanxu Fu, Pei Zhang, Han Zhang, Xiaohong Huang, Yan Ma, Kun Xie, Dandan Li |
| **소속** | Beijing University of Posts and Telecommunications / Tsinghua University / Zhongguancun Laboratory |
| **학회** | NGNO '25 (1st Workshop on Next-Generation Network Observability), co-located with SIGCOMM 2025 (September 8–11, 2025, Coimbra, Portugal) |
| **DOI** | 10.1145/3748496.3748993 |
| **키워드** | BGP, BGP Anomaly, BGP Features, BGP Observability |

---

## 연구 배경 및 문제

### BGP 라우팅 이상의 심각성
- BGP는 인터-도메인 라우팅의 사실상 표준 프로토콜이지만, 분산적 특성과 내재적 보안 부재로 다양한 이상에 취약
- 실제 사례:
  - **CenturyLink (2020.8)**: BGP 설정 오류 → 글로벌 인터넷 트래픽 3.5% 감소
  - **KT Corporation (2021.10)**: BGP 백본 라우터 장애 → 한국 전국적 서비스 중단
  - **Facebook (2021.10)**: DNS 권한 서버 라우팅 프리픽스 철회 → 7시간 서비스 장애, 수십억 달러 경제 손실

### 기존 BGP 피처 추출의 한계

| 한계 | 설명 |
|------|------|
| **Collector/VP 수준 집계의 노이즈** | 대규모 이상 탐지에는 효과적이나, 소규모 이상의 신호를 희석시키는 데이터 노이즈 유발 |
| **그래프 피처의 계산 복잡도** | 글로벌 라우팅 가능 AS가 ~80,000개로 증가 → 그래프 기반 피처 추출의 계산 복잡도·지연 문제, 실시간 탐지 성능 저하 |

---

## BGPFlow 프레임워크

### 핵심 아이디어
- **Original AS를 키**로 BGP 피처 플로우를 조직하고, **국가 수준**에서 집계
- 정량적(volume) 피처와 그래프 기반(topological) 피처를 통합하는 다중 수준 집계 프레임워크
- **Graph Autoencoder(GAE) 기반 그래프 클러스터링**으로 국가 수준 토폴로지를 압축 → 구조적 의미 보존 + 계산 복잡도 대폭 감소

### 워크플로우 (4단계)

```
[1. BGP 데이터 수집] → RIB Dump + Update Dump (RouteViews, RIPE RIS)
         ↓
[2. 프리픽스 도달성 테이블 구축] → 실시간 갱신, 프리픽스→Original AS 매핑, 국가별 AS 그래프 구축
         ↓
[3. 피처 집계] → Original AS 수준 → 국가 수준 (volume + AS-PATH 피처)
         ↓
[4. 그래프 피처 집계] → GAE로 국가 토폴로지 압축 → K-Means 클러스터링 → 압축 그래프에서 그래프 피처 계산
```

---

### 단계 1: 프리픽스 도달성 테이블 구축 (§3.1)

- RIB Dump에서 초기 테이블 구축: 2차원 딕셔너리 $T_P[p][v]$ = VP $v$에서 프리픽스 $p$의 Original AS까지의 AS 경로
- 동시에 3가지 구조 유지:
  1. **프리픽스 도달성 테이블** ($T_P$): 실시간 경로 정보
  2. **프리픽스→Original AS 매핑 테이블** ($M_{P \to A}$): 각 프리픽스의 Origin AS
  3. **국가별 AS 그래프** ($G_C = (V_C, E_C)$): AS 경로에서 AS 쌍 추출, 동일 국가 소속 시 에지 추가
- Update 메시지로 실시간 갱신:
  - Withdrawal → $T_P[p][v]$ 삭제, $M_{P \to A}[p]$ 갱신
  - Announcement → $T_P[p][v]$ 갱신, $M_{P \to A}[p]$ 기록

### 단계 2: Original AS 수준 피처 추출 (§3.2)

- 프리픽스별 m차원 피처 벡터 $F_{prefix_i} = (f_1, f_2, ..., f_m)_t$ (volume 피처 + AS-PATH 피처)
- Original AS $a$의 피처 = 해당 AS에 매핑된 모든 프리픽스 피처의 집계(평균/합계/최대):

$$F_{origin_a} = \text{AGGREGATE}\{F_{prefix_p} \mid M_{P \to A}[p] = a\}$$

- **Original AS 관점의 장점**: 네트워크 운영자가 자기 네트워크의 라우팅 동태를 직접적 관점에서 모니터링 가능

#### MOAS(Multi-Origin AS) 탐지
- Original AS 하이재킹 시 피해 AS의 announcement/withdrawal 패턴이 변하지 않을 수 있음
- → 각 Original AS에 대해 **moas_num** 피처 유지: 시간 간격 내 새로운 MOAS 발생 횟수 기록
- 인메모리 프리픽스 도달성 테이블로 RIB에 이미 존재하는 안정적 MOAS 제외, 신규 MOAS만 탐지

### 단계 3: 국가 수준 피처 집계 (§3.3)

- 각 시간 간격 T마다, Original AS의 국가 소속 기반으로 집계:

$$F_{country_c} = \text{AGGREGATE}\{F_{origin_a} \mid M_{A \to C}[a] = c\}$$

- **국가 수준 집계의 장점**:
  1. 대규모 인터넷 토폴로지를 작은 국가 서브그래프로 분할 → 계산 복잡도 감소, 노이즈 격리
  2. 국가는 네트워크 생태계의 자연스러운 행정 단위 → 국가 당국이 대규모 이상 대응 조정에 적합
  3. 각 국가 서브그래프 내 이상 탐지 + 근본 원인 지역화 용이

#### AS 지리적 위치 정확도 보정
- 일부 AS가 등록 국가와 실제 운영 국가가 다를 수 있음 → 집계 노이즈 유발
- 3개 IP 지리 위치 DB (DB-IP, MaxMind GeoIP2, IP2Location) 교차 확인
- 각 AS의 발표 프리픽스 → /24 서브넷 분할 → 랜덤 3개 IP 쿼리 → 국가 레이블 합집합 계산
- 등록 국가와 지리 위치 매핑 국가 집합 간 교집합 없으면 해당 국가 AS 집합에서 제외
- 추가 검증: WHOIS 레코드 + 연관 조직 정보

### 단계 4: GAE 기반 그래프 압축 (§3.3, Algorithm 1)

미국처럼 18,000+ AS를 포함하는 국가의 계산 복잡도 해결:

**Graph Autoencoder 아키텍처:**
- **Encoder**: 2-layer GCN → 잠재 임베딩 $Z \in \mathbb{R}^{n \times d}$ ($d \ll n$)
- **Decoder**: Inner-product → 인접 행렬 재구축 $\hat{A} = \sigma(ZZ^\top)$
- 손실 함수: $\mathcal{L} = \|A - \hat{A}\|_F^2 + \lambda\|Z\|_2^2$

**클러스터링:**
- 학습된 임베딩 $Z$에 K-Means 적용 → $k$개 구조적 유사 그룹
- 각 클러스터 = 슈퍼노드 → 압축 그래프 $G' = (V', E')$ 구성
- 압축 그래프에서 그래프 피처(degree, centrality, clique, eigenvector, pagerank, clustering, triangles, assortativity 등) 효율적 계산

---

## 사례 연구

### 1. Facebook 장애 (2021.10.4) — 조직 수준
- Collector 수준 vs Original AS 수준(AS32934) 피처 집계 비교
- **Original AS 수준**: 노이즈 대폭 감소, 이상 탐지 **약 20분 조기 가능**
- 이유: 다수 AS에서 collector까지의 경로 탐색이 무관한 노이즈·지연 유발

### 2. KT Corporation 장애 (2021.10.25) — 국가 수준
- Collector 수준 vs 국가 수준(한국) 피처 집계 비교
- **국가 수준**: 데이터 노이즈 대폭 감소
- 프리픽스 도달성 테이블로 장애 전 국가 토폴로지 재구축 → KT Corporation 네트워크를 근본 원인으로 정확 지목

### 3. CenturyLink 장애 (2020.8.31) — 글로벌 수준
- 미국 AS 수준 토폴로지에서 클러스터링 전후 그래프 피처 비교
- 클러스터링 후에도 대부분 피처의 시간적 추세 일관 유지 (PCC 상당수 >0.9)
- 계산 시간: 원본 568.4초 → 클러스터링 후 38.8~60.5초 (1~2 자릿수 감소)

### 그래프 피처 압축 성능 (미국 토폴로지)

| 피처 | k=3/40 PCC | k=4/40 PCC | k=5/40 PCC | 원본 시간 | 압축 후 시간 |
|------|-----------|-----------|-----------|---------|----------|
| degree | 0.931 | **0.979** | 0.976 | 0.542s | 0.057s |
| pagerank | **1.000** | **1.000** | **1.000** | 8.961s | 0.859s |
| eigenvector | 0.140 | 0.845 | **0.904** | 148.26s | 6.865s |
| clustering | 0.021 | 0.753 | 0.832 | 185.07s | 13.36s |
| triangles | 0.783 | **0.958** | 0.892 | 185.24s | 10.43s |
| **전체** | - | - | - | **568.42s** | **60.52s** |

- 일부 피처(cliques, average neighbor degree)는 로컬 연결 패턴에 민감 → 작은 $k$에서 PCC 낮음

---

## 핵심 기여 요약

1. **Original AS 키 기반 피처 플로우**: collector/VP 관점 대신 Original AS를 키로 집계 → 노이즈 감소, 조기 탐지(~20분), 운영자 관점의 직접적 모니터링
2. **국가 수준 집계**: 글로벌 토폴로지를 행정적·토폴로지적으로 자연스러운 단위로 분할 → 계산 복잡도 감소, 근본 원인 지역화, 이상 격리
3. **GAE 기반 그래프 클러스터링**: GCN 인코더 + K-Means로 대규모 국가 토폴로지 압축 → 구조적 의미 보존(PCC >0.9)하면서 계산 시간 1~2 자릿수 감소

---

## 한계 및 향후 과제

- 사례 연구(case study)만 제시, 체계적 정량 평가(precision/recall 등) 미실시
- ML 모델과의 end-to-end 통합 미구현 (향후 계획)
- GAE 클러스터링 시 로컬 연결 패턴(cliques 등) 정보 손실 — 세밀한 클러스터링 개선 필요
- AS 지리적 위치 매핑의 부정확성이 여전히 국가 수준 집계에 노이즈 유발 가능
- LLM/에이전트 기반 BGP 이상 분석 탐구 계획 언급
