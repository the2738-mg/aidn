# AI Model LifeCycle · InfiniBand(RDMA) · RoCEv2

# 1. AI 모델 생명주기와 병목

## 1.1 훈련 · 추론 · 에이전틱 AI

세 가지는 동일 선상의 단계가 아니다. **훈련과 추론은 모델 생명주기의 단계**이고, **에이전틱 AI는 그 모델을 사용하는 애플리케이션 구조**다. 에이전틱 시스템 내부에서는 보통 추론이 여러 번 수행된다.

- **훈련(Training)**: 데이터로부터 모델 가중치, 즉 "지능 자체"를 만드는 과정. 가중치가 계속 변경된다.
- **추론(Inference)**: 학습이 끝난 모델에 입력을 넣어 예측·답변을 생성하는 과정. 가중치는 고정된다.
- **에이전틱(Agentic) AI**: 추론 모델을 두뇌로 사용하면서 `계획 → 도구 실행 → 결과 확인 → 재계획`을 반복해 실제 목표를 완수하는 시스템 아키텍처.

| 구분 | 훈련 | 추론 | 에이전틱 |
| --- | --- | --- | --- |
| 핵심 목적 | 패턴 학습 | 결과 생성 | 목표 달성 위해 여러 행동 |
| 모델 가중치 | 변경됨 | 고정 | 고정 모델 반복 호출 |
| 입력 | 학습 데이터셋 | 프롬프트 등 | 목표·상태·도구 결과·기억 |
| 출력 | 모델 체크포인트 | 예측/생성 토큰 | 실제 업무 결과·환경 변경 |
| 주요 연산 | Forward+Backward+Optimizer | Forward 중심 | 다수 추론 + 도구 호출 |
| 주요 자원 | GPU/TPU 연산·네트워크 | HBM·KV Cache·메모리 대역폭 | 추론 인프라 + DB/API/스토리지/보안 |
| 주요 지표 | 학습시간·MFU·Goodput | TTFT·TPOT·비용/토큰 | 성공률·비용/결과·도구 오류율 |
| 주요 위험 | 학습 실패·데이터 문제 | 지연·비용·환각 | 연쇄 오류·권한 남용 |

에이전틱 AI 시대의 핵심 지표는 단순 `tokens/sec`이 아니라 **Cost per Successful Outcome(완료된 업무당 비용)** 으로 이동하고 있다.

## 1.2 왜 GPU인가 (질문 1)

딥러닝의 핵심 연산은 **행렬 곱셈 누산(MAC/GEMM)** 이다. 동일 연산을 거대한 데이터에 동시에 적용하는 작업으로, 수천~수만 개의 단순 코어로 병렬 처리할 수 있는 GPU에 최적이다. CPU는 소수의 복잡한 코어로 순차·분기 작업에 강하지만, 대규모 동형(同形) 병렬 연산에서는 GPU의 처리량을 못 따라간다.

| 단위 | 초당 연산 |
| --- | --- |
| GFLOPS | 10^9 |
| TFLOPS | 10^12 |
| PFLOPS | 10^15 |
| EFLOPS | 10^18 |

## 1.3 진짜 병목은 연산이 아니라 메모리 (질문 2)

GPU 연산 코어는 충분히 빠른데, **메모리에서 데이터를 가져오는 속도가 못 따라가** 코어가 데이터를 기다리며 노는 "데이터 기아(starvation)" 상태가 발생한다.

### 메모리 계층과 트레이드오프

메모리는 한 종류만 쓰지 않는다. **저장용량·접속시간·대역폭·비용**이 상충하기 때문이다.

```
레지스터 > 캐시(L1/L2/L3) > DRAM/VRAM > NVMe SSD > SATA SSD > HDD
빠름·작음·비쌈  ───────────────────────────────►  느림·큼·저렴
```

| 경로 | 지연 |
| --- | --- |
| ALU ↔ 레지스터 | 0.312 ns (1 clock) |
| ALU ↔ L3 캐시 | 7.5 ns (24 clocks) |
| CPU ↔ DRAM | 45 ns (150 clocks) |
| GPU ↔ VRAM(24GB) | 250 ns (800 clocks) |
| CPU ↔ GPU | ~300 ns |

### HBM (High Bandwidth Memory)

HBM은 클럭(처리 속도)을 높이는 대신 **인터페이스 너비(Interface Width)** 를 확장한다. DRAM 칩을 수직으로 쌓고, 실리콘 인터포저 위에서 GPU와 HBM을 넓은 버스로 연결한다. GDDR이 클럭은 높아도 I/O 핀 수까지 고려하면 HBM의 총 대역폭이 더 높다.

### SRAM vs HBM — 그래도 병목

