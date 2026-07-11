# Scale-Up · 차세대 인터커넥트(UEC) · 장애/모니터링 · AI 스토리지

Scale-out 네트워크만으로는 한계에 부딪힌 AI 데이터센터가 **여러 xPU를 하나의 거대한 가속기로 묶는 Scale-Up**, 그 위에서 **차세대 이더넷 전송(UEC)** 으로 나아가는 흐름과, 대규모 클러스터의 숙명인 **장애·모니터링**, 그리고 GPU를 굶기지 않는 **AI 스토리지**를 정리한다.

---
---

# Part 1. Scale-Up · 차세대 인터커넥트 · UEC

## 1. Scale-Up vs Scale-Out

- **Scale-out**: GPU 서버 여러 대를 Ethernet/RoCEv2·InfiniBand·UEC 같은 **Scale-out Fabric**으로 수평 확장. 수천~수십만 GPU까지 늘리기 좋지만, GPU 간 통신이 IP/UDP/NIC/스위치 계층을 지나 **지연이 크다**.
- **Scale-up**: 랙 내부(최대 몇 개 랙) 안에서 xPU를 NVLink·UALink·SUE-T 같은 초저지연 interconnect로 직접 연결해, 수십~수백~최대 약 **1,000개 xPU를 하나의 "virtual super-accelerator"** 처럼 사용. 대형 LLM·MoE 추론·멀티모달처럼 GPU 간 메모리 교환이 잦은 workload에 특히 중요.

| 항목 | Scale-out | Scale-up |
| --- | --- | --- |
| 연결 단위 | 서버 NIC ↔ Leaf/Spine | xPU ↔ Scale-up Switch |
| 계층 | Leaf-Spine, 3/5-stage Clos | 보통 **single-hop** |
| Protocol | RoCEv2, UET, InfiniBand | NVLink, UALink, SUE-T |
| Latency | μs 단위 | **ns~sub-μs** 목표 |
| Memory | 분산 memory | shared/coherent 또는 **PGAS** view |
| 확장성 | 수천~수십만 GPU | 랙/Pod 내부 수십~1,000 xPU |

둘은 대체가 아니라 **함께** 쓴다 — Scale-up이 Pod 내부 초저지연 통신을, Scale-out이 Pod·Rack·Cluster 간 연결을 담당한다. Scale-up Pod는 외부 scale-out fabric에 보통 **multi-plane** NIC로 연결해 plane 하나가 죽어도 나머지로 통신이 유지되게 한다.

## 2. 인터커넥트 비교 — PCIe / CXL / NVLink

| 구분 | PCIe | CXL | NVLink |
| --- | --- | --- | --- |
| 정체 | 범용 장치 연결 버스 | PCIe 기반 cache/memory-coherent interconnect | NVIDIA GPU 전용 고속 링크 |
| 목적 | CPU↔GPU/NIC/NVMe 연결 | 메모리 확장·풀링·coherency | GPU↔GPU 고속 통신 |
| 강점 | 범용성·생태계·호환성 | memory semantics, pooling, coherence | 매우 높은 GPU 간 대역폭 |
| 약점 | GPU 간 통신 병목 | 생태계 성숙 중, collective 대체 아님 | 벤더 종속(proprietary) |

- **CXL**은 PCIe를 대체하는 "더 빠른 버스"가 아니라 PCIe 물리 위에 메모리 기능을 얹은 프로토콜. 세 하위 프로토콜: **CXL.io**(장치 발견·I/O), **CXL.cache**(장치가 호스트 메모리를 cache-coherent 접근), **CXL.mem**(CPU가 장치 메모리를 load/store로 접근). AI에서는 CPU 메모리 확장(대형 전처리·vector DB·embedding index), 메모리 계층화(HBM↔CXL memory↔NVMe↔object), CPU-가속기 coherency에 강하다.

## 3. 왜 PCIe/CXL만으로 부족한가

| 대역폭·지연 | 값 |
| --- | --- |
| PCIe 6.0 x16 | ≈ 128 Gbps 양방향 |
| NVLink 5.0 | ≈ **1.8 Tbps** 양방향 |
| PCIe 5.0 latency | 500 ns~1 μs |
| NVLink latency | **100~300 ns** |

PCIe는 서버 내부 장치 연결엔 좋지만, 수십~수백 GPU가 하나의 거대한 메모리/계산 시스템처럼 동작하기엔 latency·bandwidth가 부족하다. 그래서 **NVLink(proprietary), UALink(open), SUE-T(Ethernet 기반)** 같은 scale-up 전용 interconnect가 등장했다.

## 4. Scale-Up 물리 구성 — Single-hop

