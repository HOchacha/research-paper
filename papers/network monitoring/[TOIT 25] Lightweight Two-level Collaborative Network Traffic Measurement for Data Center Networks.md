# Lightweight Two-level Collaborative Network Traffic Measurement for Data Center Networks

## 기본 정보
- **저자**: Zhongjun Qiu, Yang Du, He Huang, Yu-E Sun, Guoju Gao
- **소속**: Soochow University (School of Computer Science and Technology / School of Rail Transportation)
- **게재**: ACM Transactions on Internet Technology, Vol. 25, No. 4, Article 23, September 2025
- **DOI**: 10.1145/3757320
- **소스코드**: https://github.com/duyang92/ltcm-paper
- **키워드**: Collaborative network traffic measurement, measurement load balancing, sketch, data center network

---

## 1. 연구 배경 및 문제 정의

### 네트워크 트래픽 측정의 필요성
- 데이터센터 네트워크(DCN)에서 로드 밸런싱, 혼잡 제어, 트래픽 엔지니어링 등 핵심 기능을 위해 트래픽 측정이 필수적
- 주요 측정 태스크: **per-flow size estimation**(플로우별 크기 추정)과 **heavy hitter detection**(대규모 플로우 탐지)

### 핵심 도전과제
- 스위치의 온칩 메모리가 매우 제한적(< 10MB)이므로 대량의 플로우를 개별 저장하는 것은 비실용적
- 기존 스케치 기반 솔루션은 단일 스위치에서 최적화 → 패킷이 여러 스위치를 통과할 때 **중복 측정** 발생

### 협업 측정(Collaborative Measurement)의 두 가지 접근법

| 접근법 | 원리 | 장점 | 한계 |
|--------|------|------|------|
| **Flow-level** (NSPA, cSamp, CountMax, DS 등) | 플로우 단위로 스위치에 측정 배분, 각 스위치가 담당 플로우의 모든 패킷 측정 | 스위치당 측정 플로우 수 감소 → 해시 충돌 줄어 **정확도 향상** | 대규모 플로우가 단일 스위치에 할당되면 **측정 오버헤드 불균형** 심각; 전략 계산에 $\mathcal{O}(n^{3.5})$ 소요 → **확장성 부족** |
| **Packet-level** (UWRA 등) | 패킷 단위로 경로 상의 스위치 중 하나를 랜덤 선택하여 측정 | 스위치당 측정 패킷 수 균등 → **오버헤드 균형** | 모든 플로우의 패킷이 모든 스위치에 분산 → 스위치당 측정 플로우 수 미감소 → **정확도 개선 없음** |

### 핵심 관찰
> **Flow-level 로드 밸런싱은 정확도 최적화**, **Packet-level 로드 밸런싱은 오버헤드 최적화**를 각각 목표로 한다.
> 두 수준의 장점을 동시에 결합할 수 있는가?

---

## 2. LTCM 핵심 아이디어

### Two-level 전략의 직관

DCN 트래픽은 **heavy-tail 분포** 특성을 보임: 소수의 대규모 플로우가 트래픽 대부분을 차지하고, 대다수는 소규모 플로우.

1. **초기**: 모든 플로우에 flow-level 전략 적용 → 각 스위치가 측정하는 플로우 수 균형
2. **임계값 $\mathcal{T}$ 초과 시**: 해당 플로우의 후속 패킷에 packet-level 전략 적용 → 대규모 플로우의 패킷을 경로 상 모든 스위치에 균등 배분

### 예시 (Figure 2)

4개 스위치, 4개 플로우($f_1$: 16, $f_2$: 10000, $f_3$: 100, $f_4$: 4 패킷), $\mathcal{T} = 100$:

| 솔루션 | 스위치당 최대 플로우 | 스위치당 최대 패킷 |
|--------|-------------------|-----------------|
| Flow-level | 1 | **10,000** (심한 불균형) |
| Packet-level | **4** (감소 없음) | 2,530 |
| **Two-level (LTCM)** | **2** (약간 증가) | **2,575** (균형) |

- $f_2$의 처음 100 패킷은 flow-level로 $S_2$가 측정, 101번째부터 packet-level로 4개 스위치에 균등 배분
- 플로우 수 증가는 최대 1개로 미미 → 정확도 유지

### ECMP 호환성
- LTCM은 트래픽 라우팅이 아닌 **측정 부하 분산**을 다룸
- 멀티패스 라우팅 환경에서도 각 경로에서 독립적으로 동작 가능, 쿼리 시 모든 관련 경로의 결과를 합산

---

## 3. FCM: Lightweight Flow-level Collaborative Measurement