GPU 내부 SRAM(19TB/s)은 HBM(~1.2~3.35TB/s)보다 약 12.5배 빠르다. 결국 연산 코어 ↔ 메모리 사이의 대역폭 격차(폰노이만 병목)는 계층을 내려갈수록 커진다.

## 1.4 2026년의 병목 = 메모리 대역폭 (메모리 장벽)

2012~2022년 NVIDIA GPU의 FP64 연산 성능은 80배 증가했지만 메모리 대역폭은 17배만 증가했다. `80 ÷ 17 ≈ 4.7` → 연산과 메모리 공급 능력 사이의 불균형이 약 4.7배 확대됐고, 이를 **메모리 장벽(The Memory Wall)** 이라 부른다.

→ 전략: **최대한 Scale-Up(NVLink 도메인)으로 해결하고, 더 큰 규모에서는 Scale-Out(IB/RoCEv2)** 한다.

## 1.5 추론의 두 단계: Prefill vs Decode

- **Prefill (사전 채우기)**: 입력 프롬프트 전체를 한 번에 병렬 처리. 대규모 행렬 곱이라 텐서 코어가 활성화되는 **Compute-bound** 단계. 1만 토큰 프롬프트가 H100에서 200~400ms. 결과로 KV Cache 생성. → TTFT(첫 토큰 시간) 결정.
- **Decode (디코딩)**: 토큰을 하나씩 순차 생성. 각 토큰이 이전 모든 토큰에 의존해 병렬 불가. 매 단계 모델 가중치(70B FP16 = 140GB) + KV Cache를 메모리에서 읽고 작은 연산 후 다시 저장. **Memory-bandwidth-bound** 단계. → TPOT(토큰당 시간) 결정.

예시: TTFT ~300ms(prefill 1회) + TPOT ~30ms × 200 토큰 = **총 decode ~6,000ms가 지배적**. 이것이 출력 토큰 가격이 입력보다 3~10배 비싼 이유다.

## 1.6 KV Cache — 대화가 길수록 비싸지는 이유

self-attention에서 각 새 토큰은 이전 모든 토큰의 Query·Key·Value를 참조한다. 캐싱이 없으면 매 단계 전체 컨텍스트를 재계산해 비용이 제곱으로 증가한다. KV Cache는 K·V를 한 번만 계산·저장하고 이후 조회한다.

```
KV Cache 크기 ∝ 배치 크기 × 레이어 수 × 컨텍스트 길이 × Hidden 크기
```

70B 모델 32K 컨텍스트는 가중치 140GB 외에 KV Cache만 ~8GB가 필요하고, 배치 32면 KV Cache가 모델 가중치를 초과할 수 있다. 추론 비용 구성: `모델 가중치 읽기 + KV Cache 읽기 + GPU 대기 시간 + 전력`. 네 항목 중 셋이 "메모리가 GPU에 충분히 빠르게 공급하지 못하는" 같은 원인이다.

## 1.7 네오클라우드

GPU 서비스에 특화된 전문 클라우드(GPUaaS). 최대 무기는 가격 경쟁력으로, DGX H100 동급 기준 네오클라우드 평균 시간당 $34 vs 하이퍼스케일러 $98(약 65% 절감). 핵심 기술: NVLink 도메인, InfiniBand(+SHARP), RoCE, 쿠버네티스, GPU 분할(Fractioning)·동적 할당.

## 1.8 에이전틱 AI가 만드는 인프라 병목 (요약)

추론 호출·토큰 수 폭발 / HBM·KV Cache 압박(캐시 위치 인식 라우팅) / Prefill·Decode 분리 서빙(NVIDIA Dynamo, NIXL) / 네트워크가 추론의 일부(TP·MoE All-to-All·KV 전송) / 스토리지 계층화 / 지연 vs GPU 사용률 충돌 / 전력·냉각 / 상태 보유 장애 처리 / 평가·관측성 / 보안·권한(Prompt Injection).

---

# 2. 딥러닝과 트랜스포머

## 2.1 신경망의 기본 원리

키·몸무게로 어른/아이를 판단하는 예: 각 입력에 **가중치**를 곱해 더한 값이 **역치**를 넘으면 1(어른). 처음 가중치는 랜덤이라 틀린다. **학습**이란 오차가 사라질 때까지 가중치를 반복 조정하는 과정이다.

## 2.2 딥러닝

층(layer)을 깊게 쌓아 복잡한 데이터를 처리하는 것이 **딥(deep)러닝**이다.

