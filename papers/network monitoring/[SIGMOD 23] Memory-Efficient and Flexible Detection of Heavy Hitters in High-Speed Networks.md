# SwitchSketch: Memory-Efficient and Flexible Detection of Heavy Hitters in High-Speed Networks

## 기본 정보
- **저자**: He Huang, Jiakun Yu, Yang Du, Jia Liu, Haipeng Dai, Yu-E Sun
- **소속**: Soochow University / Nanjing University
- **게재**: Proc. ACM Manag. Data (SIGMOD), Vol. 1, No. N3, Article 214, November 2023
- **DOI**: 10.1145/3617334
- **소스코드**: https://github.com/duyang92/switch-sketch-paper/
- **키워드**: Sketch, Heavy Hitter Detection, Network Traffic Measurement, Network Monitoring

---

## 1. 연구 배경 및 문제 정의

### Heavy Hitter란?
- 측정 기간 동안 사전 정의된 임계값을 초과하는 크기의 네트워크 플로우
- 수는 적지만 전체 대역폭의 상당 부분을 차지
- 탐지 대상: **threshold-t** (크기 > t인 플로우) 또는 **top-k** (가장 큰 k개 플로우)

### 핵심 도전과제
- 라우터 온칩 메모리(보통 <10MB)와 고속 네트워크 트래픽 양의 **극심한 불일치**
- 트래픽의 **동적이고 편향된(skewed) 분포**: 소수의 heavy hitter + 다수의 mice flow

### 기존 알고리즘의 한계

| 카테고리 | 대표 알고리즘 | 한계 |
|---------|-----------|-----|
| **Sketch 기반** (Count-Min, CM-CU 등) | 여러 플로우가 카운터를 공유 | 큰 카운터가 mice flow에 낭비, mice flow가 heavy hitter 측정에 노이즈 유발 |
| **KV 기반** (HeavyGuardian, DHS 등) | KV 쌍으로 heavy hitter 분리 | KV 쌍의 수/크기를 배포 시 고정 → 동적 트래픽에 대응 불가. DHS는 추가 메모리 접근 필요 + 카운팅 범위 제한(≤65536) |

---

## 2. SwitchSketch 핵심 아이디어

> 스케치가 **서로 다른 모드 간 동적으로 전환(switch)** 하여 미지의 네트워크 트래픽과 불균형한 플로우 분포에 적응. **메모리의 모든 비트를 최대한 활용**.

### 설계 핵심 요소 4가지

| 요소 | 설명 |
|------|------|
| **가변 길이 셀 (Variable-length Cells)** | 큰 셀은 big flow, 작은 셀은 mice flow 추적 → 플로우 크기에 대한 유연성 |
| **축소 카운터 (Shrunk Counters)** | 셀의 카운터를 ε비트 축소하여 새 셀 공간 확보 → 메모리에 대한 유연성 |
| **임베디드 메타데이터 (Embedded Metadata)** | 버킷 내 τ비트로 현재 모드를 인코딩 → 추가 메모리 접근 제거 |
| **전환 가능 모드 (Switchable Modes)** | 셀 조합을 동적으로 변경하여 heavy hitter에 집중 |

---

## 3. 데이터 구조 설계

### 유연한 버킷 구조 (Flexible Bucket Structure)

```
┌──────┬────────────────────────────────────────────┐
│ meta │  C₁ cell │ C₁ cell │ ... │ C₂ cell │ C₃ cell │
│ (τ)  │  (w₁)   │  (w₁)   │     │  (w₂)    │  (w₃)    │
└──────┴────────────────────────────────────────────┘
         ◀──────────── W bits (고정 크기 버킷) ──────────▶
```

- **버킷**: m개의 고정 크기(W비트) 버킷 배열 $B = \{B_1, B_2, ..., B_m\}$
- **셀**: Key 필드(fingerprint) + Counter 필드로 구성
  - 셀 $C_i$: $8(i+1)$ 비트 (축소 전), Key와 Counter 각각 절반
  - fingerprint 사용 이유: 5-tuple은 104비트로 너무 긺 → 해시값으로 압축

### 셀 축소 (Counter Shrinking)
- 셀의 카운터를 ε비트(0~2) 축소하여 새 셀 공간 확보
- 카운팅 범위 유지: $\frac{1}{2^\varepsilon}$ 확률로 증가/감소
- 예: 16비트 셀(8비트 key + 8비트 counter) → 15비트(8+7) 또는 14비트(8+6)

### Two-mode Active Counter
- 가장 큰 셀 $C_L$의 카운터가 오버플로우 시 사용
- **Normal Mode**: 일반 증가 (값 0~127)
- **Exponential Mode**: 계수부(α) × $2^{β+γ}$로 추정, $\frac{1}{2^{β+γ}}$ 확률로 증가
- 카운팅 범위: $[0, 2^{2^\gamma + \phi - 1} - 2^{2^\gamma - 1 + \gamma}]$