### Interval-Matching 기법

각 스위치에 구간(interval)을 배정하여, 패킷의 플로우 해시값 $H(f) \in [0, 1]$이 해당 구간에 속할 때만 측정:

- 경로 $Q_i$ 위의 스위치 $S_{Q_{i,j}}$에 구간 $[x_{i,j}, y_{i,j}]$ 배정
- 구간들은 **비중첩(non-overlapping)** 이며 합집합이 $[0, 1]$을 커버
- 같은 플로우의 모든 패킷은 동일한 $H(f)$ → 동일 스위치에서 측정

### 플로우 수 균형을 위한 선형 방정식

스위치의 flow-level 측정 부하를 구간 길이 $L_{i,j} = y_{i,j} - x_{i,j}$로 표현:

$$L_{i,1} = L_{i,2} = \cdots = L_{i,t_i}$$

### h-tier Clos 토폴로지로 확장

**메모리 최적화**: 같은 길이의 경로를 공유하는 구간으로 통합

- 원래: 각 스위치가 betweenness centrality $\sigma_j$개의 구간 유지 필요 (k=48 FatTree에서 65만 개 → ~5MB)
- **최적화 후**: 각 스위치가 최대 $2h$개 구간만 유지 (h=2~3이므로 약 **96 바이트**)

### 최종 연립방정식 (Equation 4)

$$\begin{cases}
\sigma_{1,1} \times L_{1,h} = C \\
\sigma_{2,1} \times L_{1,h-1} + \sigma_{2,2} \times L_{2,h-1} = C \\
\vdots \\
\sigma_{h,1} \times L_{1,1} + \sigma_{h,2} \times L_{2,1} + \cdots + \sigma_{h,h} \times L_{h,1} = C \\
2 \times (L_{1,1} + \cdots + L_{1,h-1}) + L_{1,h} = 1 \\
\vdots \\
L_{h,1} = 1
\end{cases}$$

- **시간 복잡도**: $\mathcal{O}(h^4)$ (h=2~3이므로 사실상 상수 시간)
- **공간 복잡도**: 스위치당 최대 $2h$개 구간 (~96 바이트)
- **확장성**: 네트워크 규모(스위치 수)와 무관, 토폴로지 계층 수에만 의존

---

## 4. BSS: Measurement Load Balancing Strategy Selector

### 역할
- Ingress 스위치에서 플로우의 네트워크 진입 패킷 수가 임계값 $\mathcal{T}$를 초과하는지 실시간 판별
- 초과 시 패킷 헤더에 **1비트 플래그**를 1로 설정 → 하류 스위치에 packet-level 전략 사용 알림

### 구조
- **$w$개 버킷**, 각 버킷에 fingerprint($fp$) + counter($cnt$) 저장
- 패킷 $(f, e)$ 도착 시:
  1. 확률 $p_r$(예: 0.001)로 샘플링하여 삽입 시도
  2. 버킷 $B[j]$에서 $j = h(f) \% w$
  3. 충돌 시 $cnt$ 감소, $cnt = 0$이면 현재 플로우로 교체

### 메모리 효율
- $\mathcal{T} = 10000$, $p_r = 0.001$일 때 카운터는 **4비트**로 충분 ($15/0.001 = 15000$ 패킷까지 추적 가능)
- 전체 BSS 메모리: **5KB**

### Packet-level 구간 설정
- 경로 $Q_i$의 $j$번째 스위치: 구간 $[(j-1)/t_i,\ j/t_i]$
- 패킷 해시 $H(e)$(플로우 해시가 아닌 **패킷 고유 해시**) 사용 → 패킷이 스위치에 균등 분배

---

## 5. TC Sketch: Two-layer Collaborative Sketch

### 설계 동기
- LTCM이 대규모/소규모 플로우를 자연스럽게 구분 → 이 정보를 활용하여 **해시 충돌 감소**

### 구조

```
┌──────────────────────────────────────┐
│  B_L (Large Buckets) — Packet-level  │
│  B_{l,1}  B_{l,2}  ...  B_{l,w_l}   │
├──────────────────────────────────────┤
│  B_S (Small Buckets) — Flow-level    │
│  B_{s,1}  B_{s,2}  ...  B_{s,w_s}   │
└──────────────────────────────────────┘

각 Bucket: d개 Cell = [fingerprint | counter]
```

- **$B_S$**: 소규모 플로우 저장 (flow-level 전략, $w_s$개 버킷, $d_s$개 셀, 16비트 카운터)
- **$B_L$**: 대규모 플로우 저장 (packet-level 전략, $w_l$개 버킷, $d_l$개 셀, 24비트 카운터)
- 메모리 할당 비율: $B_L : B_S = 2 : 7$