- 데이터 흐름: `입력층 → 은닉층 → 출력층` 전파(forward).
- 출력 오차가 생기면 출력에 가까운 가중치부터 거꾸로 수정(**역전파, backpropagation**).
- 사람이 특징을 지정하지 않아도 **스스로 특징을 추출**한다. 이미지는 필터로 직선·곡선 특징을 뽑아 조합해 확률적으로 예측(CNN).

## 2.3 언어 처리 — 유연한 구조

언어는 입력이 단어/문장으로 가변적이라 유연한 구조가 필요하다. 초기 RNN 류는 앞 단어를 기억해 다음을 인식하는 과정을 반복했으나 순차 처리라 느렸다.

## 2.4 LLM = 다음 단어 예측기

- LLM의 본질은 **다음 토큰의 확률 분포 예측**. 수백억~수천억 파라미터, 처음엔 랜덤.
- 훈련 반복으로 예측이 정답에 가까워지도록 역전파로 조정. 충분히 반복하면 처음 보는 문장도 그럴듯하게 생성.

## 2.5 트랜스포머 (Attention Is All You Need, 2017)

2017년 구글 트랜스포머는 순차 처리의 한계를 깨고 **병렬 처리**를 가능하게 했다. 핵심은 **어텐션(주의) 메커니즘**.

1. **임베딩**: 문장 → 의미·맥락 숫자 벡터. 토큰화 + 토큰 임베딩 + 위치 인코딩.
2. **트랜스포머 블록**(GPT-2 기준 12개): **멀티 헤드 어텐션 + MLP**.
   - **셀프 어텐션**: 토큰 임베딩을 Q·K·V로 변환, 토큰들이 맥락에 따라 서로 가중치 조절(예: "눈"이 내리는 눈인지 사람 눈인지 구분).
   - **멀티 헤드**: 여러 패턴 동시 학습. **Masked Self Attention**: 미래 토큰 가림. 출력 = Attention × Value.
   - **MLP(Feedforward)**: 언어 패턴 저장, 토큰 표현 개선.
3. **확률(Output Logits)**: 마지막 출력 임베딩 × 최종 가중치 → 다음 토큰 확률 분포(temperature·sampling으로 다양성 조절).

---

# 3. 대규모 학습 — 2조 파라미터 LLM과 NCCL

## 3.1 세 가지 제약

H100 한 장은 80GB HBM3. 2조(2T) 파라미터 학습은 세 제약이 동시에 걸린다.

- **메모리**: 파라미터당 ~16바이트(가중치 복사본+그래디언트+옵티마이저 상태) → 2T × 16B = **약 32TB** → H100 **약 400장**.
- **Activations Cache**: 시퀀스 4096 기준 한 번 642GB, 배치 32면 **약 20TB**(모델 상태 32TB에 맞먹음).
- **컴퓨트**: 약 `4.8 × 10^26` Ops. H100 50% 효율로 1장 **30,700년**, 16,384장으로 약 **2년**.

## 3.2 병렬화 기법

| 기법 | 무엇을 나누나 | 통신 특성 | 비고 |
| --- | --- | --- | --- |
| **Data (DP)** | 배치 데이터 | 그래디언트 All-Reduce | 모델을 모든 GPU에 복사 |
| **Tensor (TP)** | 하나의 행렬 연산 | 단계마다 블로킹, 초고대역 필요 | 가장 빠른 링크(NVLink)에서만 |
| **Pipeline (PP)** | 모델 레이어 | 단계 간 전달 | GPU bubble 발생 |
| **Expert (EP/MoE)** | MoE Expert | All-to-All | 저지연이 매우 중요 |
| **Context/Sequence** | 긴 시퀀스 | — | 긴 컨텍스트 분산 |

- **Ring All-Reduce** = Reduce-Scatter + All-Gather. N개 GPU를 논리적 링으로 배치, 그래디언트를 N 청크로 분할, 매 단계 모든 링크가 동시 송수신해 **유휴 링크가 없다**. 필요 대역폭은 **GPU 개수와 무관, 그래디언트 크기에 비례**.
- **TP**는 층마다 블로킹 동기화 → **가장 빠른 링크에서만 효율적**. 그래서 Llama3는 TP를 서버 내부 8 GPU로 제한(NVLink). *(→ 이 "8" 전제는 SXM/NVSwitch 기반이다. 하단 추가 고려사항 참조.)*
- **PP**: GPU bubble를 **마이크로 배치**로 해결. Zero-bubble로 처리량 23% 향상.
- **MoE**: 토큰마다 Expert가 달라 **All-to-All** 통신이 빈번 → **레이턴시가 결정적**.

## 3.3 메모리 절감

ZeRO 샤딩(상태 분산), 그래디언트 체크포인트(메모리↓·컴퓨팅 ~33%↑), Selective Checkpoint, Flash Attention.

