# GPU 인터커넥트 · NCCL · NUMA · InfiniBand/RoCEv2 · 혼잡제어 · 클러스터 설계

> 칩(다이)에서 랙까지, GPU를 잇는 인터커넥트 계층과 그 위에서 도는 집합통신(NCCL)·RDMA·혼잡제어, 그리고 이를 담는 GPU 클러스터 네트워크 설계를 정리한다.

---

# 1. GPU 인터커넥트 계층 (다이 → 랙)

GPU를 잇는 길은 거리에 따라 종류가 다르다. **패키지 안 → 패키지 간 → 노드 안 → 랙 안 → 랙 간** 순으로 인터커넥트가 바뀐다.

## 1.1 인터커넥트 4형제 + PCIe

| 인터커넥트 | 잇는 대상 | 특징 |
| --- | --- | --- |
| **NV-HBI** (High-Bandwidth Interface) | 한 패키지 안 GPU 다이 2개 | Blackwell은 다이 2개를 ~10 TB/s로 이어 OS에는 단일 GPU로 보이게 함 |
| **NVLink-C2C** (Chip-to-Chip) | 같은 패키지 내 **이종 칩**(CPU↔GPU) | Grace↔Blackwell 900 GB/s, cache-coherent 공유 메모리(복사 없이 상호 접근) |
| **NVLink** | GPU ↔ GPU | CPU 우회 직접 통신. 위 두 기술의 뿌리 |
| **NVSwitch** | 여러 GPU의 NVLink 포트 | "교차로" — All-to-All 패브릭 |
| **PCIe** | CPU ↔ 주변장치(GPU/NIC/SSD) | 업계 표준 버스. 범용·호환성↑이나 AI 데이터량엔 좁음 |

핵심: **NVLink는 GPU끼리의 지름길, PCIe는 시스템 전체의 표준 도로**다. 동세대 기준 대역폭이 약 14배 차이 난다(PCIe Gen5 x16 ~128 GB/s vs NVLink 5 1.8 TB/s). PCIe가 사라진 것은 아니며 CPU↔GPU 표준 경로와 NIC 같은 I/O 장치 연결 통로로 여전히 쓰인다.

## 1.2 레인(lane) vs 레일(rail)

- **레인(lane)**: 신호가 흐르는 전선 한 가닥(차동 한 쌍) 수준의 최소 물리 단위. "PAM4 레인당 200 Gb/s"의 그 레인이다. 레인 여러 개 → 1개 링크, 링크 18개 → GPU당 1.8 TB/s.
- **레일(rail)**: 훨씬 위층, 랙 간 네트워크의 통로 개념. rail-optimized 토폴로지에서 각 노드의 0번 GPU를 전부 '레일 0' 스위치로, 1번 GPU를 '레일 1'로 묶어 집합통신(AllReduce 등)이 충돌 없이 흐르게 한다.

## 1.3 GPU 세대별 진화 (A100 → B300)

| 세대 | 메모리 | 대역폭 | 정밀도 | 다이 | NVLink |
| --- | --- | --- | --- | --- | --- |
| **A100** (Ampere, 2020) | 80GB HBM2e | ~2 TB/s | FP16/BF16 | 단일 | 3세대 600 GB/s |
| **H100** (Hopper, 2022) | 80GB HBM3 | 3.35 TB/s | **FP8**(Transformer Engine) | 단일 | 4세대 900 GB/s |
| **H200** (Hopper, 2024) | **141GB** HBM3e | ~4.8 TB/s | FP8 | 단일 | 4세대 900 GB/s |
| **B200** (Blackwell, 2024-25) | 192GB HBM3e | ~8 TB/s | **FP4** | **듀얼**(NV-HBI) | 5세대 1.8 TB/s |
| **B300** (Blackwell Ultra, 2025) | 288GB HBM3e | ~8 TB/s | FP4 강화 | 듀얼 | 5세대 1.8 TB/s |

방향성: A100→H100은 **FP8**(연산 처리량), H100→H200은 **메모리만** 키움(긴 컨텍스트·KV 캐시 추론 겨냥), H200→Blackwell은 **FP4 + 듀얼 다이**(추론 밀도).

## 1.4 NVLink 세대와 물리 계층

| 세대 | 구성 | GPU당 |
| --- | --- | --- |
| 3세대 (A100) | 12 링크 × 50 GB/s | 600 GB/s |
| 4세대 (H100/H200) | 18 링크 × 50 GB/s | 900 GB/s |
| 5세대 (B200/B300) | 18 링크 × **100 GB/s** | **1.8 TB/s** |

5세대는 링크 수(18)를 그대로 두고 **링크당 속도만 두 배**로 올렸다. 링크당 속도는 물리 계층의 **SerDes + PAM4**가 끌어낸다(데이터시트의 "224G PAM4").

## 1.5 NVSwitch 세대와 NVL72

- 2세대 (DGX A100): 8 GPU를 6칩으로 완전 연결
- 3세대 (HGX H100/H200): 시스템당 4칩, 8 GPU bisection 3.6 TB/s, **SHARP**(in-network compute) 도입
- 4세대 (Blackwell): **랙 레벨 스위칭** — NVSwitch를 서버 밖 전용 트레이로 빼내 랙 전체를 하나의 NVLink 도메인으로

**NVL72**: NVLink Switch 칩이 랙 안 72개 GPU를 묶어 **130 TB/s All-to-All**, GPU당 1.8 TB/s를 유지한 채 모든 GPU와 동시 통신하는 논블로킹 구조 → 72 GPU가 사실상 하나의 거대 GPU처럼 동작.

실무 확인: `nvidia-smi nvlink -s`(링크 속도), `nvidia-smi topo -m`(토폴로지 — NVx=NVLink, SYS=PCIe).