Scale-up 시스템은 vendor가 랙 단위로 사전 구성(전력·냉각·switch·cabling 포함)한다. 핵심은 **single-hop** — xPU가 scale-up switch를 한 번만 거쳐 다른 xPU와 통신하고, 랙 내부엔 보통 spine이 없다. xPU당 **1.6/3.2 Tbps급**을 목표로 **200 Gbps SerDes**를 여러 lane으로 묶는다.

## 5. SUE-T (Scale-Up Ethernet Transport)

OCP ESUN framework의 Ethernet 기반 scale-up transport. Ethernet PHY를 최대한 쓰면서 xPU 간 load/store/memory transaction을 초저지연으로 전달하고, RoCEv2/UET보다 overhead가 작은 frame을 쓴다. RoCEv2의 BTH가 필요 없다.

- **전송 순서**: `AFH Gen2(Src/Dst xPU) → EtherType → RH(xPU ID/VC/PSN) → SUE Payload → R-CRC → Ethernet FCS`
- **AFH(AI Fabric Header)**: source/destination xPU 정보와 forwarding. **RH(Reliability Header)**: sequence(NSPN/ASPN), VC, partition, reliability 정보. 256-byte frame을 flit처럼 쓰고 최대 4,096-byte SUE PDU를 VC 위에서 전달.
- **LLR**(link 구간 즉시 재전송, NACK 기반)과 **CBFC**(receiver credit이 있을 때만 전송)로 신뢰성·flow control.
- xPU 인터페이스: **FIFO**(단순 streaming) vs **AXI 4.0**(읽기/쓰기 주소·데이터·응답 5채널 분리) — AXI가 inter-xPU 메모리 동기화에 유리. **On-die 통합**(외부 NIC 없이 GPU 칩에 SUE-T 블록 내장)이 특징.

## 6. UALink (Ultra Accelerator Link)

Open scale-up interconnect 표준. NVLink 같은 lock-in을 줄이고 멀티벤더 xPU를 하나의 scale-up 시스템으로 묶는 것이 목표.

- 최대 약 **1,000 accelerator** 도메인, **200 Gbps lane** × 4 = 1 station, xPU당 최대 8 station → 32 lane × 200 Gbps = **6.4 Tbps/xPU**. sub-1μs RTT, fixed-size flit, **PGAS** shared memory.
- **Protocol Stack**: Protocol Layer(read/write/atomic) → **TL**(64-byte flit packing, ordering) → **DL**(640-byte flit, CRC, replay, credit) → **PL**(200 Gbps/lane PHY). RoCEv2의 UDP 방식보다 predictability·효율↑.
- **UPLI**(UALink Protocol Level Interface): originator↔completer request/response(Req/OrigData/RD Res/WR Res).
- **주소 3단계**: **GVA**(프로그램이 보는 가상주소) → source MMU가 **NPA**(fabric이 이해하는 원격 주소)로 변환 → destination link MMU가 **SPA**(로컬 물리주소)로 변환.

## 7. Memory Coherence — Hardware vs Software

여러 xPU가 같은 memory space를 볼 때 모두 최신 값을 보게 해야 한다.

- **Hardware Coherence**(NVLink/NVSwitch): 실리콘이 cache 일관성 보장, 매우 빠르나 **수십 노드를 넘어 확장하기 어렵고** 벤더 종속.
- **Software Coherence**(UALink): runtime/driver/NCCL·SHMEM이 동기화 시점 제어(RMA_FENCE/NOTIFY로 flush/invalidate). 더 많은 xPU로 확장·vendor-neutral에 유리. UALink는 수백 xPU를 위해 SW coherence + PGAS에 초점.

## 8. SUE-T vs UALink vs NVLink

| 항목 | SUE-T | UALink 1.0 | NVLink/NVSwitch |
| --- | --- | --- | --- |
| 성격 | OCP Ethernet scale-up | Open accelerator spec | NVIDIA proprietary |
| PHY | Ethernet PHY | 802.3 계열 | proprietary |
| 전송 단위 | Ethernet frame + AFH/RH | TL 64B → DL 640B flit | NVLink flit |
| Memory | One-sided, PGAS | PGAS(load/store/atomic/fence) | HW-coherent shared |
| Coherence | Software | Software | **Hardware** |
| Latency | sub-μs | 수백 ns~sub-μs | 수십~수백 ns |
| 성숙도 | 초기 framework | 상세 spec | **가장 성숙(상용)** |

> 미래 AI DC는 **Scale-up Pod(내부 초저지연) + Scale-out Fabric(Pod 간)** 의 하이브리드가 될 가능성이 높다.

## 9. UEC (Ultra Ethernet Consortium)

### 등장 배경