## 3.4 하드웨어 장애 (운영 현실)

Meta Llama3 54일 학습(16,384 GPU)에서 **매 3시간 꼴 장애**. 방어=체크포인트, 419건 중 사람 개입 3건뿐. 1개 GPU만 느려도 나머지 16,383개를 멈추게 할 수 있다(straggler).

## 3.5 전체 학습 파이프라인 (5계층)

1. 데이터 준비(웹 1PB → 중복제거·토큰화 → 160TB 병렬 파일시스템)
2. 데이터 구성(web 50% / math 25% / code 17% / 기타 8%, 6단계 윈도우 확장)
3. 공급(GPU가 인덱싱 슬라이스 가져옴, CPU prefetch queue)
4. 연산(NVLink TP + IB/RoCEv2 PP 동시 실행)
5. 제어(스케줄러, 병렬 메시, 체크포인트, 모니터링, 오케스트레이터)

## 3.6 NCCL과 집합 통신

### 기본 4 패턴

| 패턴 | 방향 | 데이터 | 용도 |
| --- | --- | --- | --- |
| Broadcast | root → all | 같은 값 복제 | 초기 weight 동기화 |
| Scatter | root → all | 서로 다른 chunk | 배치 분배 |
| Gather | all → root | concat | 결과 수집 |
| Reduce | all → root | element-wise op | loss/gradient 집계 |

조합: **AllReduce** = Reduce + Broadcast(또는 ReduceScatter + AllGather), **AllGather** = Gather + Broadcast, **ReduceScatter** = Reduce + Scatter, **AlltoAll** = Scatter 전치. 대표 8종: `ncclAllReduce`(DDP gradient sync), `ncclAllGather`(ZeRO-3/FSDP), `ncclReduceScatter`(FSDP gradient), `ncclAlltoAll`(MoE token dispatch) 등.

### 데이터 경로 (질문 5의 핵심)

**Intra-node 우선순위**: ① P2P over NVLink → ② P2P over PCIe → ③ SHM(host memory 경유) → ④ NIC loopback.

**Inter-node**:

```
GPU kernel → GPU vidmem → (CPU proxy thread) → NIC → wire → NIC → ...
                               └→ RDMA write (IB/RoCE) 또는 socket send
```

- GPU 커널이 버퍼를 채우면 CPU의 `ncclProxyProgress` 스레드가 NIC의 RDMA write/socket send를 post한다. **CPU가 데이터 자체를 만지지는 않지만 NIC 오케스트레이션은 host 스레드의 몫**.
- **GPUDirect RDMA 가능** 시(기본 `PATH_PXB`): NIC가 GPU 메모리를 직접 read/write(`ncclTopoCheckGdr` 결정, `NCCL_NET_GDR_LEVEL` override).
- **불가능** 시: host pinned memory staging — `GPU → host copy → NIC RDMA → host → GPU copy`로 PCIe를 두 번 더 건넌다.
- 전송 앞에 양쪽 버퍼 준비 상태를 합의하는 **rendezvous**.

---

# 4. CPU/GPU 상세 동작과 RDMA

## 4.1 Linux 프로세스가 코드를 실행할 때 (질문 3)

```
Application Process ↓ CPU ↕ CPU Cache ↕ Main Memory(RAM) ↕ Disk
```

**CPU는 디스크 데이터를 직접 읽지 못한다.** 먼저 RAM으로 올라온 뒤 접근한다(파일 읽기엔 커널·Page Cache 개입). 캐시 계층: CPU가 데이터를 요청하면 L1 → L2 → L3 순으로 찾고(히트 시 즉시 반환), 모두 미스면 RAM에서 캐시 라인 단위로 끌어올린다. RAM에도 없으면 페이지 폴트 → 커널이 NVMe에서 읽어 Page Cache 적재.

## 4.2 DMA — CPU가 바이트를 직접 옮기지 않는다

DMA가 없으면 CPU가 `Disk Controller 읽기 → Register → RAM 쓰기`를 반복해 CPU 사용률·인터럽트·대역폭 낭비가 커진다. **DMA**는 CPU가 `Source/Destination 주소, 크기, 방향, 완료 알림`만 전달하면 DMA 장치가 데이터를 옮긴다. 단 CPU는 여전히 Descriptor 생성·주소 매핑·권한 확인·드라이버 실행·완료 처리를 담당한다.

## 4.3 단일 GPU 서버에서 PyTorch 실행 (질문 4)

```
Python → PyTorch C++ → CUDA Runtime → NVIDIA Driver → GPU Command Queue → GPU Kernel → SM/Tensor Core
```