## 1.6 C2C vs NVLink-C2C (용어 구분)

- **C2C** (Chip-to-Chip / Chiplet): 업계 일반 명사. 단일 거대 칩 대신 여러 칩렛을 초고속 인터페이스로 이어 붙이는 패키징. B200 듀얼다이는 "넓은 의미의 C2C"가 맞다.
- **NVLink-C2C**: NVIDIA 고유 기술명. 서로 다른 독립 패키지(CPU↔GPU)를 보드/모듈에서 직접 연결, **unified address space**(하드웨어 코히어런트 메모리). 대표 예 GB200(Grace 1 ↔ Blackwell 2, 900 GB/s).
- 따라서 B200 듀얼다이는 NVLink-C2C가 **아니다**. (GH200의 C2C는 신호당 40 Gbps × 9 데이터신호, 10 링크 → 방향당 450 GB/s, Arm AMBA CHI 지원)

---

# 2. CUDA와 NCCL

## 2.1 CUDA

CUDA(Compute Unified Device Architecture)는 GPU 가속 애플리케이션 개발 환경이다. GPU 가속 라이브러리, 디버깅·최적화 도구, C/C++ 컴파일러, 런타임 라이브러리를 포함한다.

## 2.2 NCCL 개요

NCCL(NVIDIA Collective Communication Library)은 멀티 GPU·멀티 노드 통신 기본요소를 구현한다. all-gather / all-reduce / broadcast / reduce / reduce-scatter / point-to-point를 제공하며, **PCIe·NVLink·NVSwitch·InfiniBand·RoCE에 걸쳐 자동 토폴로지 감지**로 최적 경로를 고른다. 통신과 연산을 단일 커널로 구현해 낮은 지연을 보장한다.

- 동작 확인·튜닝: `NCCL_DEBUG`(로깅), `NCCL_ALGO`/`NCCL_PROTO`/`NCCL_NTHREADS`(알고리즘·프로토콜·스레드).
- 프로파일링: **Nsight Systems**(CPU↔GPU·시스템 전체 타임라인), **Nsight Compute**(CUDA 커널 내부, memory-bound vs compute-bound).

## 2.3 집합통신 기본 4패턴과 8종 Collective

| 패턴 | 방향 | 데이터 | 용도 |
| --- | --- | --- | --- |
| Broadcast | root → all | 같은 값 복제 | 초기 weight 동기화 |
| Scatter | root → all | 서로 다른 chunk | 배치 분배 |
| Gather | all → root | concat | 결과 수집 |
| Reduce | all → root | element-wise op | loss/gradient 집계 |

조합: **AllGather** = Gather+Broadcast, **AllReduce** = Reduce+Broadcast(또는 ReduceScatter+AllGather), **ReduceScatter** = Reduce+Scatter, **AlltoAll** = Scatter 전치.

| 함수 | ML 용도 |
| --- | --- |
| `ncclAllReduce` | DDP gradient sync |
| `ncclBroadcast` | init param sync |
| `ncclReduce` | norm 집계 |
| `ncclAllGather` | ZeRO-3 / FSDP param |
| `ncclReduceScatter` | FSDP gradient |
| `ncclGather` / `ncclScatter` | 결과 취합 / 배치 분배 |
| `ncclAlltoAll` | **MoE token dispatch** |

## 2.4 NCCL 알고리즘 — 같은 Collective, 다른 Schedule

같은 `AllReduce`라도 메시지 크기·rank 수·토폴로지(NVSwitch 유무)에 따라 ring·tree·NVLS가 달리 골라진다. NCCL은 호출마다 **알고리즘 7 × 프로토콜 3 = 21칸 표**를 만들고 각 칸의 예상 실행 시간을 cost model로 계산해 **가장 짧은 칸을 고른다(argmin)**. eligibility 조건이 대부분 칸을 미리 제거한다.

- **Algorithm 7종**: `Ring`(거의 모든 collective), `Tree`(Double Binary, AllReduce, tree 지연 + ring 대역폭), `CollNetDirect`/`CollNetChain`(SHARP, IB SHARP NIC), `NVLS`(NVSwitch multicast, Hopper+), `NVLSTree`(2노드+), `PAT`(Bruck, AllGather/ReduceScatter)
- **Topology Pattern**: BALANCED_TREE / SPLIT_TREE / TREE / RING / NVLS / COLLNET_DIRECT — BALANCED/SPLIT은 NIC 트래픽을 두 GPU에 나눠 PCIe/NVLink 병목을 푼다.
- **Protocol 3종**:

| Protocol | 효율 | 적합 |
| --- | --- | --- |
| `LL` (8B: 4B data+4B flag) | 50% | 짧은 메시지, latency |
| `LL128` (128B: 120B+8B) | 93.75% | NVLink intra-node 중간 메시지 |
| `Simple` | ~100% | 큰 메시지, throughput |

LL/LL128은 data 옆에 flag를 같이 보내 receiver가 단일 word load로 ready 폴링(PCIe doorbell 불필요), 대가는 효율 손실. LL128은 NVLink cache line(128B)을 그대로 활용해 sweet spot.

## 2.5 데이터 경로 — Intra-node vs Inter-node

**Intra-node 우선순위**: ① P2P over NVLink → ② P2P over PCIe → ③ SHM(host memory 경유) → ④ NIC loopback.

`NCCL_P2P_LEVEL`로 P2P 사용 최대 거리를 제어한다: `LOC`(비활성) / `NVL`(NVLink) / `PIX`(같은 PCI 스위치) / `PXB`(여러 PCI 스위치) / `PHB`(같은 NUMA, CPU 경유) / `SYS`(NUMA 간, QPI/UPI 경유).

`nvidia-smi topo -m` 경로 등급(가까울수록 좋음):