기존 AI DC는 InfiniBand 또는 RoCEv2/DCQCN Ethernet을 썼다. RoCEv2는 **PFC·ECN·DCQCN·버퍼 임계값·순서·NIC reordering을 끊임없이 튜닝**해야 한다. AI 클러스터가 이미 **10만 GPU급**을 운영하고 앞으로 **100만 GPU급**을 봐야 하는 규모에서는 이 튜닝이 지나치게 복잡해진다. UEC는 이를 **산업 표준 레벨**에서 풀려는 시도로, IB의 credit·collective 장점과 Ethernet의 개방성·확장성·멀티벤더 생태계를 결합한다.

### 목표와 스택

성능(낮은 latency·높은 throughput·빠른 session ramp), 확장성(1M endpoint), 운영 단순화(plug-and-play), full-stack 통합(NIC·switch·Libfabric·NCCL·collective). **switch뿐 아니라 end-host NIC까지 표준화**한다.

```
PyTorch → MPI/NCCL/RCCL → Libfabric(요구를 전송 의미로 변환)
        → SES(Semantic: JobID/PIDonFEP/Resource Index)
        → PDS(Packet Delivery: 순서/재전송/ACK)
        → UET(Fabric 위 transport packet)
```

핵심은 네트워크가 5-tuple 해시만 보는 게 아니라 **JobID·Resource Index 같은 AI workload 의미 정보**를 활용한다는 점이다.

### UET vs RoCEv2

- **RoCEv2**: UDP 위에 IB **BTH + IB payload**, QPair 기반.
- **UET**: BTH/QPair를 버리고 **PDS/SES** 헤더 사용. JobID·Resource Index·Entropy를 transport에 직접 포함. QPair 대신 JobID/Resource Index로 memory mapping.
- **Encapsulation 2가지**: UDP 기반(Dst Port 49150, Src Port=entropy — 기존 IP Fabric 호환 최선) / Raw IP 기반(IP Protocol 253, PDS의 Entropy field로 load balancing — 더 lightweight).

### Packet Delivery Mode 4가지

| Mode | 의미 | 적합 |
| --- | --- | --- |
| **ROD** | Reliable Ordered | HPC·MPI·순서 중요한 control |
| **RUD** | Reliable Unordered | **AI model parallelism·bulk data** |
| **RUDI** | Reliable Unordered Idempotent | RMA write·gradient update |
| **UUD** | Unreliable Unordered | telemetry·fire-and-forget |

- **RUD가 AI에 중요한 이유**: 서로 다른 GPU 메모리 영역에 쓰는 데이터는 순서가 바뀌어 와도 정확한 위치에 배치할 수 있다. ROD는 reordering buffer로 대기해 latency가 늘지만, RUD는 순서와 무관하게 직접 memory placement가 가능해 **packet spraying과 잘 맞는다**. RUD/RUDI는 손실된 packet만 선택 재전송(go-back-N 회피).

### Congestion Management

- **NSCC**(송신자 중심): congestion window로 전송량 조절(ACK/SACK/NACK·ECN·RTT 기반).
- **RCCC**(수신자 중심 credit): 수신자가 credit을 나눠줘 여러 sender가 한 receiver로 몰리는 incast를 완화.
- **CBFC**(link-level credit): PFC가 "멈춰라"라면 CBFC는 "받을 공간 있을 때만 보내라". port당 최대 **32 VC**, PFC보다 많은 lossless class.
- **Packet Trimming**: 혼잡 시 packet을 drop하는 대신 payload를 잘라 목적지로 보내, 어떤 packet을 재전송해야 하는지 빠르게 알린다(JCT 개선).
- **LLR**: end-to-end 재전송을 기다리지 않고 문제 link 구간에서 즉시 재전송(ASIC 구현).
- **INC(In-Network Collectives)**: AllReduce/AllGather를 spine이 collective tree root처럼 도와 중복 traffic·leaf-spine 대역폭을 줄인다.

### InfiniBand vs RoCEv2 vs UET

| 항목 | InfiniBand | RoCEv2 | UET |
| --- | --- | --- | --- |
| 목표 규모 | 100K 미만 중심 | 100K+ 가능 | **1M endpoint** |
| 혼잡 제어 | Credit | DCQCN/ECN/PFC | NSCC/RCCC/CBFC/trimming |
| 전달 모드 | RC 중심 | RC + 일부 unordered | RUD/ROD/RUDI/UUD |
| Encap | Native IB | UDP/BTH/IB payload | UDP+PDS/SES 또는 IP-only |
| 생태계 | vendor 제한 | Ethernet 넓음 | 표준화 지향(초기) |

> UEC 1.0은 우선 **Scale-out**을 다루고 Scale-up은 이후 단계다. 초기엔 기존 400G/800G Ethernet 위 UDP-UET(AI Base)가 먼저 쓰이고, Raw IP·CBFC·LLR은 greenfield에서 본격화된다. 같은 link에서 PFC와 CBFC를 병행하는 것은 복잡해 greenfield가 유리하다.