`cpu_tensor.to("cuda")` 경로: `Host RAM → (PCIe, GPU Copy Engine + DMA) → GPU VRAM`. 실제 복사는 **GPU Copy Engine과 DMA**가 하고 **CPU는 설정만** 한다. 연산은 GPU의 SM/Tensor Core.

**OS Kernel vs GPU Kernel(용어 혼동 주의)**: OS Kernel은 프로세스 스케줄링·가상 메모리·PCIe 장치·드라이버; GPU Kernel은 GPU에서 도는 병렬 함수(행렬 곱·Softmax·Attention 등).

## 4.4 두 GPU 서버 간 통신 — TCP/IP vs RDMA

### 일반 TCP/IP (CPU·커널 집약적)

```
GPU1 → Host RAM1 → CPU/Kernel1 → Socket Buffer → NIC1 → Network
     → NIC2 → Kernel/Socket Buffer → Host RAM2 → GPU2
```

### RDMA

한 서버의 **RNIC가 다른 서버의 등록된 메모리에 직접 접근**한다. 대표 기술: InfiniBand, RoCEv2, iWARP, AWS EFA, NVIDIA ConnectX.

- **Memory Registration**: 페이지 고정(pinning), 주소 변환, 권한 생성, Local/Remote Key 발급.
- **Queue Pair**: Send + Receive Queue + Completion Queue.
- **연산**: RDMA Write(원격 메모리에 씀, 원격 CPU 미관여) / RDMA Read(원격 메모리 읽음) / Send·Receive(메시지 기반).

## 4.5 RDMA 전송 유형 — IB vs RoCE vs RoCEv2

- **InfiniBand(IB)**: RDMA 전용 네트워크 패브릭(IBTA 관리).
- **RoCEv1**: 링크 계층만 이더넷 → 현재 미사용.
- **RoCEv2**: IB 네트워크 계층을 **IP + UDP**로 대체, IB BTH·페이로드 헤더 유지 = **Ethernet + IP + UDP 위에 IB RDMA transport**. 라우팅 가능해 현재 모든 RoCE 구현은 RoCEv2(발음 '로키').

## 4.6 GPU 서버 간 RDMA — Host 경유 vs GPUDirect

```
[Host RDMA]      GPU1 VRAM → Host RAM1 → RNIC1 → Network → RNIC2 → Host RAM2 → GPU2 VRAM
[GPUDirect RDMA] GPU1 VRAM → RNIC1 → Network → RNIC2 → GPU2 VRAM   (Host RAM 복사 제거)
```

## 4.7 Control Path vs Data Path

| Control Path (CPU) | Data Path (장치) |
| --- | --- |
| 코드 실행·커널 실행 요청, DMA Descriptor | Disk → RAM DMA, RAM → VRAM DMA |
| RDMA QP 관리·메모리 등록, 동기화·오류 처리 | GPU Kernel 계산, RNIC↔RNIC, VRAM↔VRAM GPUDirect |

> RDMA·GPUDirect도 CPU·OS를 완전히 제거하지 않는다. **초기 설정·메모리 등록·Queue 관리·연결 제어·오류 처리에는 CPU·커널이 필요**하고, 반복 대용량 전송 경로에서만 개입을 최소화한다.

---

# 5. AI 데이터센터 네트워크 설계 (논리)

## 5.1 패브릭 분리

AI/ML 워크로드(데이터 수집·전처리 → 학습 → 배포·모니터링)를 위해 목적이 다른 패브릭을 분리한다.

| 패브릭 | 목적 |
| --- | --- |
| **Storage Fabric** | 데이터셋·체크포인트 I/O |
| **Training Fabric** | GPU 간 집합통신, 저지연·고대역 |
| **Inference Fabric** | 추론 서빙, KV 전송 |

## 5.2 Rail-Optimized Design (ROD)

서버 내부 GPU는 내부 스위치(NVSwitch)로 통신(외부보다 빠름). 8개 초과 GPU가 필요하면 여러 서버 분산 → **east-west 트래픽**.

- 각 GPU에 매핑된 **NIC를 여러 리프 스위치에 연결**, 서버 간 **원홉(one-hop)** 통신으로 지연 최소화.
- 서버의 **각 GPU는 서로 다른 "레일"**, 서버 간 **같은 번호 GPU가 같은 레일**에 매핑.

```
[Server 1] Rail4:GPU4 → Rail4:Leaf4 → Spine:Rail5 → Rail5:Leaf5 → Rail5:GPU5 [Server N]
```

## 5.3 256 GPU 클러스터 예시

256 GPU(서버 32 × GPU 8) = 스위치당 32×400Gbps 다운링크 × 리프 8대. 64×400Gbps × 리프 8대면 최대 512 GPU. 이 규모는 **스파인 계층 없이도** 운영 가능.