### 모드 (Mode)
- 버킷 내 셀 조합 = 하나의 모드
- 초기: 작은 셀로 최대한 많은 플로우 추적
- 카운터 오버플로우 시 → 더 큰 셀로 **모드 전환(switching)**

### 셀 설정 최적화 (Min-Sum)
- 목표: 버킷 내 미사용 비트의 총합 최소화

$$\min \sum_{i=1}^{L} \lambda_i, \quad \lambda_i = W - \tau - \lfloor \frac{W - \tau}{w'_i} \rfloor w'_i$$

- W=128, L=3일 때 최적: $(w'_1, w'_2, w'_3) = (15, 24, 31)$, $\tau = 4$

---

## 4. 인코딩 기반 전환 (Encoding-based Switching)

### 모드 테이블 (W=128, L=3)

| 모드 $(n_1, n_2, n_3)$ | 인코딩 | 모드 $(n_1, n_2, n_3)$ | 인코딩 |
|---|---|---|---|
| (8, 0, 0) | 0000 | (3, 2, 1) | 1000 |
| (6, 1, 0) | 0001 | (1, 3, 1) | 1001 |
| (5, 2, 0) | 0010 | (4, 0, 2) | 1010 |
| (3, 3, 0) | 0011 | (2, 1, 2) | 1011 |
| (1, 4, 0) | 0100 | (0, 2, 2) | 1100 |
| (0, 5, 0) | 0101 | (1, 0, 3) | 1101 |
| (6, 0, 1) | 0110 | (0, 1, 3) | 1110 |
| (4, 1, 1) | 0111 | (0, 0, 4) | 1111 |

- 16개 합법 조합 → **4비트 메타데이터**로 현재 모드 표현
- 작은 모드 → 큰 모드 방향으로만 전환 (heavy hitter에 집중)

---

## 5. 워크플로우

### 4가지 연산

#### 1) Initialization
- 모든 버킷을 첫 번째 모드(0000)로 설정, key/counter 초기화

#### 2) Insertion (패킷 도착 시)
- 해시로 버킷 선택: $j = h(f_i) \% m$
- 메타데이터 디코딩 → 셀 확인 (큰 셀부터 탐색)

| Case | 상황 | 동작 |
|------|------|------|
| **Case 1** | 플로우 기록됨 + 카운터 오버플로우 없음 | 카운터 +1 |
| **Case 2** | 플로우 기록됨 + 카운터 오버플로우 | Switch 호출 → 더 큰 셀로 전환. 실패 시 exponential decay로 작은 플로우 제거 |
| **Case 3** | 새 플로우 + 빈 셀 있음 | 가장 작은 빈 셀에 삽입 |
| **Case 4** | 새 플로우 + 빈 셀 없음 | 가장 작은 셀에 exponential decay 적용 ($Pr = b^{-\log_2 c_k}$, b=1.08) |

#### 3) Switch
- 현재 셀 $C_j$ 오버플로우 → $C_{j+1}$로 교체
- 작은 셀부터 제거하며 새 모드에 맞는 할당 탐색
- 실패 시 현재 모드 유지 + 실패 보고

#### 4) Query
- **Highest type first**: $C_L$ → $C_1$ 순서로 탐색 (heavy hitter가 대부분의 트래픽이므로 고수준에서 조기 종료)
- **온라인 탐지**: 보조 리스트에 threshold-t 초과 플로우 기록
- **오프라인 탐지**: 측정 후 모든 플로우 순회 + **double-check** 메커니즘

### Double-Check 메커니즘 (오프라인 전용)
- 두 번째 해시 함수 $h'(\cdot)$로 패킷을 50:50 확률로 두 버킷에 분배
- 두 카운터 $c_i$, $c'_i$ 비교:

$$1 - \frac{\min(c_i, c'_i)}{\max(c_i, c'_i)} < \theta \quad (\theta = 0.6)$$

- 합법: 추정 크기 = $c_i + c'_i$ / 불법: 추정 크기 = 1
- mice flow의 fingerprint 충돌로 인한 **false positive 대폭 감소**
- 삽입 시 추가 오버헤드 없음 (패킷당 여전히 1회 해시)

---

## 6. 구현

### 하드웨어 (NetFPGA-1G-CML)
- 3단계 파이프라인: Memory Read → (Query + Switch + Insert) → Memory Write
- 전체 파이프라이닝으로 높은 처리량 달성

| 버킷 크기 | 처리량 (Mpps) |
|---------|-------------|
| 64비트 | 114.29 |
| 128비트 | 114.29 |
| 256비트 | 109.96 |

### 소프트웨어 (C++)
- Intel Xeon E5-2643 v4 @3.40GHz, 256GB RAM
- Murmur Hash 사용

---

## 7. 실험 결과

### 실험 환경
- **데이터셋**: CAIDA-2016 (31M 패킷, 0.58M 플로우), CAIDA-2019 (36M 패킷, 0.37M 플로우)
- **비교 대상**: HeavyGuardian, WavingSketch, DHS
- **기본 설정**: W=128비트, b=1.08, θ=0.6, γ=4

### 온라인 Threshold-t 탐지

#### FNR (False Negative Ratio) — CAIDA-2016, t=750, M=150KB
| 알고리즘 | 미탐지 플로우 수 (top=1000 이상 4141개 중) |
|---------|------|
| **SwitchSketch** | **2** |
| DHS | 52 |
| WavingSketch | 135 |
| HeavyGuardian | 258 |

#### F_β-Score (β=2, Recall 중시)
- SwitchSketch: 모든 메모리 크기/threshold에서 **항상 0.938 이상**
- 50KB에서 SwitchSketch 0.951 vs HeavyGuardian 0.731 / WavingSketch 0.871 / DHS 0.872

### 오프라인 Top-k 탐지

#### Precision Rate — M=100KB
- k=512: SwitchSketch **100%** (512/512) vs HeavyGuardian 493, WavingSketch 506, DHS 491
- k=1024: SwitchSketch **99%+**

#### ARE (Average Relative Error)
- SwitchSketch의 ARE: 다른 알고리즘 대비 **2~3 자릿수(orders of magnitude) 낮음**
- SOTA 대비 ARE **30.77%~99.96% 감소**
- Double-check 적용 시 ARE: 17.8 → **0.10** (M=50KB, CAIDA-2016)

### 처리량 (Throughput)

| 알고리즘 | M=100KB (Mops) | M=200KB (Mops) |
|---------|----------------|----------------|
| WavingSketch | ~50 (fingerprint 미계산) | ~55 |
| **SwitchSketch** | **~25** | **~32** |
| DHS | ~17 | ~22 |
| HeavyGuardian | ~20 | ~25 |

- SwitchSketch: fingerprint 기반 알고리즘 중 **최고 처리량**
- "Highest type first" 탐색으로 조기 종료 + 임베디드 메타데이터로 추가 메모리 접근 제거

---

## 8. 민감도 분석 (Ablation Study)

| 파라미터 | 최적값 | 근거 |
|---------|------|------|
| 버킷 크기 W | **128비트** | 정확도와 처리량 간 최적 트레이드오프 |
| Double-check θ | **0.6** | 높은 정확도 + 낮은 ARE 균형 |
| Exponent γ | **4** | 추정 정확도 + 충분한 카운팅 범위 균형 |
| Exponential decay b | **1.08** | 지나친 교체(b↑) vs elephant flow 무감(b↓) 균형 |
| 축소 비트 ε | **0~2** | ε=4이면 업데이트 확률 저하로 정확도 감소 |

### 동적 할당 vs 고정 할당
- SwitchSketch의 동적 할당이 모든 고정 할당(4L0S, 3L2S, 2L4S, 1L6S)을 압도
- 동적/예측 불가능한 트래픽에서 고정 할당은 실용적이지 않음

### Dataset Skewness
- skewness 0.3 (균등에 가까운 분포): SwitchSketch **유의미하게 우수** (WavingSketch F_β=0.15)
- skewness 3.0 (극도로 편향): 모든 알고리즘 높은 성능

---

## 9. 관련 연구와의 차별점

| 분야 | 기존 연구 | SwitchSketch 차별점 |
|------|---------|-----------------|
| Sketch 기반 | Count-Min, CM-CU, SALSA | mice flow 노이즈 문제 해결 — 모드 전환으로 heavy hitter에 집중 |
| KV 기반 | HeavyGuardian, DHS, WavingSketch | 고정 KV 할당의 한계 극복 — **세밀한(fine-grained) 인코딩 기반 전환** |
| 하드웨어 | Elastic Sketch (SIGCOMM'18) | **NetFPGA 구현** 검증, SW/HW 양쪽 호환 |

---

## 10. 핵심 기여 요약

1. **유연한 버킷 구조**: 가변 길이 셀 + 축소 카운터로 메모리의 모든 비트를 활용
2. **인코딩 기반 전환**: 임베디드 메타데이터(4비트)로 16개 모드 간 동적 전환, 추가 메모리 접근 없음
3. **Double-check 메커니즘**: 오프라인 탐지에서 ARE를 2~3 자릿수 감소
4. **SW/HW 양쪽 구현**: CPU(C++) + NetFPGA(최대 114 Mpps)
5. **100KB 메모리에서 F_β > 0.938, top-k PR > 99%, ARE 30.77~99.96% 감소** (SOTA 대비)