### 삽입 (Algorithm 2)

| Flag | 조건 | 동작 |
|------|------|------|
| 0 | $H(f) \in [x_i, y_i]$ | $B_S$에 삽입 (오버플로우 시 $B_L$로 승격) |
| 1 | $H(e) \in [l_i, r_i]$ | $B_L$에 삽입 |

### 교체 전략 (Conservative Update)
- 기존 unbiased 방식: 카운터를 $n_{min}+1$로 증가 후 확률 $\frac{1}{n_{min}+1}$로 교체
- **LTCM 방식**: 교체 실패 시 기존 카운터 수정하지 않음 → 이미 기록된 대규모 플로우의 카운터 보호, heavy hitter 탐지 정확도 향상

### 쿼리 (Algorithm 3)
1. 경로 상에서 flow-level 전략으로 $f$를 측정하는 스위치를 찾아 $B_S$ 쿼리 → $n_s$
2. 경로 상 모든 스위치의 $B_L$ 쿼리 → $n_{l,i}$
3. **추정 크기**: $\hat{f}_s = n_s + \sum n_{l,i}$

---

## 6. 하드웨어 구현 (Tofino P4 Switch)

### 파이프라인 설계 (8 Stage)

```
Network packets → [Ingress Pipeline]
  Stage 0: Routing Tables (포워딩 규칙 + 경로 길이 계산)
  Stage 1-2: Interval Matching Tables (플래그 + 경로 길이 기반 구간 매칭)
  Stage 4-5: Small Buckets (Fingerprint Registers + Counter Registers)
  Stage 6-7: Large Buckets (Fingerprint Registers + Counter Registers)
→ Traffic Manager → Egress
         ↑ Resubmit (read-modify-write 필요 시)
```

### 주요 구현 기법
- **Read-Write 분리**: Read Phase에서 버킷 데이터를 메타데이터에 저장 → Write 필요 시 Tofino **resubmit** 기능으로 파이프라인 재진입
- **확률적 교체 단순화**: Tofino에서 나눗셈 미지원 → 충돌 시 카운터 -1, 0이 되면 fingerprint 교체

### 하드웨어 리소스 사용량

| 리소스 | 사용량 | 비율 |
|--------|--------|------|
| Hash Bits | 263 | 5.30% |
| SRAM | 57 | 5.90% |
| Map RAM | 54 | 9.40% |
| TCAM | 4 | 1.40% |
| Stateful ALU | 6 | 12.50% |
| VLIW Instr | 21 | 5.50% |
| Match Xbar | 68 | 4.90% |

- 대부분 10% 미만, Stateful ALU만 12.5% (레지스터 read/write 처리)
- Resubmit 비율: 최대 ~5%, 실제 배포 시 거의 0에 수렴

---

## 7. 수학적 분석

### FCM 해의 존재성 증명 (Section 5.1)
- 계수 행렬 $A$의 차원: $(2h-1) \times \frac{h(h+1)}{2}$
- 가우스 소거법으로 행 사다리꼴 변환 → 좌측에 $(2h-1) \times (2h-1)$ 단위행렬 형성
- $\text{rank}(A^*) = 2h - 1$ → 해 존재 보장
- **풀이 시간 복잡도**: $\mathcal{O}(h^4)$

### TC Sketch 추정 오류 한계 (Section 5.2)

플로우 $f_i$의 추정 크기:

$$\hat{c}_i = c_i + X_i - Y_i + t \cdot (P_i - Q_i)$$

- $X_i$: $B_S$에서의 fingerprint 충돌로 인한 증가분
- $Y_i$: $B_S$에서 교체로 인한 감소분
- $P_i, Q_i$: $B_L$에서의 동일 유형 오차

**상한 오류 한계**:
$$\Pr\{\hat{c}_i - c_i \geq \epsilon N\} \leq \frac{1}{\epsilon N 2^r}\left(\frac{N_s}{w_s} + \frac{tN_l}{w_l}\right)$$

**하한 오류 한계**:
$$\Pr\{c_i - \hat{c}_i \geq \epsilon N\} \leq \frac{N_s}{\epsilon N} \cdot \frac{e^{-\frac{i-1}{w_s}}(\frac{i-1}{w_s})^{d_s-1}}{w_s(d_s-1)!} + \frac{tN_l}{\epsilon N} \cdot \frac{e^{-\frac{i-1}{w_l}}(\frac{i-1}{w_l})^{d_l-1}}{w_l(d_l-1)!}$$

---

## 8. 실험 평가