---

# 6. 운영 현장 노트

## 6.1 ConnectX-7 — IB / Ethernet 모드 설정

ConnectX-7은 IB·Ethernet 모두 지원 → 모드를 명시 설정.

```bash
mlxconfig -d <device> set LINK_TYPE_P1=<1|2> LINK_TYPE_P2=<1|2>   # 1=IB, 2=ETH, 적용 후 reboot
```

## 6.2 Optic는 조용히 죽어간다 — Pre-FEC BER

광링크는 갑자기 죽지 않고 서서히 열화한다. **Pre-FEC BER**(FEC 실행 전 비트 오류율)이 올라가면 `FEC 보정 한계 → Post-FEC 폭증 → link down → NCCL timeout`.

```bash
show interfaces all phy detail | egrep "Eth|uncorrected|Pre-FEC"
```

| 원인 | 조치 |
| --- | --- |
| MPO 커넥터 오염 | 청소 |
| 레이저 열화 | 교체 |
| 광케이블 굴곡 | 포설 재점검 |

대응 기준: 교체 일정 `≥10⁻⁶`, 즉시 교체 `≥10⁻⁴`.

## 6.3 PCIe ACS — GPU-NIC P2P를 막아 RDMA를 망친다

**ACS(Access Control Services)** 가 켜지면 GPU↔NIC P2P 직결이 막혀 메모리 복사가 강제되고 **RoCEv2/AllReduce 대역폭 30~50% 손실**. ACS는 VM 간 DMA 격리 보안 기능이라 가상화엔 필요하나 베어메탈 학습엔 방해.

```bash
lspci -vvv | grep -i ACS      # 상태 확인
nvidia-smi topo -m            # GPU-NIC P2P 경로 확인
```

해결: BIOS `ACS Override`/`PCIe ACS Support`=Disabled, 커널 `pcie_acs_override=downstream,multifunction`, IOMMU `iommu=pt`(SR-IOV 시 별도 검토).

## 6.4 ZTR / RTTCC — 스위치 설정 없는 혼잡 제어

**NVIDIA ZTRCC(Zero Touch RoCE Congestion Control)**: 스위치 설정 없이 NIC에서 RTT를 실시간 분석해 대규모 GPU 클러스터의 안정 성능·저지연 유지(RoCEv2 PFC/ECN 튜닝 부담 경감).

---
---

# 📌 추가 고려사항 — H100·H200을 PCIe 타입으로 구성할 때 (SXM vs PCIe)

> 대규모 학습 클러스터는 SXM/NVSwitch가 정석이지만, **테스트·개발 환경이나 작은 규모로 시작하는 조직**은 비용·전력·유연성 때문에 **PCIe 타입**을 선택지로 고려한다.
> 특히 **B200(Blackwell) 이전 세대인 H100·H200을 PCIe 타입으로 구성**했을 때 짚어야 할 점을 정리한다 — SXM과의 폼팩터 차이, PCIe 스위치의 역할, 그리고 PCIe에서 두드러지는 **CPU 소켓 간(UPI/xGMI) 통신 병목**.
> (GB200/B200 세대는 NVL72처럼 랙 단위 SXM·NVLink가 기본이라, PCIe 폼팩터 고려는 주로 H100·H200 세대에 해당한다.)

## 추가-1. SXM vs PCIe 폼팩터

| 항목 | **SXM** (HGX/DGX 베이스보드) | **PCIe** (표준 카드) |
| --- | --- | --- |
| 장착 | 전용 소켓(HGX baseboard) | 일반 PCIe 슬롯 |
| GPU 간 NVLink | **NVSwitch 기반 8-way all-to-all 풀메시** | **NVLink Bridge로 2-GPU 쌍(pair)만** |
| 쌍을 넘는 통신 | 풀 NVLink(균일 고대역) | **PCIe로 fallback** + 최악 시 소켓 간 링크 |
| NVSwitch | 있음 | **없음** |
| H100 NVLink | 18 links × 25 GB/s/방향 × 2방향 = **900 GB/s(총 대역폭)**, "PCIe Gen5의 7배" | 브리지 쌍당 **600 GB/s**(2-way) |
| H100 TDP | **700 W** | **350 W** |
| 8-GPU 대역폭 | HGX H100 = **집계 7.2 TB/s**(8×900GB/s) / **bisection 3.6 TB/s**, 4× 3rd-gen NVSwitch | 해당 없음(쌍 단위) |
| 멀티노드 NVLink | NVLink Switch System 256 GPU / 57.6 TB/s ⚠ | 없음(노드 간은 NIC/RDMA) |
| 주 용도 | **대규모 학습**, TP 무거운 워크로드 | **추론·파인튜닝, 테스트·개발, 소규모 시작**(H100/H200 PCIe). TP는 NVLink 브리지로 묶은 2-GPU 쌍이 한도 |