## 10. 하이퍼스케일러 트렌드 — SRv6 & Microsoft Fairwater

### 왜 AI Traffic은 다른가

전통 workload는 독립 transaction이 많아 entropy가 충분해 자연 분산되지만, AI는 수만 GPU가 **lockstep**으로 움직인다 — 장시간 synchronous, 낮은 entropy elephant flow, 같은 path·buffer에 몰리는 microburst. **한 packet·한 congested path·한 straggler가 전체 JCT를 늘린다.** PFC/ECN은 microsecond burst에 head-of-line blocking을 만들고, 초 단위 telemetry는 혼잡이 사라진 뒤에야 counter를 읽는다.

### SRv6 Source Routing

IPv6 destination address에 여러 segment(uSID)를 encode해 **송신자(NIC)가 패킷 경로를 통째로 지정**한다. 각 홉을 지날 때마다 처리된 uSID가 shift되고, 최종 목적지에서 원 payload가 드러난다. AI DC가 거대한 단일 job처럼 움직이므로, coordinated scheduling 정보를 가진 host/NIC가 경로를 정하는 것이 ECMP보다 유리하다.

### Microsoft Fairwater (SONiC 기반)

한 DC 안에서 수십만 GPU가 하나의 supercomputer처럼 동작하도록 **51.2 Tbps ASIC**(100G lane 기준 512 fan-out), **512 T0/256 T1** plane, 800G NIC를 8×100G로 나눈 **8 plane**으로 약 **500K GPU**까지 확장. 네 가지 enabling tech:

1. **BGP Scale**: 800G port를 100G로 세분해 pizza box 한 대에서 **512 BGP session** 처리(FRR v10 기반, convergence < 0.1초 목표).
2. **SRv6 Source Routing**: 위 방식으로 host/NIC가 경로 제어.
3. **Packet Trimming**: drop 대신 packet을 trim해 높은 priority로 전달 → destination NIC가 NACK → source가 빠르게 재전송(RDMA timeout 회피).
4. **High-Frequency Streaming Telemetry(HFST)**: ASIC이 **push** 방식으로 **IPFIX**를 SONiC `counter syncd`에 전달, **ms~μs cadence**. (SONiC 2025.1부터 사용)

> Scale-up network도 과거 rack 내부에서 이제 **512/1K/4K port** 규모로 확장되는 흐름이며, 여기서도 Ethernet이 자연스러운 선택지로 제시된다.

---
---

# Part 2. AI DC 장애 & 모니터링

## 1. 장애는 "예외"가 아니라 "기본 전제"

- Meta Llama3: 16,384 GPU로 54일 학습 시 **약 3시간마다 HW 장애**.
- **10만 GPU 규모**: 장애 간격 ~18분, 재시작 ~10분 → **사이클당 유효 학습은 8분**. 상당 시간이 학습이 아니라 복구에 소비된다. GPU는 비싸므로 **장애 내성 = GPU 투자 효율** 문제다.

## 2. 컴포넌트별 장애 분류