| 등급 | 의미 | P2P 품질 |
| --- | --- | --- |
| `NV#` | NVLink # 개 묶음 | 최상 |
| `PIX` | 단일 PCIe 브리지(같은 스위치) | 최적 |
| `PXB` | 다중 PCIe 브리지(Host Bridge 미경유) | 양호 |
| `PHB` | PCIe + Host Bridge(보통 CPU) | 저하 시작 |
| `NODE` | 같은 NUMA 노드 내 Host Bridge 간 | 저하 |
| `SYS` | NUMA 노드(소켓) 간 SMP 인터커넥트(QPI/UPI/xGMI) | 최악 |

**Inter-node**:

```
GPU kernel → GPU vidmem → (CPU proxy thread: ncclProxyProgress) → NIC → wire → ...
                              └→ RDMA write (IB/RoCE) 또는 socket send
```

CPU가 데이터 자체를 만지지는 않지만 NIC 작업 오케스트레이션은 host 스레드의 몫이다. **GPUDirect RDMA 가능**(NIC와 GPU가 같은 PCIe switch, 기본 `PATH_PXB`)이면 NIC가 GPU 메모리를 직접 read/write(`ncclTopoCheckGdr` 결정, `NCCL_NET_GDR_LEVEL` override). 불가능하면 host pinned memory에 staging(PCIe를 두 번 더 건넘). 전송 앞에 양쪽 버퍼 준비를 합의하는 **rendezvous**.

---

# 3. NUMA와 토폴로지 인식

## 3.1 NUMA 배경 (SMP → NUMA)

과거 UMA/SMP 구조는 모든 CPU가 **중앙 메모리 버스**를 공유해, CPU가 늘수록 버스가 병목이 됐다. **NUMA**(Non-Uniform Memory Access)는 CPU마다 **로컬 메모리**를 두고, 다른 CPU의 메모리는 **원격 메모리**로 분류하되 접근은 가능하게 한다(원격은 느림). 중앙 버스 병목이 사라지고 CPU 확장성이 좋아진다.

- 비일관적 메모리 접근: 가까운 로컬 메모리를 우선 사용해 성능↑.
- Point-to-Point 인터커넥트: Intel QPI/UPI, AMD HyperTransport/Infinity Fabric.
- 소켓 2개 이상 = NUMA Node 2개 이상. NUMA 자체는 캐시 일관성을 보장하지 않아 **ccNUMA**(Cache-Coherent NUMA)가 등장(Intel/AMD 모두 지원). Intel SNC(sub-NUMA cluster)로 die 내부도 세분.

## 3.2 CPU 소켓 간(cross-socket) 병목

GPU와 그 NIC(또는 두 GPU)가 서로 다른 소켓에 붙으면 트래픽이 소켓 간 링크를 건넌다.

- **Intel UPI**: Sapphire Rapids 기준 16 GT/s, 소켓당 최대 4 링크.
- **AMD Infinity Fabric / xGMI**: 4th Gen EPYC 기준 최대 4 xGMI 링크 × 32 Gbps, 각 16 lane.

이 소켓 간 링크는 NVLink는 물론 로컬 PCIe Gen5 x16보다도 느리고 지연이 크다. **GPUDirect P2P/RDMA는 두 장치가 같은 PCIe 루트 컴플렉스를 공유해야** 하며, `SYS` 경로(QPI/UPI 횡단)는 성능이 극히 제한되거나 동작하지 않을 수 있어 트래픽이 host memory를 경유(bounce)한다. → **GPU·NIC·메모리를 같은 소켓/NUMA 노드에 고정(pin)**, PCIe ACS off.

## 3.3 NUMA 인식 스케줄링과 설정

- **OS 기본 정책(Linux first-touch)**: 메모리 페이지에 **처음 접근한 CPU**가 속한 NUMA Node에 물리 페이지가 할당된다.
- **NUMA-Aware Application**: 목표는 **CPU·Memory·I/O를 같은 NUMA Node에 묶기**. Thread를 특정 소켓에 고정, 메모리도 같은 Node에 할당, NIC·NVMe·GPU도 같은 Node에 배치.
- **Kubernetes**: 자원을 같은 NUMA Node에 모으도록 설정.

```yaml
cpuManagerPolicy: static
topologyManagerPolicy: single-numa-node
memoryManagerPolicy: Static
```

- **NVIDIA CDMM**: GPU 메모리를 소프트웨어 NUMA 노드로 OS에 노출하지 않고 드라이버가 직접 관리하는 대체 모드(PCIe 연결 GPU 모델에서 영감).

## 3.4 단일 호스트 Multi-GPU 토폴로지

"GPU 개수"보다 "**GPU들이 어떤 경로로 연결되어 있는지**"가 scaling 효율을 좌우한다. 분산 모델은 작은 단계의 collective를 반복하며 **각 단계는 가장 느린 GPU 간 경로를 기다린다**.

균일성·예측성이 높아지는 순서:

```
PCIe 독립 < NVLink pair(2) < 4-way NVLink domain < dual NVLink domain < HGX NVSwitch fabric
유연·병목 큼 ───────────────────────────────────────────────► 균일·예측성 높음
```

| 구성 | Interconnect | Point-to-Point BW |
| --- | --- | --- |
| RTX PRO 6000 (PCIe Gen5) | PCIe | 128 GB/s |
| H100 NVL (2-card bridge) | NVLink | 600 GB/s |
| H200 NVL (2-way / 4-way bridge) | NVLink | 900 GB/s / 1.8 TB/s aggregate(900 per GPU) |
| HGX H100/H200 (SXM+NVSwitch) | NVSwitch | 900 GB/s per GPU |
| HGX B200 (SXM+NVSwitch) | NVSwitch | 1.8 TB/s per GPU |