- 참고 세대값: A100 SXM = 12 links × 50 GB/s = **600 GB/s**; H200 NVL(PCIe) = 2-way 브리지 900 GB/s, 4-way 최대 1.8 TB/s, TDP 600W; GB200(NVLink5) = GPU당 1.8 TB/s, NVL72 = 130 TB/s.


## 추가-2. PCIe 세대별 대역폭 (NVLink 대비)

| 링크 | x16 대역폭 | 출처 |
| --- | --- | --- |
| PCIe Gen4 x16 | 64 GB/s 합계 (방향당 32) | |
| PCIe Gen5 x16 | 128 GB/s 합계 (방향당 **64**) | |
| PCIe Gen6 x16 | Gen5의 2배 = 방향당 ~128 (PCI-SIG 사양) | |
| H100 NVLink | 900 GB/s — **PCIe Gen5의 7배** | |


## 추가-3. PCIe 스위치 (Broadcom PEX 등)

PCIe 스위치는 **같은 스위치 하위의 GPU↔GPU·GPU↔NIC가 CPU 루트 컴플렉스를 거치지 않고 P2P DMA**를 하게 해준다 — CPU 병목을 우회하고 GPUDirect RDMA를 살린다. NVIDIA도 "경로에 PCIe 스위치만 있을 때가 최적"이라고 명시한다. (Broadcom PEX 8747 = 48-lane 5-port Gen3 스위치, GPU P2P 용도)

### `nvidia-smi topo -m` 경로 등급 (가까울수록 좋음)

| 등급 | 의미 | P2P 품질 |
| --- | --- | --- |
| `NV#` | NVLink # 개 묶음 경유 | 최상 |
| `PIX` | **단일 PCIe 브리지(같은 스위치)** | P2P 최적 |
| `PXB` | 다중 PCIe 브리지(Host Bridge 미경유) | 양호 |
| `PHB` | PCIe + **Host Bridge(보통 CPU)** 경유 | 저하 시작 |
| `NODE` | 같은 NUMA 노드 내 Host Bridge 간 | 저하 |
| `SYS` | **NUMA 노드(소켓) 간 SMP 인터커넥트(QPI/UPI 또는 xGMI) 경유** | 최악(제한·불가) |

## 추가-4. CPU 소켓 간(cross-socket) 병목

듀얼 소켓 서버에서 **GPU와 그 NIC(또는 두 GPU)가 서로 다른 CPU 소켓**에 붙으면, 트래픽이 소켓 간 링크를 건너야 한다:

- **Intel UPI(Ultra Path Interconnect)**: Sapphire Rapids 기준 **16 GT/s, 소켓당 최대 4 링크**. (per-link GB/s는 Intel 미공개 ⚠ — 임의 환산 금지.)
- **AMD Infinity Fabric / xGMI**: 4th Gen EPYC(Genoa) 기준 **최대 4 xGMI 링크 × 32 Gbps, 각 16 PCIe lane**. (집계 GB/s는 AMD 미공개 ⚠.)

이 소켓 간 링크는 **NVLink는 물론 로컬 PCIe Gen5 x16보다도 느리고 지연이 크다.** 그래서:

- **GPUDirect P2P/RDMA는 두 장치가 같은 PCIe 루트 컴플렉스를 공유해야** 하며, **QPI/UPI를 건너는 경로(=`SYS`)는 "성능이 극히 제한되거나 아예 동작하지 않을 수 있다"**. 이 경우 트래픽이 **host memory를 경유(bounce)** 한다.
- 따라서 **GPU·NIC·메모리를 같은 소켓/NUMA 노드에 고정(pin)** 해야 한다(NUMA locality).
- 6.3절의 **PCIe ACS**도 같은 맥락 — ACS가 켜지면 P2P가 루트 컴플렉스까지 올라가 스위치 P2P 이점을 무력화하므로 **비활성화** 권장.

```
[나쁜 배치: SYS 경로]                       [좋은 배치: PIX 경로]
GPU(소켓0) ─PCIe─ CPU0 ═UPI/xGMI═ CPU1 ─PCIe─ NIC(소켓1)   GPU ─┐
   → 소켓 간 링크 + host bounce, P2P 불가/저하              PCIe switch ├─ NIC
                                                          → CPU 우회 P2P, GPUDirect RDMA 직결
```

## 추가-5. 실무 배치 가이드 — GPU Card PCIe 구성시 어떻게 구성할까