| 컴포넌트 | 비율 | 성격 |
| --- | --- | --- |
| **GPU HBM3 Memory** | 22.9% | 최대 원인. ECC/XID/GPU reset/**SDC**(조용히 틀린 값) |
| **PCIe Device** | 18.0% | 증상이 네트워크 문제처럼 보임(NCCL timeout으로 오인) |
| **NCCL Watchdog Timeout** | 9.0% | 통신 미완료. 원인이 GPU/NIC/네트워크/PCIe로 다양 |
| Software Bug | 7.1% | 학습코드·프레임워크·드라이버 |
| Network Switch/Cable | 5.3% | 비율 낮지만 **영향 범위 큼** |
| SSD | 4.4% | 데이터 로딩·checkpoint 실패 |
| Faulty GPU Compute | — | 잘못된 결과(correctness) |
| Kernel Fault/Reboot | — | 노드 1대 소실로 동기식 Job 전체 실패 |
| Numerics/SDC | — | NaN/Inf + 조용한 오류 |
| Cooling/Thermal | — | throttling → straggler화 |

핵심 오해 두 가지: ① **NCCL timeout = 무조건 네트워크 문제**가 아니다(느린 GPU 하나가 전체 통신을 막을 수 있다). ② 완전히 죽은 링크는 ECMP에서 빠지지만, **"살아 있는 척하는 불량 링크"(symbol error, Up 상태)** 가 경로 후보로 남아 더 위험하다.

## 3. 방어 — 체크포인트 & 자동화

대규모 학습의 1차 방어선은 **checkpoint + 자동 장애 대응**: 장애 노드 빠른 감지 → 스케줄링 제외 → 마지막 checkpoint에서 재시작. Meta 사례는 **419건 장애 중 사람의 상당한 개입은 단 3건** — 나머지는 자동 처리. SDC 대응은 checksum·validation run·loss/gradient norm 이상 감지·ECC 모니터링·canary workload.

## 4. 네트워크 장애 = Failure Amplifier

동기식 학습에서는 하나의 전송 지연이 전체 iteration을 늘린다. 평균 throughput보다 **tail latency·outlier transfer**가 치명적이다. classic RoCE는 ECMP hash로 flow가 단일 경로에 고정되어 경로 충돌·낮은 entropy·elephant flow·plane 미활용 문제가 있다.

**OpenAI MRC(Multipath Reliable Connection)**: 하나의 GPU 전송을 **수백 경로로 packet spraying**, **SRv6 source routing**으로 송신자가 경로 지정, 장애·혼잡 경로를 **마이크로초 단위 우회**. multi-plane(800G NIC를 여러 100G plane) + adaptive spraying + SRv6로 **단 2개 스위치 계층으로 10만+ GPU**를 연결해 전력·장애 지점·비용을 낮췄다.

## 5. 왜 AI 모니터링은 더 어려운가

- AI Fabric의 microburst·queue buildup·PFC·ECN·tail latency는 **마이크로초 단위**로 발생해 JCT에 직접 영향. "포트 up/down·대역폭 %"만으로 부족하고 **큐별 버퍼·PFC pushback·ECN·per-hop latency·flow entropy·fat flow**까지 봐야 한다. 평균 대역폭 60%여도 순간 burst가 PFC를 유발해 다른 flow까지 멈출 수 있다.

**SNMP가 약한 이유**: ① 폴링 granularity(1분 주기여도 초 단위 이벤트 소실) ② 장비 CPU overhead ③ 이벤트 의미론 부족 ④ counter wrap(400G에서 32-bit는 7초 내 wrap) ⑤ PFC storm·ECN·RoCE retransmission burst 같은 짧은 이벤트 놓침.

## 6. Streaming Telemetry (gNMI)

모델을 뒤집어 **장비가 collector로 push**(gRPC/gNMI, structured data).

| 방식 | 언제 | 대상 |
| --- | --- | --- |
| **On-Change** | 값이 바뀔 때만 | interface up/down, BGP state, route change |
| **Sample** | 주기마다 | queue depth, byte counter(계속 증가) |

운영 원칙: **상태 이벤트=on-change, 성능 카운터=sample**로 분리(link/BGP flap을 실시간 감지). **High-Frequency Streaming Telemetry**는 ms/μs 해상도로 상태를 push하는 "블랙박스 기록장치" — AI 문제는 "평균"보다 "순간 피크"가 중요하기 때문.

## 7. AI Fabric 필수 지표 & 상관분석

Egress buffer 사용률 / shared vs dedicated buffer / leaf 간 latency / leaf-spine 대역폭 / application flow(sFlow·IPFIX로 elephant·low-entropy) / **PFC(XOFF/pushback)** / **ECN** / per-queue frame loss / mirroring on drop.

switch telemetry만으로는 부족하다. JCT 증가 원인은 GPU compute·storage read·network congestion·PFC/ECN·data loader·scheduler 등 다양 → **switch + server 데이터 상관분석**이 원인 특정의 핵심.

## 8. IFA (In-Band Flow Analyzer)

실제 flow(또는 복제 probe)에 metadata를 붙여 **각 hop 정보를 패킷이 지나며 쌓게** 한다. **Initiator**(측정 대상 생성/clone) → **Transit**(hop마다 metadata 추가) → **Termination**(제거 후 IPFIX export). 쌓는 값: residence time·per-hop latency·ingress/egress port·RX timestamp·queue ID·congestion bit. 효과는 "전체가 느리다"가 아니라 **"몇 번째 hop의 어떤 queue에서 지연"** 까지 좁히는 것. 단 IFA header가 붙으므로 경로 전체 **MTU(보통 jumbo 9000B)** 를 확인해야 한다. **Mirror-on-drop**은 drop counter("몇 개")를 넘어 "무슨 packet이 왜 drop됐는지"를 보여준다.

## 9. Corrective Actions & 자율 운영

congestion 원인 서버 차단 / Job 재시작 / 여유 있는 location으로 Job 이동 / traffic engineering / ACL. 진화는 **사람 판단 → 원인·조치 추천 → 승인 시 자동 → 반복 패턴 자동 → self-driving network**.

> **한계**: 모니터링은 잘못 설계된 네트워크를 좋게 바꾸지 못한다. oversubscription 오설계·부적합 load-balancing·optics 품질 미달·PFC/ECN threshold 오류는 관측만으로 해결되지 않는다 — **올바른 capacity planning과 topology 설계가 선행**되어야 한다.

## 10. 실전 사례 (NVIDIA Spectrum-X)

스위치·SuperNIC(BlueField-3/ConnectX-8)·GPU·NetQ를 통합, **OpenTelemetry/gNMI** 개방 인터페이스. 방향은 네트워크만이 아니라 **AI 애플리케이션·GPU·NIC·스위치를 함께 보는 전체론적 관측성**.

- **대역폭 저하 추적**: LLM effective bandwidth 급락 → SuperNIC의 `roce_adp_retrans`(RoCE 재전송) 상승 → 스위치 telemetry에서 특정 spine 포트 **symbol error** 특정 → 포트 비활성화 후 회복. 핵심은 **AI 워크로드 지표 + SuperNIC 재전송 + 스위치 포트 오류를 연결**한 것.
- **Fabric Misconfiguration**: RoCE 트래픽이 spine/leaf에 불균형 분산되면 leaf 설정 차이를 의심 — telemetry는 장애뿐 아니라 **구성 오류 검증**에도 쓴다.

---
---

# Part 3. AI 데이터센터 스토리지

## 1. 관점 변화 — "창고" vs "초고속 물류"

- **전통 스토리지**: 안전한 저장·공유·백업이 목표. 느리면 파일/백업 지연 문제.
- **AI 스토리지**: **GPU가 데이터를 기다리지 않게** 하는 것이 목표. 느리면 **수천 GPU가 동시에 대기하는 비용 문제**. 한 줄로: 전통=보관 창고, AI=GPU 공장에 원자재를 끊김 없이 공급하는 초고속 물류.

| 구분 | 전통 스토리지 | AI 특화 스토리지 |
| --- | --- | --- |
| 목적 | 안전 저장·공유 | GPU가 안 기다리게 지속 공급 |
| 성능 기준 | IOPS·throughput·latency·가용성 | **GPU utilization·epoch time·checkpoint/restore time·dataloader throughput** |
| 접근 패턴 | 예측 가능 | 수천 worker 동시 read/write, checkpoint storm, small file storm |
| 아키텍처 | SAN/NAS/object | NVMe scale-out, parallel FS, tiered, GPU-aware I/O |
| 네트워크 | FC/iSCSI/NFS/SMB | 100~800G Ethernet, RDMA/RoCE, InfiniBand, NVMe-oF, **GPUDirect Storage** |
| 대표 위험 | 디스크/컨트롤러 장애 | **GPU idle**, checkpoint 병목, metadata 폭주 |

## 2. AI 스토리지가 갖춰야 할 특징

NVMe scale-out + **Parallel File System** + NVMe-oF + RDMA + **GPUDirect Storage** + 데이터 캐시/오케스트레이션 + **tiered checkpoint** + object-file 통합 네임스페이스 + metadata scale-out. AI 워크로드는 다수 worker가 같은 데이터셋 동시 read, 다양한 형식(이미지·비디오·parquet·tar shard), checkpoint 시 큰 write burst, restore 시 다수 동시 read, 추론의 model loading·KV Cache 같은 새 패턴을 만든다. checkpointing은 여러 parallelism과 결합돼 `GPU memory → host → local storage → external repository` 전체 스택이 병목이 될 수 있는 **big-data I/O 문제**다.

## 3. Parallel File System

여러 스토리지 서버·디스크가 파일 데이터를 나눠 병렬 처리(Lustre, GPFS/Spectrum Scale, BeeGFS, WEKA, DDN, VAST, Hammerspace). AI 학습이 병렬이므로 스토리지도 병렬로 공급해야 GPU가 안 기다린다. PyTorch/TF는 POSIX(`open/read/write/stat`)를 쓰고, 병렬 FS 클라이언트가 RDMA/TCP로 요청해 분산 클러스터가 data/metadata를 분산 저장한다. **robust한 스토리지 계층이 없으면 centralized storage가 I/O 병목을 만들어 비싼 GPU가 idle이 된다.**

## 4. GPUDirect Storage (GDS)

**GPU↔스토리지 데이터 이동에서 CPU/system memory를 우회**하는 NVIDIA 기술.

```
일반: Storage → Kernel buffer/Page cache → User CPU buffer → GPU HBM  (CPU copy, DRAM 대역폭 소모)
GDS : Storage → nvidia-fs.ko / cuFile → GPU HBM                      (data path의 CPU 복사 제거)
```

- CPU가 완전히 사라지는 것은 아니다 — 파일 open·metadata·I/O submission 등 **control path에는 여전히 관여**하고, 대용량 data path의 복사만 줄인다.
- 흐름: `cudaMalloc → cuFileBufRegister → Memory Pinning(물리주소 고정) → NIC driver → Zero-copy I/O`. **Pinning이 핵심**(OS가 메모리를 옮기면 장치가 잘못된 주소 참조). PyTorch v2.7.0부터 GDS API 직접 지원.
- 유리한 워크로드: 대용량 tensor shard·checkpoint·embedding·preprocessed binary. 반대로 이미지 decode·tokenization 등 CPU 전처리가 많으면 여전히 CPU가 병목. 스토리지·NIC·GPU·드라이버·FS·커널·PCIe·IOMMU가 모두 맞아야 하는 **만능 아님**.

## 5. NVMe-oF (NVMe over Fabrics)

원격 NVMe를 "로컬 NVMe처럼" 쓰는 **block-level** 기술.

| 방식 | Transport | 네트워크 | CPU/지연 | 적합 |
| --- | --- | --- | --- | --- |
| **NVMe/TCP** | TCP/IP | 일반 lossy IP 가능 | 높음 | cold/warm, 비용, **장거리** |
| **NVMe/RDMA** | RoCEv2/IB | Lossless 튜닝 필요 | 낮음(offload) | hot data, 저지연, GDS 연계 |
| **NVMe/FC** | Fibre Channel | FC SAN | 낮음 | 기존 FC(brownfield) |

- **NVMe/TCP**: TCP 3-way(dst 8009) → NVMe Connect → Capsule Command/Response → Data PDU. **PDU**=NVMe/TCP 메시지 단위, **Capsule**=명령/응답 상자. Read(0x02)/Write(0x01), NSID, PRP/SGL. ACK/순서는 TCP가 처리.
- **NVMe/RDMA**: UDP connectionless라 native TCP flow control이 없고 **RDMA state machine + lossless fabric**이 담당. Discovery(TCP 8009) → RDMA CM(UDP 4420) → NVMe Connect Capsule → I/O Queue 생성 → **RoCEv2 데이터(UDP 4791)**. `Ethernet/IP/UDP/BTH/RDMA/NVMe CMD/Data`. **단일 SQ/CQ pair가 단일 RDMA QPair에 1:1 매핑**.

## 6. 스토리지 네트워크 설계

원칙: **"빠른 성능"보다 먼저 "안정성과 경로 이중화"**.

- **Local PCIe SSD**는 매우 낮은 지연이나 용량·공유·장애 접근성 한계 → 대규모엔 전용 network storage 필요.
- AI 서버는 보통 **NIC 2종**: GPU/Training NIC(RoCEv2 fabric) + Storage NIC(데이터셋·checkpoint). **Training Network와 Storage Network를 분리**(스토리지는 최소 100/200G).
- **A/B Path Diversity**: Fabric A 장애 시 Backup Fabric B가 **동일 IOPS·대역폭·지연을 full capacity로** 제공해야 한다(단순 연결 백업이 아님). Physical A/B(강한 격리, 비용 2배) / Logical A/B(EVPN-VXLAN·VRF·NIC multipath, 비용↓ 복잡도↑) / Collapsed(소규모·edge).
- **장거리**: RoCEv2/RDMA는 **40km 이상 lossless 보장이 어려워** inter-site RDMA는 드묾. 장거리·일반 IP엔 **NVMe/TCP**(TCP가 신뢰성·재전송 담당)가 현실적.
- 중요 질문: checkpoint write burst가 RoCE 학습 트래픽과 같은 Fabric을 쓰면 NCCL latency에 영향 → **물리 분리 vs 같은 Fabric QoS 분리**를 결정해야 한다.

## 7. Block / File / Object & Tiering

| 구분 | 접근 | AI 예시 |
| --- | --- | --- |
| Block | 블록 단위 | NVMe-oF, SAN, 고성능 volume |
| File | POSIX 경로 | NFS/Lustre/WekaFS/VAST — shared dataset·checkpoint |
| Object | S3 API | 모델 아카이브·cold data·cloud backup |

- **Hot/Cold Tiering**: Hot=온프레미스 NVMe(현재 데이터셋·active checkpoint), Cold=클라우드 Object/S3(오래된 checkpoint·archive). WekaFS 예: **Tier 1 = WekaFS over RDMA(hot)**, **Tier 2 = S3 object(cold)**. 앱은 POSIX만 쓰고 클라이언트가 내부적으로 RDMA를 사용.

## 8. InfiniBand for Storage

NVMe/RDMA transport로 사용. 장점: 초저지연, Subnet Manager(LID 중앙 할당), link-level **CBFC**(credit 하나 ≈ 64 bytes), GPUDirect Storage 친화. 제약: 벤더 생태계 좁음, 논리 격리(멀티테넌시)·장거리·troubleshooting(black box) 한계. QoS는 **Virtual Lane(VL, 최대 16)**.

| 항목 | IB CBFC | Ethernet PFC |
| --- | --- | --- |
| 방식 | Credit 기반(사전 제어) | Pause frame(혼잡 시 멈춤) |
| 단위 | per-VL segment | priority class |
| 단점 | credit 부족 시 latency↑ | HOL blocking·PFC storm |

## 9. 추론에서의 AI 스토리지

전통의 "모델 저장소"에서 확장: **모델 weight 빠른 로딩, 여러 LoRA adapter·fine-tuned 모델 artifact 배포, 긴 context의 KV Cache가 GPU 메모리를 크게 점유**. NVIDIA BlueField-4 STX는 agentic AI의 context-heavy·KV cache 문제를 겨냥해 RDMA over Spectrum-X로 NVMe·KV cache storage를 직접 관리한다 → AI 스토리지가 **추론 메모리 확장·context storage·KV cache tier**로 확장.

## 10. 종합

```
AI 학습 성능 = GPU 성능 + GPU 네트워크 성능 + 스토리지 성능
```

좋은 AI 스토리지는 TB가 많은 장비가 아니라, **GPU utilization을 유지하고, checkpoint 시간을 줄이고, 장애 후 빠르게 복구하고, 데이터를 적절 tier에 배치하고, 다수 worker 동시 접근을 안정적으로 처리하는 데이터 공급 플랫폼**이다.

---
---

# 📌 추가 고려사항

## 추가-1. Scale-Up도 결국 물리(전력·냉각)에 갇힌다

Scale-up이 지연·대역폭에서 이상적이어도, 고밀도 xPU 랙은 **전력·냉각·케이블 길이**라는 물리 제약에 묶인다(NVL72급 ~120kW). single-hop을 유지하려면 pod가 물리적으로 컴팩트해야 하므로, scale-up 도메인 크기는 interconnect 성능만이 아니라 **랙 전력밀도·액냉 용량**이 함께 정한다. Part 1의 이상과 Day3의 물리 설계는 한 문제다.

## 추가-2. UEC는 표준이지만 "언제 쓰나"가 관건

UEC는 100만 GPU급을 겨냥하나, 대부분 조직은 그 규모가 아니다. brownfield에서 PFC와 CBFC를 같은 link에서 병행하는 복잡도, NIC·스위치의 UET 지원 성숙도를 감안하면 **당장은 RoCEv2/DCQCN, 신규 대규모 greenfield에서 UEC**가 현실적이다. "UET로 갈아타자"가 아니라 규모·성숙도로 판단해야 한다.

## 추가-3. 장애 대응의 본질은 "빠른 격리 + 값싼 재시작"

장애를 0으로 만들 수는 없다(10만 GPU면 ~18분마다 발생). 그래서 핵심은 **감지-격리-재시작 사이클을 얼마나 싸게 만드느냐**다. checkpoint 빈도↑는 안전하지만 storage write burst를 키우고, 빈도↓는 재시작 손실을 키운다 — checkpoint 주기는 **장애율 × 재시작 비용 vs checkpoint I/O 비용**의 최적화 문제이며, 그래서 스토리지(Part 3)와 직결된다.

## 추가-4. "살아 있는 척하는" 장애가 가장 비싸다

완전한 장애(포트 down, GPU 사라짐)는 자동으로 우회·격리된다. 진짜 비싼 것은 **symbol error 링크·SDC·thermal straggler**처럼 겉으로 정상인데 성능·정확도를 조용히 갉아먹는 회색 장애다. 그래서 모니터링이 up/down을 넘어 **Pre-FEC BER·RoCE 재전송·ECC·gradient norm·per-hop latency** 같은 "회색지대 신호"를 봐야 한다.

## 추가-5. 세 주제는 하나의 루프다

Scale-up/UEC(더 촘촘한 통신) → 더 큰 클러스터 → 더 잦은 장애 → 더 정밀한 모니터링·더 빠른 checkpoint → 더 강한 스토리지. 통신을 키울수록 장애·모니터링·스토리지 요구가 함께 커진다. AI DC 설계는 이 넷을 따로가 아니라 **한 시스템의 상호 제약**으로 봐야 한다.

## 추가-6. Day1~3과의 연결

Day2의 RDMA·혼잡제어(ECN/PFC/DCQCN), Day3의 케이블링·백엔드 토폴로지 위에, Day4는 **① 그 위를 넘어서는 scale-up/UEC(더 큰 통신), ② 그 규모의 장애·모니터링, ③ 그것을 먹여 살리는 스토리지**를 얹는다. 개념(Day1) → RDMA·패브릭(Day2) → 물리·구현(Day3) → 대규모 운영·차세대(Day4) 순으로 읽으면 된다.