- **NVLink domain**: NVLink로 직접 연결되어 domain 밖으로 안 나가는 GPU 그룹. 도메인 밖 통신은 PCIe 경로(최악 시 `두 PCIe hop + UPI crossing`).
- **NVIDIA Fabric Manager**: NVSwitch 기반 HGX에서 부팅 시 topology 탐지, routing table 구성, GPU partition을 OS에 노출. 8-GPU 시스템은 보통 4-GPU partition 두 개.
- 8-GPU 서버라도 **두 개의 4-GPU island**일 수도, **하나의 NVSwitch fabric**일 수도 있다 — 병렬화(DP/TP/PP/MoE)에 맞춰 topology를 골라야 한다.

## 3.5 SHARP — 스위치가 대신 계산한다

3세대 NVSwitch는 임베디드 ALU(최대 400 GFLOPS FP32)를 넣어 **GPU 대신 reduce 연산을 스위치 내부에서** 수행한다(GPU offload).

All-Reduce 트래픽 비교:

| 단계 | A100 | H100 + NVLink SHARP |
| --- | --- | --- |
| Read & reduce | GPU가 받아서 직접 더함 | **NVSwitch가 내부에서 더하고** GPU는 결과만 받음 |
| Broadcast | GPU가 직접 write | GPU 1회 write → **NVSwitch가 multicast 복제** |
| Traffic | 2N send / 2N recv | **N+1 send / N+1 recv** |

→ NVSwitch가 단순 전달 장치에서 **collective 연산 가속 장치**로 진화. 유효 NVLink 대역폭 약 2배.

## 3.6 NVLink Network vs TCP/IP

NVLink Switch System은 GPU가 네트워크를 통해 전송할 때 쓰는 별도 주소 공간을 가져 격리·보안을 강화한다. 기존 네트워킹 개념과 대응:

| 계층 | 기존 | NVLink 네트워크 |
| --- | --- | --- |
| 네트워크 | IP | NVLink 주소 지정·관리 프로토콜 |
| 전송 | TCP | 온칩 HW/FW |
| 세션 | 소켓 | SHARP groups |
| RDMA offload | NIC 엔진 | GPU 내부 복제 엔진 |
| 집합 연산 offload | NIC/스위치 엔진 | NVSwitch 내장 SHARP |

## 3.7 JCT와 Tail Latency

- **JCT(Job Completion Time)**: 작업 시작~종료 시간. 보통 1작업이 여러 스레드로 나뉘어 여러 GPU에서 병렬 실행되는데, **가장 느린 스레드가 전체를 결정**한다.
- **Tail Latency(꼬리 지연)**: 높은 백분위(예 P99) 지연. 직렬·병렬 모두에 영향(병렬에서도 느린 1개를 모두가 기다림). 백만 작업 중 한 번의 저하도 대규모에서는 상시 발생한다. 낮은 JCT를 위해 패브릭은 무혼잡·효율적 로드밸런싱·낮은 tail latency가 필요하다.

---

# 4. InfiniBand · RDMA · RoCEv2

## 4.1 InfiniBand 개요

IB는 PCI 공유 버스의 대역폭·확장성·신뢰성 한계를 넘기 위한 **스위치 기반 P2P 패브릭**이다. 단순 케이블이 아니라 물리~전송~관리~소프트웨어를 아우르는 아키텍처다.

- 계층: Physical / Link / Network / Transport / Upper(Verbs).
- **Link Layer**: 최대 4KB 페이로드, **LID(16bit, Subnet Manager가 부여)** 로 로컬 식별, **VL(Virtual Lane)** 로 QoS(VL15=관리 최우선), **credit 기반 flow control**(태생적 무손실), VCRC(홉별)/ICRC(엔드투엔드).
- **Transport**: RC(신뢰·연결) / RD / UC / UD / Raw. 전송 계층이 하드웨어로 구현됨.
- 구성: **HCA**(호스트), **TCA**(I/O 장치), **Switch**(LID 포워딩), **Router**(서브넷 간, GRH의 IPv6), **Subnet Manager**(LID 할당·라우팅).
- VIA 모델: 클라이언트가 **WQE**를 큐에 넣고 HCA가 처리, 완료는 **CQE**. **QP**(Queue Pair) = Send Queue + Receive Queue (소켓의 종단점 역할).

## 4.2 RDMA가 빠른 이유

```
TCP:   APP → kernel buffer → kernel TCP/IP → NIC → wire → ...
RDMA:  APP → HCA → wire → HCA → APP        (커널 우회, 카피 없음)
```

세 가지가 결합한다: **Kernel bypass**(데이터 경로 커널 우회) + **Zero-copy**(사용자 버퍼에서 직접 NIC DMA) + **Transport offload**(헤더·재전송·ordering을 HCA 하드웨어가 처리). 단 **Control path는 커널 경유** — QP 생성·메모리 등록은 고비용 1회 작업, 이후 send/recv/read/write는 매우 빠르다.

- **Two-sided (Send/Recv)**: 양쪽 active, 수신자가 미리 receive를 post해야 함(없으면 `RNR_RETRY_EXC_ERR`).
- **One-sided (RDMA Read/Write)**: 원격 CPU가 인지하지 못한 채 HCA가 rkey로 권한 검증 후 등록된 메모리에 직접 DMA. **RDMA의 핵심 강점**.

| 연산 | requester | responder CPU | 용도 |
| --- | --- | --- | --- |
| Send/Recv | 양쪽 active | 개입 O | 제어 메시지 |
| RDMA Write | 한쪽만 | 개입 X | 일방 전달 (NCCL 기본) |
| RDMA Read | 한쪽만 | 개입 X | pull (KV cache 회수) |
| Atomic | 한쪽만 | 개입 X | 분산 락, counter |