- **GPU와 그 NIC를 같은 PCIe 스위치 하위(PIX)에 배치**해, GPUDirect RDMA가 CPU 루트 컴플렉스도 소켓 간 링크도 건너지 않게 한다. H100/H200 PCIe의 표준 NIC는 ConnectX-7(NDR 400G, PCIe Gen5 x16)이다.
- **GPU·NIC·메모리를 같은 CPU 소켓/NUMA 노드에 고정(pin)** 한다 — 듀얼 소켓이면 GPU·NIC를 소켓별로 반씩 나눠 각 소켓 안에서 GPU↔NIC 경로를 닫는다.
- **PCIe ACS는 끈다**(6.3절) — 켜져 있으면 같은 스위치 P2P도 루트 컴플렉스로 끌려 올라가 무력화된다.
- **TP가 필요한 GPU 쌍은 NVLink 브리지로 묶는다** — H200 NVL은 2-way 브리지 900 GB/s, 4-way 최대 1.8 TB/s를 제공한다. 쌍을 넘는 TP는 PCIe로 떨어지므로 피한다.
- 배치 결과는 `nvidia-smi topo -m`으로 검증한다 — GPU↔NIC가 `PIX`/`PXB`면 양호, `PHB`/`NODE`/`SYS`면 재배치 대상이다.

**SXM이 이 고민 대부분을 없애는 이유**: HGX/DGX의 NVSwitch 메시가 GPU-GPU 900 GB/s all-to-all을 **PCIe 트리·CPU 소켓과 무관하게** 제공하므로, 소켓 간 PCIe가 GPU-GPU 경로에 끼지 않는다. PCIe 서버는 NVSwitch가 없고 쌍 단위 NVLink 브리지뿐이라, **모든 쌍 외 통신·GPU↔NIC 전송이 PCIe 스위치 + NUMA 배치에 의존**한다 → 잘못 배치하면 `SYS` 경로로 떨어진다.

### Reference Links

| URL | 비고 |
| --- | --- |
| https://developer.nvidia.com/blog/nvidia-hopper-architecture-in-depth/ | H100 NVLink 900GB/s·TDP·PCIe Gen4/5 GB/s (2022, ⚠ >12개월) |
| https://developer.nvidia.com/blog/introducing-nvidia-hgx-h100-an-accelerated-server-platform-for-ai-and-high-performance-computing/ | HGX H100 4× NVSwitch non-blocking (2022) |
| https://www.nvidia.com/en-us/data-center/nvlink/ | 세대별 all-to-all(7.2/130 TB/s) |
| https://developer.nvidia.com/blog/introducing-hgx-a100-most-powerful-accelerated-server-platform-for-ai-hpc/ | A100 600GB/s·12 links (2020) |
| https://www.nvidia.com/content/dam/en-zz/Solutions/gtcs22/data-center/h100/PB-11133-001_v01.pdf | H100 PCIe NVLink bridge 600GB/s (⚠ PDF, 스니펫 확인) |
| https://developer.nvidia.com/blog/deploying-nvidia-h200-nvl-at-scale-with-new-enterprise-reference-architecture/ | H200 NVL 2-way/4-way, 2:1 GPU:NIC |
| https://www.nvidia.com/en-us/data-center/gb200-nvl72/ | GB200 NVL72 |
| https://developer.nvidia.com/blog/nvidia-gb200-nvl72-delivers-trillion-parameter-llm-training-and-real-time-inference/ | NVLink5 1.8TB/s, 130TB/s |
| https://developer.nvidia.com/blog/nvidia-connectx-8-supernics-advance-ai-platform-architecture-with-pcie-gen6-connectivity/ | PCIe Gen6, 2:1, 50GB/s/GPU IO |
| https://www.broadcom.com/products/pcie-switches-retimers/pcie-switches/pex8747 | PEX 8747 48-lane Gen3 |
| https://docs.broadcom.com/doc/12351854 | PEX P2P 용도 (⚠ PDF 미추출) |
| https://docs.nvidia.com/cuda/gpudirect-rdma/index.html | 같은 루트 컴플렉스 요건, QPI/UPI 경로 제한 |
| https://docs.nvidia.com/gpudirect-storage/configuration-guide/index.html | topo 범례(PIX~SYS), ACS 비활성 권장 |
| https://www.intel.com/content/www/us/en/developer/articles/technical/fourth-generation-xeon-scalable-family-overview.html | UPI 2.0 16GT/s·4 links (⚠ per-link GB/s 미공개) |
| https://www.amd.com/content/dam/amd/en/documents/products/epyc/4th-gen-epyc-processor-architecture-white-paper.pdf | xGMI 4 links×32Gbps (⚠ 집계 GB/s 미공개) |