### 실험 환경
- **데이터셋**: CAIDA 2019 (181.8M 패킷, 1.09M 플로우), IMC 2010 (61.7M 패킷, 6.89M 플로우)
- **토폴로지**: FatTree (4 core + 8 agg + 8 edge), SpineLeaf (4 spine + 8 leaf)
- **비교 대상**: CMAX, DS, UWRA, NSPA + 로컬 측정 알고리즘(CS, WS)
- **스위치당 메모리**: 24KB~96KB

### 파라미터 설정
- BSS: $p_r = 0.001$, 16비트 fingerprint, 4비트 카운터, 5KB
- TC Sketch: 8 cells/bucket, 16비트 fingerprint, $B_L$ 24비트 카운터, $B_S$ 16비트 카운터, $B_L:B_S = 2:7$, $\mathcal{T} = 15000$

### 8.1 측정 부하 균형

#### Flow-level (Figure 7)
- LTCM ≈ FCM > NSPA >> 나머지
- FCM은 NSPA 대비 최대 플로우 수가 16.07%~21.66% 낮으면서 계산 복잡도가 훨씬 낮음
- LTCM과 FCM 성능이 거의 동일 → 소수의 대규모 플로우에만 packet-level 적용하므로 flow-level 균형에 미미한 영향

#### Packet-level (Figure 8)
- LTCM이 모든 조합에서 **최소 표준편차** 달성
- FatTree+IMC에서 다른 솔루션 대비 표준편차 59.5%~80.3% 감소
- NSPA/FCM은 flow-level은 우수하지만 packet-level에서는 heavy-tail 분포로 인해 불균형 발생

### 8.2 Per-Flow Size Estimation (Figure 9)

| 환경 (48KB) | LTCM AAE 감소율 |
|-------------|----------------|
| vs. CMAX | 91.94% |
| vs. UWRA | **96.60%** |
| vs. DS | 96.24% |
| vs. WS_NSPA | 61.51% |
| vs. CS_NSPA | 86.87% |

- LTCM이 모든 메모리 크기에서 **최저 AAE** 달성
- WS/CS 단독 → WS_NSPA/CS_NSPA → LTCM 순으로 정확도 향상: flow-level 균형 + TC Sketch의 대/소 플로우 분리 효과

### 8.3 Heavy Hitter Detection (Figure 10, 11)

- LTCM이 거의 모든 $\theta$와 메모리 크기에서 **최고 $F_1$-Score** 및 **최저 AAE**
- TC Sketch의 대/소 플로우 분리 저장이 heavy hitter 탐지 정확도 향상에 기여
- IMC 데이터셋에서 WS, UWRA 등은 플로우 밀도 6.31배 증가로 성능 저하가 심각하지만, LTCM은 강건한 성능 유지

### 8.4 Throughput (Figure 12)

- LTCM이 FatTree에서 **최고 처리량** 달성 (SpineLeaf에서 UWRA와 유사)
- DS, WS, CS: 모든 패킷 처리 → 처리량 최저
- UWRA: 샘플링 기반으로 처리량은 높지만 정확도 희생
- LTCM: **높은 처리량과 높은 정확도 동시 달성**

---

## 9. 핵심 기여 요약

| 기여 | 내용 |
|------|------|
| **Two-level 전략** | Flow-level + Packet-level 측정 부하 균형을 동시에 달성하여 오버헤드 감소 및 정확도 향상 |
| **Interval-matching (FCM)** | 토폴로지 특성을 활용한 경량 flow-level 협업 프레임워크, $\mathcal{O}(h^4)$ 복잡도로 네트워크 규모와 무관한 확장성 |
| **BSS** | Ingress 스위치에서 5KB 메모리로 대규모 플로우를 실시간 탐지, 전략 전환 결정 |
| **TC Sketch** | 대규모/소규모 플로우를 분리 저장하여 해시 충돌 감소, 패킷 처리 속도 및 정확도 향상 |
| **Tofino 구현** | P4로 구현, 리소스 사용량 대부분 10% 미만, resubmit 비율 최대 5% |

---

## 10. 한계 및 논의

- ECMP 등 멀티패스 환경에서는 쿼리 시 모든 경로를 순회해야 하므로 **쿼리 오버헤드 증가** 가능
- BSS의 임계값 $\mathcal{T}$ 설정이 성능에 영향 — 최적값은 트래픽 패턴에 의존
- TC Sketch의 $B_L : B_S$ 메모리 비율(2:7)이 고정 — 트래픽 분포에 따른 적응형 할당은 미탐구
- Tofino 구현에서 resubmit 연산이 처리량에 약간의 영향 (최대 ~5%)