객체 모델: `Context → PD(Protection Domain) → {MR, QP(SQ/RQ)}`, `CQ`. WR(Work Request), WC(Work Completion), SGE(Scatter-Gather). Transport는 대부분 **RC**(Read/Atomic이 RC에서만 동작). QP 상태: `RESET → INIT → RTR(수신 가능) → RTS(송신 가능)` — RTR 전이에 상대 QPN과 LID/GID 필요.

## 4.3 InfiniBand vs RoCEv2 — 같은 코드, 다른 wire

RDMA의 "위쪽 절반"(Verbs API, QP, MR, RC/UC/UD)은 IB·RoCEv2 공통이다. 차이는 **네트워크 계층(wire 전송)** 에서만 발생한다.

```
Application + libibverbs API        ← 동일
RDMA Transport (RC, UC, UD ...)     ← 동일
Network Layer                       ← 분기
  IB:     IB Network (LID 라우팅)
  RoCEv2: UDP/IP (port 4791)        ← 표준
Link: IB cable / Ethernet cable
```

- **주소**: IB는 LID(16bit, Subnet Manager). RoCEv2는 **GID**(IPv4/IPv6를 IPv6 형식으로 인코딩) 사용. `show_gids`로 RoCEv2 GID 인덱스 확인(잘못 고르면 RoCEv1로 동작).
- **코드 차이**: `ah_attr.is_global` 한 플래그 — IB는 0(LID로 dlid), RoCEv2는 1(GRH 필수, dgid≈IP).
- **관리**: IB는 Subnet Manager(`opensm`). RoCEv2는 이더넷 스위치 설정 중요(SM 불필요).

### RoCEv2의 핵심 과제: Lossless Ethernet

IB는 link layer가 **credit-based flow control**이라 drop이 거의 없다. RoCEv2는 일반 이더넷(기본 lossy) 위에서 돌고, RC 재전송은 **go-back-N**이라 패킷 하나 손실에 이후 전부 재전송 → 이더넷을 lossless에 가깝게 만들어야 한다.

- **PFC**(802.1Qbb): 큐별 PAUSE. RDMA를 특정 priority에 격리. 위험: head-of-line blocking, PFC storm, deadlock.
- **ECN + DCQCN**: 스위치가 drop 없이 ECN 마크 → 수신 NIC가 CNP → 송신 NIC rate 감소. PFC보다 부드럽고 현재 대규모 RoCE 표준.
- 실무: **DCQCN 메인 + PFC 백업**. MTU는 Jumbo 9000 + RoCE 4096. RoCEv2는 UDP/IP 위라 **L3 라우팅·ECMP 가능**(UDP source port entropy로 multipath). 같은 코드가 IB는 line rate, RoCEv2는 절반이면 대개 PFC/ECN/DSCP 설정 문제.

## 4.4 RoCEv2 패킷 구조와 IB BTH

```
InfiniBand:  [LRH][GRH opt][BTH][Extended Transport][Payload][ICRC][VCRC]
RoCEv2:      [Ethernet][IP][UDP][BTH][Extended Transport][Payload][ICRC]
```

RoCEv2는 Ethernet/IP/UDP(목적지 포트 **4791**) 위에 **IB transport 헤더(BTH)** 를 올린다. **IB BTH(Base Transport Header)** 주요 필드:

| 필드 | 의미 |
| --- | --- |
| **OpCode** | RDMA operation 종류(Send/Write/Read/ACK/CNP…) |
| **P_Key** | Partition Key (IB 논리 격리) |
| **Destination QP** | 목적지 Queue Pair 번호 |
| **PSN** | Packet Sequence Number (신뢰성·순서) |

RoCEv2는 UDP라 신뢰성을 **PSN**으로 확보한다. 세션은 3-way handshake로 QP·seq 교환 → 메모리 정보 교환 → 데이터 → 종료. ACK는 AETH(MSN으로 누락 감지).

## 4.5 RDMA가 "커널 우회" — 기존 커널 기능은 누가?

데이터 경로만 우회하고, 제어 경로(드라이버·등록·pinning·PD·QP·CQ·MR)는 커널이 맡는다. 기존 커널 기능 분산:

| 기존 커널 기능 | RDMA에서 |
| --- | --- |
| 소켓 송수신 | QP + Work Request |
| 커널 버퍼 복사 | Memory Registration + DMA |
| TCP segmentation | RNIC/HCA |
| ACK/retransmission | IB/RoCE transport |
| 흐름 제어 | credit / RNR / PFC/ECN |
| 혼잡 제어 | RoCEv2 ECN/DCQCN |
| 주소 변환·권한 | MR, lkey/rkey, Protection Domain |
| 완료 알림 | Completion Queue |

`ibv_reg_mr`이 ① 페이지 pin ② HCA 페이지 테이블에 가상→물리 매핑 ③ **rkey** 발급. 클라이언트가 RDMA Write에 rkey+remote_addr를 실으면 서버 HCA가 rkey 검증 후 CPU 인터럽트 없이 직접 DMA. (lkey=로컬 접근, rkey=원격 접근)

---

# 5. RoCEv2 혼잡 제어

## 5.1 혼잡은 어디서 생기나

전체 용량이 충분(무오버서브스크립션)해도 **여러 흐름이 같은 링크에 몰리면** 혼잡이 생긴다. 대표 위치: Local leaf / Leaf-spine / Spine-leaf / Leaf-server / Spine-superspine / Superspine-spine.

**Incast**: 여러 송신자가 하나의 수신자·출력 링크로 동시에 보내는 패턴. AI 학습(여러 GPU→파라미터/스토리지, AllReduce 동기화)에서 흔하다. AI 트래픽은 소수의 거대 흐름(**elephant flow**)이 많아 ECMP가 쏠리기 쉽다.

## 5.2 ECN — 표시하고 감속시키기

스위치가 큐 임계(ECN Threshold)를 넘으면 패킷을 버리지 않고 **CE 마킹** → 수신자가 송신자에게 **CNP(Congestion Notification Packet)** → 송신자가 해당 flow 감속.

```
스위치 혼잡 감지 → ECN 마킹 → 수신자 → CNP → 송신자 감속
```

IP DSCP/ECN 2비트(00 미지원 / 10 ECN 가능 / 11 혼잡 경험 / 01 CNP). CNP의 IB BTH opcode 129. 한계: 송신자에 알리기까지 **시간 지연** → 트래픽 급증 시 CNP 도착 전 큐가 차 drop 가능(ECN 큐 = Lossy Queue).

## 5.3 PFC — 멈추기

PFC(Priority Flow Control)는 혼잡 시 상위 장비에 **PAUSE/XOFF frame**(EtherType 0x8808)을 보내 특정 **Traffic Class 전체**를 일시 정지시킨다. XOFF(정지)/XON(재개) threshold로 제어.

- **HOL Blocking**: 같은 Priority Class의 정상 flow까지 멈춤.
- **PFC Storm**: Pause가 상위로 계속 전파되어 넓은 영역이 멈춤.
- **PFC Watchdog**: 포트별 과도한 Pause를 **감지(Detection) → 완화(Mitigation: Drop 또는 Forward) → 복구(Restoration)**. 일부 패킷 손실을 감수하더라도 Storm 확산을 끊는다.

## 5.4 DCQCN — ECN 먼저, PFC 나중

DCQCN(Data Center Quantized Congestion Notification)은 ECN을 1차 제어, PFC를 최후 방어로 결합한다. 설계 원칙: **ECN Threshold < PFC XOFF Threshold**.

```
큐 낮음 → ECN Threshold → ECN 마킹 → CNP → 송신자 감속 (부드러운 브레이크, end-to-end)
큐 높음 → PFC XOFF Threshold → PFC Pause → Class 일시 정지 (급브레이크, hop-by-hop)
```

ECN이 먼저 동작하면 PFC가 안 생겨 같은 큐의 다른 flow가 멈추지 않는다. **per-flow granularity**로 혼잡 확산 전에 감속. 스위치는 RED+ECN만 지원하면 되고 **핵심 계산·속도 조절은 NIC**가 한다.

| 단계 | 기술 | 영향 범위 |
| --- | --- | --- |
| 1차 | ECN / CNP / DCQCN | flow 중심 |
| 최후 | PFC | Priority Class 전체 (hop-by-hop) |
| 보호 | PFC Watchdog | 포트/큐 |

## 5.5 SFC — 소스 직접 제어 (차세대)

SFC(Source Flow Control, Source PFC)는 혼잡 스위치가 **송신자에게 직접 Flow 단위 신호**를 보낸다. 혼잡 패킷의 Source/Destination IP를 뒤집고 Payload를 Trim해 송신자로 되돌린다.

- ECN보다 빠르고(목적지 왕복 불필요), PFC보다 영향 작다(Class 전체가 아니라 Flow 단위).
- NIC가 SFC 미지원이면 Leaf가 PFC로 변환. NIC·Leaf·Spine 모두 SFC 이해 필요.
- 열린 질문: fallback 경로가 1-Hop 넘으면 IP 기반이라 반환 경로가 달라져 uni-directional이 될 수 있음.

## 5.6 CSIG — 풍부한 혼잡 정보 (IETF draft)

CSIG(Congestion Signaling)는 경로상 장비가 **병목 정보(용량/단계/Device ID/uplink·downlink)** 를 패킷의 **CSIG Tag(L2~L3 헤더 사이)** 에 in-band telemetry로 싣고, 수신자가 송신자에게 **reflection**한다. "혼잡 있음"을 넘어 "어디서·어느 링크가·얼마나" 병목인지 알려줘 **경로 선택까지 최적화** 가능. 단 draft 단계, ASIC/NOS/NIC 지원과 표준화 성숙도 필요.

| 기술 | 알려주는 정보 | 제어 | 한계 |
| --- | --- | --- | --- |
| ECN | 혼잡 여부 | 수신자→CNP→송신자 감속 | 단순·지연 |
| PFC | Class 정지 | hop-by-hop Pause | Class 전체·Storm |
| SFC | 혼잡 flow 직접 | 혼잡 스위치→송신자 | 장비/NIC 지원 |
| CSIG | 병목 위치·상태 | 경로 tag + reflection | 표준 성숙도 |

> 목표는 "loss 난 뒤 복구"가 아니라 "애초에 loss가 거의 안 나게": **PFC로 drop 막고, ECN/DCQCN으로 감속, ECMP/고급 LB로 경로 분산**. proactive(LB) 우선, reactive(ECN/PFC) 최소화.

---

# 6. GPU 클러스터 네트워크 설계

## 6.1 설계 목표

핵심은 **비싼 GPU가 네트워크 때문에 기다리지 않게**, 학습 job이 빨리 끝나게(JCT↓), 애초에 패킷 loss가 안 나게 만드는 것이다.

- **High-radix(다포트) 스위치**: 포트가 적으면 계층이 늘고(장비·케이블·latency·장애지점·비용↑). 포트 많은 스위치로 더 큰 클러스터를 더 단순한 토폴로지로.
- **Oversubscription 1:1**: 아래(서버측) 대역폭 = 위(패브릭측) 대역폭. 모든 서버가 동시에 line rate로 보내도 막히지 않는 **non-blocking / full bisection**.
- **Optimized design(Rail)**: GPU·NIC·PCIe·NUMA·NVLink·NCCL ring/tree·RDMA path·ToR 배치를 함께 고려. GPU0↔NIC0가 가까우면 GPU0 트래픽은 NIC0를 쓰게.

## 6.2 목적별 패브릭

| 패브릭 | 역할 |
| --- | --- |
| **Frontend / Inference** | 사용자·스케줄러·관리 제어 평면 (지연·손실 요구 낮음) |
| **Training / GPU Backend** | GPU 간 RoCEv2/IB 데이터 평면 (저지연·무손실 핵심) |
| **Storage Backend** | 데이터셋·체크포인트 I/O (저지연·무손실) |

GPU 서버 포트도 목적별로 분리: GPU training(RDMA backend), CPU/frontend, NVMe/storage. 한 네트워크에 섞으면 거대 training 트래픽이 다른 트래픽을 밀어내므로 패브릭 분리 또는 강한 QoS 적용.

---

# 7. Rail-Optimized Design (ROD)

## 7.1 Rail과 one-hop

AI 서버는 보통 GPU 8개 + NIC 8개로 GPU/NIC가 짝(GPU0↔NIC0 …)을 이룬다. 같은 번호끼리 묶은 외부 네트워크 경로가 **rail**이다. 8-GPU 서버 클러스터는 GPU 번호별 8개의 논리 레일로 나뉜다.

- 같은 rail leaf에 붙은 서버 간 통신은 **leaf 하나만 거치는 one-hop**(낮은 지연).
- 8개 leaf가 8개 GPU를 연결하는 묶음 = **row(stripe)**.

```
            Row / Stripe
 Leaf0  Leaf1  Leaf2  ...  Leaf7
   │      │      │           │
 GPU0   GPU1   GPU2        GPU7   (각 서버의 같은 번호 GPU가 같은 Leaf로)
```

## 7.2 1:1 oversubscription과 규모

64포트 400G 스위치를 서버측 32 + spine측 32로 쓰면 양쪽 12.8 Tbps로 **1:1**.

- **256 GPU** = 32 서버 × 8 GPU. 스위치당 32×400G 다운링크 × 8 leaf.
- **512 GPU** = 64 서버 × 8 GPU. 스위치당 64×400G 다운링크 × 8 leaf — 같은 rail 내 통신만이면 **스파인 없이** 운영 가능.

## 7.3 intra-rail vs inter-rail

| 구분 | intra-rail (GPU0↔GPU0) | inter-rail (GPU0↔GPU3) |
| --- | --- | --- |
| 경로 | Leaf 하나 | Leaf → Spine → Leaf |
| 지연 | 낮음 | 약 2배 |
| Spine | 불필요 | 필요 |

```
inter-rail: [Server1] Rail4:GPU4 → Leaf4 → Spine → Leaf5 → Rail5:GPU5 [ServerN]
```

## 7.4 확장 — 3-stage Clos와 chassis spine

- 8 leaf + 4 spine(각 64×400G) → 32 서버(256 GPU). 두 줄(16 leaf) + 8 spine → 64 서버(512 GPU).
- 더 키우면 spine 수 증가로 ECMP 경로·케이블·관리·장애 도메인이 복잡해진다. **chassis spine**(예 256×400G)을 쓰면 64포트 spine 대비 spine 수를 **1/4**로 줄인다(단가·장애영향·전력·벤더종속은 trade-off).

## 7.5 Rail-only Design

inter-rail 통신을 외부 패브릭이 아니라 **서버 내부 GPU switch(NVSwitch)** 가 처리하게 하고, 외부 패브릭은 같은 rail끼리만 연결한다.

- 장점: spine/super-spine 대역폭 절감, 토폴로지 단순화, **장애 격리**(Rail3 장애가 Rail0~2에 영향 적음), troubleshooting 용이.
- 전제: 워크로드가 inter-rail 외부 통신을 거의 안 해야. 필요하면 `GPU0 → 내부 NVSwitch → GPU3 → Rail3 외부` 우회.

---

# 8. Rail-Unified Design (RUD)

ROD가 "GPU별로 Leaf/Rail을 나눠 연결"이라면, RUD는 "**여러 GPU를 하나의 Leaf로 묶는**" 구조다(8개를 한 Leaf, 또는 4+4).

- 장점: 케이블 단순, 확장 용이, 익숙한 구조. ToR + 구리(DAC/AEC)에 특히 유리.
- 단점: 하나의 Leaf에 서버의 모든 GPU가 붙으면 그 **Leaf 장애 시 서버 전체가 외부에서 고립**.
- 같은 Leaf에 연결된 4서버는 **intra-rail·inter-rail 모두 one-hop**(가장 낮은 지연). 다른 Leaf 서버는 Spine 경유 two-hop.
- 하나의 Leaf가 여러 rail 트래픽을 다루므로 rail 구분을 위해 **deterministic path forwarding** 필요(Rail0→Spine0, Rail1→Spine1 … 처럼 예측 가능하게).
- 서버 내부 GPU 통신을 내부 NVSwitch 대신 외부 Leaf로 처리하려는 고객에게 매력적이나, 비용·성능 trade-off를 따져야 한다(NVSwitch는 GPU 간 900 GB/s 제공, 외부 스위치는 더 비쌈).

---

# 9. Rack 설계 (ToR / MoR / EoR)

Leaf 스위치를 랙의 어디에 두느냐에 따라 **전력·냉각·케이블 길이·비용·장애 도메인**이 달라진다.

- 42U 표준 랙. DGX H100 = 8U → 랙당 4~5대 + Leaf 1대, **48~60 kW**(일반 DC 20~25 kW의 2~3배). ROD는 서버 1대가 8개 Leaf에 연결되어 케이블이 많다(32서버 = 256 server-to-leaf 링크).

| 구분 | ToR (랙 상단) | MoR (행 중간) | EoR (행 끝) |
| --- | --- | --- | --- |
| Leaf 위치 | 각 서버 랙 상단 | 행 중간 별도 랙 | 행 끝 별도 랙 |
| 케이블 길이 | 가장 짧음 | 중간(5~6랙) | 가장 김(10랙) |
| 서버 랙 전력·냉각 | **증가** | 감소 | 감소 |
| 별도 네트워크 랙 | 적게 필요 | 필요 | 필요 |
| DAC/AEC(구리) 가능성 | 높음 | 거리 따라 | 낮아짐(광 필요) |
| RUD 적합성 | 매우 좋음 | 가능 | 가능 |
| ROD 적합성 | 랙 간 케이블 고려 | 중앙 집중 관리 쉬움 | 케이블 길이 부담 |

ToR이라도 ROD는 8개 Leaf로 분산 연결해 랙 간 케이블이 생긴다. Spine은 별도 랙(서버 랙 부담↓·케이블↑) 또는 같은 랙(compact·부담↑)에 둘 수 있다.

---
---

# 📌 추가 고려사항

본문이 짚지 않았거나 가볍게 지나간, 실무에서 자주 부딪히는 지점들.

## 추가-1. UEC (Ultra Ethernet) — RoCEv2의 다음

RoCEv2의 무손실 운용(PFC/ECN/DCQCN 튜닝)은 까다롭고, go-back-N 재전송·낮은 엔트로피·PFC storm 같은 구조적 약점이 있다. 이를 이더넷 레벨에서 근본적으로 풀려는 것이 **UEC(Ultra Ethernet Consortium)** 표준이다 — 선택적/다중경로 전달, 향상된 혼잡제어, packet spraying을 지향한다. 본문은 RoCEv2/DCQCN까지만 다뤘으므로, 차세대 패브릭을 검토한다면 UEC와 NVIDIA Spectrum-X(이더넷 기반 RoCEv2 확장)를 비교 축에 추가하는 것이 좋다.

## 추가-2. IB vs RoCEv2 — "둘 중 무엇을 살까"의 실판단

본문은 동작 차이를 설명했지만 도입 의사결정 기준은 부족하다.

| 축 | InfiniBand | RoCEv2(이더넷) |
| --- | --- | --- |
| 무손실 | 태생적(credit) | PFC/ECN/DCQCN 튜닝 필요 |
| 운영 난이도 | 낮음(턴키, Subnet Manager) | 높음(스위치·NIC·DSCP 일관 설정) |
| 생태계·락인 | NVIDIA/Quantum 중심·높음 | 멀티벤더·낮음 |
| 라우팅/규모 | 단일 서브넷 중심 | L3 라우팅·ECMP로 유연 |
| 대표 사례 | DGX SuperPOD, top-tier 학습 | 클라우드·하이퍼스케일러 백엔드 |

> 512~1152 GPU급은 IB 스위치 포트 수 제약 때문에, 단일 대형 섀시(예 800G 576포트급 이더넷)로 구성하는 RoCEv2가 오히려 유리할 수 있다 — "IB vs RoCE"보다 "**Leaf/Spine 다수 vs 단일 섀시**" 비교가 더 정확한 경우가 있다.

## 추가-3. SFC 반환 경로의 단방향 우려

SFC는 혼잡 스위치가 S/D IP를 뒤집어 송신자로 신호를 되돌린다. 1-Hop을 넘으면 IP 기반 라우팅이라 **반환 경로가 정방향과 달라질 수 있어**, fallback이 uni-directional하게 전달될 위험이 있다. 정방향=역방향 경로를 보장하는 장치(예: 대칭 라우팅, flow-pinning)가 없다면 SFC 효과가 제한될 수 있다 — 도입 전 검증 필요.

## 추가-4. SHARP·NVLS의 전제 조건과 한계

SHARP(in-network reduce)와 NVLS(NVSwitch multicast)는 All-Reduce를 크게 가속하지만 **하드웨어 전제**가 있다 — SHARP는 IB SHARP NIC/3세대+ NVSwitch, NVLS는 Hopper+ NVSwitch. 이종·구형 클러스터나 RoCEv2-only 환경에서는 적용되지 않아 NCCL이 Ring/Tree로 폴백한다. "AllReduce가 빠르다"를 가정하기 전에 토폴로지·세대 지원 여부를 확인해야 한다.

## 추가-5. 정밀도(FP8/FP4)와 통신량

세대 표의 FP8(H100)·FP4(Blackwell)는 연산만이 아니라 **통신량**도 좌우한다. 더 낮은 정밀도로 텐서를 주고받으면 같은 모델도 gradient/activation 전송 바이트가 줄어 집합통신 부담이 작아진다. 반대로 KV 캐시·activation을 FP16으로 유지하면 메모리·대역폭 압박이 커진다 — 정밀도 선택은 컴퓨트뿐 아니라 패브릭 사이징의 입력값이다.

## 추가-6. Storage Fabric은 rail-optimized가 아니다

GPU Backend는 rail-optimized(GPU당 1 NIC)이지만, **Storage Backend는 rail 개념이 없고 서버당 단일 연결**이 일반적이다(본문 6.2에서 패브릭 분리만 언급). 체크포인트 쓰기·데이터셋 로딩이 GPU를 굶기지 않으려면 Storage Fabric도 non-oversubscribed로 별도 사이징해야 하며, NVMe-oF(RoCE) Incast 혼잡도 GPU Backend와 동일하게 PFC/ECN 대상이다.

## 추가-7. 전력·냉각은 별도 설계 영역

랙당 48~60 kW(NVL72급은 ~120 kW)는 일반 DC의 수 배다. ToR/MoR/EoR 선택이 전력·냉각·케이블에 직접 영향을 주지만, 본 문서는 네트워크 토폴로지 관점만 다뤘다. 실시공에서는 랙 전력밀도·액냉(DLC)·바닥하중·CDU 등 물리 설계를 별도로 검토해야 한다(전용 설계 문서 영역).
