# AI 데이터센터 물리 계층 — 케이블링 · AI 패브릭 심화 · 전원/냉각

---

# Part 1. 케이블링 (Cabling)

## 1. 전기 신호 vs 광 신호 — 근본 갈림길

- **구리 DAC(Direct Attach Copper)**: 도체로 전기가 그대로 통해 변환 장치가 불필요. 전력 소모·지연이 낮다. 단 **표피효과(skin effect)** 로 인한 손실 때문에 단거리 전용.
- **광(Optical)**: 유리섬유는 부도체라 전기가 안 통한다. 전기↔빛을 변환하는 **트랜시버(transceiver)** 가 반드시 필요하다.

핵심 트레이드오프는 **가격 / 전력 / 지연 / 거리** 네 요소다.

| 항목 | Copper | Fiber |
| --- | --- | --- |
| 신호 | 전기 | 빛 |
| 거리 | 짧음 | 매우 김 |
| EMI | 영향 받음 | 거의 없음 |
| 발열/전력 | 고속에서 불리 | 상대적으로 유리 |
| 비용 | 짧은 거리 저렴 | 모듈 포함 시 비쌈 |
| 대표 용도 | 랙 내부 | 랙 간·건물 간·DC 간 |

- 100G Ethernet에서도 구리는 대략 **3 m** 수준으로 제한 — 같은 랙 서버↔ToR엔 되지만 장거리엔 부적합.
- 운영 중 400G optic을 뽑으면 손에 델 정도로 뜨겁다 = 냉각 비용. 광-전 변환에서 열·전력 손실이 크다.
- 짧은 거리(수 m)에서는 오히려 구리가 지연이 더 짧다. NVIDIA·AMD 모두 랙스케일 AI 서버에서 구리를 적극 활용.

## 2. 구리 케이블의 진화: DAC → AEC

| 종류 | 특징 | 전력 | 거리 |
| --- | --- | --- | --- |
| **Passive DAC** | 전자부품 거의 없는 직결 구리 | < 0.15 W | 최대 ~7 m |
| **Active DAC** | 신호 보강 부품 포함 | 증가 | ~10 m |
| **AEC (Active Electrical Cable)** | **retimer(리타이머)** 칩 내장 — 신호 증폭·복원, equalization, 클럭 재설정(CDR) | 중간 | 고속에서 ~7 m |

신호 품질 보정 강도: **일반 DAC < AEC < 광케이블**.

### AEC가 확대된 배경

1. 전송 방식이 **NRZ(1비트: 0,1)** → **PAM4(2비트: 0,1,2,3)** 로 이동. 800G 이상은 PAM4 필수.
2. **대역폭이 커질수록 DAC가 굵어진다** → 장비 전면을 막아 공기 순환 방해 → 냉각 문제.

이 두 이유(신호 손실 + 두께/냉각)로 굵고 무거운 Passive DAC 대신 얇고 신호 복원 가능한 **AEC 영역이 확대**되고 있다.

## 3. 광 트랜시버 내부 구조

800G 트랜시버 주요 부품(가격순): **DSP > EML 레이저 > Driver IC > TIA > TEC > Isolator**.

- **DSP (Digital Signal Processor)**: 트랜시버 안에서 신호를 보정하는 두뇌. 신호 성형/복원, 타이밍, 오류 정정. 초당 수천억 번 연산 → 비싸고 전력·발열이 크다. (BOM 비용의 20~40%, 트랜시버 전력의 약 50%)
- **EML (Electro-absorption Modulated Laser)**: 송신부에서 전기를 빛으로. **DFB Laser**(항상 켜진 손전등) + **EA Modulator**(초고속 셔터). PAM4에서는 4단계 밝기(00~11)로 변조.
- **Driver IC**: DSP 신호를 레이저 구동용으로 증폭(송신).
- **TIA (Trans-Impedance Amplifier)**: 광→전 변환된 미세 전류를 DSP가 읽을 전압으로 증폭(수신).
- **PD (Photo Diode)**: 빛→전기(수신).
- **TEC (Thermo-Electric Cooler)**: 레이저 온도 유지.
- **Isolator**: 반사되어 역주행하는 빛 차단.

**광통신 순서**: ASIC → DSP → Driver IC → [TOSA: EML+TEC+Isolator]로 빛 변환 → 광섬유 → 상대 [ROSA: PD→TIA] → DSP → 목적지.

## 4. 신호 처리 경로 (Optics → PFE ASIC)

고속 광모듈은 단순 변환기가 아니라, 고속 신호를 여러 lane으로 쪼개고 보정해 ASIC이 처리할 형태로 바꾸는 복잡한 장치다.

**경로**: Pluggable Optics → **Demux** → **DSP** → **SerDes** → **PFE ASIC** → SerDes → Mux → Optics

- **Demux/Mux**: 외부엔 400G/800G 한 포트지만 내부는 여러 lane. 400G → 8×50G 또는 4×100G, 800G → 8×100G 또는 4×200G.
- **SerDes(Serializer/Deserializer)**: 병렬 lane ↔ 직렬 신호 변환. **ASIC 내부 SerDes 속도가 포트를 몇 개 lane으로 쪼갤지를 결정**.
- **DSP 기능**: Modulation/Demodulation, FEC(오류 정정), CDR(클럭 복구), Equalization(왜곡 보정).

> 핵심: 포트 속도만 800G로 올린다고 끝이 아니라, ASIC 내부 SerDes lane 속도·수·DSP 처리·전력/발열까지 모두 맞아야 한다.

## 5. 전송 매체와 거리 계층

| 구분 | 거리 | 특징 | 용도 |
| --- | --- | --- | --- |
| Copper/DAC | 매우 짧음 | 저렴·저지연 | 서버↔ToR |
| **MMF** (Multi-Mode) | 짧음~중간 | 850/1300nm, LED/VCSEL, 저렴, 감쇠 큼 | 랙 간 |
| **SMF** (Single-Mode) | 중장거리 | 코어 8~10μm, 레이저, 비쌈 | 건물 간·DC 간 |
| **DWDM** | 장거리/초대용량 | 한 광섬유에 여러 파장 | DCI, 백본 |

- **MMF 등급**: OM3(100G~400G 최대 100m), OM4(150m), OM5(WDM 고려 850+953nm). 여러 빛 경로의 도착 시간 차 → 장거리 부적합.
- **SMF**: 단일 경로. 1310nm(중장거리)/1550nm(장거리·DWDM).
- **광섬유가 EMI에 강한 이유**: 빛으로 이동해 전자기장 영향이 거의 없고 감쇠가 적으며 전기적으로 절연된다.

## 6. 트랜시버 타입 (VR/SR/DR/FR/LR/ZR/CR)

| 타입 | 이름 | 거리 | 매체 |
| --- | --- | --- | --- |
| VR | Very Short Reach | 50 m | MMF |
| SR | Short Reach | 100 m | MMF |
| DR | Data center Reach | 500 m | SMF |
| FR | Far Reach | 2 km | SMF |
| LR | Long Reach | 10 km | SMF |
| ZR | extended Reach | 80 km+ | DWDM |
| CR | Copper | Passive ~7 m / Active ~10 m | Copper |

- 명명 예시 **400G-SR8**: 400G(포트) + SR(단거리) + 8(광 lane 8개, 각 ~53G). 내부 8 lane을 Mux로 결합.
- 물리 배치별 케이블 길이: **ToR**(가장 짧음, 랙마다 스위치) / **MoR**(중간, 스위치 집중) / **EoR**(길고 편차 큼, 스위치 집중).

## 7. 커넥터: LC vs MPO/MTP

- **LC (Lucent Connector)**: 흔한 광 커넥터, 보통 2심(dual). SMF 장거리에 자주. 예: QSFP-DD-FR4 2km는 dual LC.
- **MPO (Multi-Fiber Push On)**: 여러 광섬유를 한 커넥터에 묶음. 고속·고밀도·breakout에 필수. 8/12/24 fiber variant. **MTP는 MPO의 특정 브랜드명**(실무 혼용). Tx/Rx 극성(polarity) 관리 필요.

## 8. 폼팩터·변조·속도 세대

| 속도 | 구성 예 | 폼팩터 | 변조 |
| --- | --- | --- | --- |
| 100G | 4×25G | QSFP28 | NRZ |
| 200G | 4×50G | QSFP56 / QSFP-DD | PAM4 |
| 400G | 8×50G | QSFP-DD / OSFP | PAM4 |
| 800G | 8×100G 또는 16×50G | OSFP / QSFP-DD800 | PAM4 |
| 1.6T | 8×200G 또는 16×100G | 차세대 | PAM4 |

- **QSFP-DD(Double Density)**: 8 lane, 하위 호환 우수 → 기존 DC 전환에 유리. 400G까지 현실적·범용.
- **OSFP(Octal SFP)**: 8 lane, 작게 만들기보다 **고속·고전력·방열**을 강하게 고려한 AI 시대 폼팩터. 800G 이상 중심, 방열/확장성 유리(모듈에 방열판 통합). QSFP와 직접 호환은 어려움.
- **QSFP-DD Cage 타입**: 400G+에서는 "꽂히는가"보다 "**열을 감당하는가**"가 중요 — Type 2A(상단 heat sink), Type 2B(더 높은 heat sink)로 발열 처리 강화.
- **광모듈 로드맵**: 200G(2020) → 400G(2025) → **800G(현재)** → 1.6T(2027) → 3.2T(2029).

## 9. 차세대 Optics: Pluggable / LPO / LRO / CPO

기존 Pluggable은 유연성 최고지만 **DSP 때문에 전력·비용이 높다**. 그래서 DSP를 밖으로 빼거나(LPO/LRO), optics를 ASIC에 통합(CPO)하는 방향으로 진화 중.

| 항목 | Pluggable | LPO | LRO | CPO |
| --- | --- | --- | --- | --- |
| DSP 위치 | 모듈 내부 | ASIC 쪽(제거) | TX 모듈, RX ASIC 쪽 | ASIC 근처 통합 |
| 유연성 | 매우 높음 | 중간 | 중간~높음 | 낮음 |
| 전력 | 높음 | 낮음(~50%↓) | 절충 | 낮음 |
| 교체성 | 모듈만 쉬움 | 가능 | 가능 | 어려움 |
| 거리 | 표준 | 단거리(~30m) | 절충 | 통합형 |
| 상호운용성 | 높음 | 도전 큼 | LPO보다 유리 | 생태계 성숙 필요 |

- **LPO (Linear-drive Pluggable Optics)**: 모듈 DSP를 빼고 보정/FEC를 스위치 ASIC이 처리. 전력·가격·지연↓, 모듈 소형화. 단 단거리 한정, 칩 호환성·상호운용성·검증 부담.
- **LRO (Linear Receive Optics)**: **수신 DSP만 제거, 송신 DSP 유지**하는 절충형 — 표준 준수·안정성 확보.
- **CPO (Co-Packaged Optics)**: optics를 스위치 ASIC 옆 기판에 통합, 긴 PCB trace 제거 → 초고속·에너지 절감(약 30%+, 사례에 따라 3.5배). 스위치는 순수 광케이블만 외부 연결. 단 초기 비용·유연성↓, 장애 시 모듈만 교체 어려움(**장애 도메인 확대**).

## 10. NVIDIA 플랫폼 세대별 케이블링 진화

| 플랫폼 | 랙 구조 | 냉각 | 판매 단위 | 케이블 |
| --- | --- | --- | --- | --- |
| **A100** | ToR | 공랭 | 랙단위 X | GPU노드↔스위치 200G 구리 40개, 랙 간 AOC |
| **H100** | MoR(컴퓨트랙+네트워크랙) | 공랭 | 랙단위 X | 서버1·2↔ToR: DAC / 서버3·4: AEC / 서버↔NVLink Switch: AOC |
| **GB200** | EoR, NVL72 | 수냉(GPU)+공냉 | **랙단위(NVL72)** | 백플레인(구리), AEC(1~4번 랙), AOC(5~8번 랙) |
| **VR200** | EoR, NVL72/**NVL144** | **전부 수냉** | 랙단위 | 백플레인(구리), **AEC를 CPO로 제거**, scale-out은 AOC |

- **H100**에서 서버↔NVLink Switch는 구리로도 거리는 되지만, **케이블 두께로 전면이 막혀 냉기 순환이 안 되어** 광(AOC)을 썼다 — "케이블 굵기 = 냉각 문제"의 대표 사례.
- **GB200**부터 컴퓨트랙에 **백플레인** 도입 → 랙 내부 케이블이 사라지고(카트리지에 NVLink 구리 사전 연결), 수냉 배관 때문에 컴퓨트랙 구역과 네트워크·관리랙 구역을 분리.
- **VR200**은 모든 부품 수냉 + **Spectrum-X 스위치에 CPO** 도입 → 광트랜시버가 칩에 내장되고 스위치는 순수 광케이블만.
- **Blind-mate(블라인드 메이트)**: GB200 NVL72부터 트레이·전력 버스바·액냉 패브릭을 서랍처럼 밀어 넣으면 후면 커넥터가 자동 체결(Auto-coupling) → 인적 오류 차단.

## 11. 운영·트러블슈팅 — 고속 광링크 지표

AI 클러스터에서는 링크가 완전히 down되지 않아도 **FEC 정정량이나 BER이 증가하면 collective 통신 성능이 흔들린다**. 모니터링: Tx/Rx Power, 온도, 전압/전류, **Pre-FEC BER**, Post-FEC BER, Corrected/Uncorrected FEC, lane별 상태, link flap, DOM/DDM. 구조별로 장애 대응이 다르다(Pluggable=모듈 교체 쉬움, CPO=스위치 단위 spare 중요).

---
---

# Part 2. 네트워크 딥러닝 인프라 (AI 패브릭 심화)

## 1. RDMA 심화 — RDMA Write 4단계

RDMA는 한 노드의 NIC가 원격 노드의 **등록된 메모리**에 CPU·커널 스택을 우회해 직접 read/write한다. RoCEv2는 RDMA를 Ethernet/IP/UDP에 캡슐화하지만 **UDP는 손실 복구를 안 하므로** lossless·저지연으로 설계해야 하고, 단 한 번의 packet loss가 학습 전체를 재시작시킬 수 있다.

RDMA Write는 "주소만 알고 바로 쓰기"가 아니라 4단계가 모두 맞아야 동작한다.

### ① Memory Registration
- **Protection Domain(PD)** 할당 — VRF/tenant 같은 격리 단위. 등록 메모리·QP가 PD에 묶인다.
- 물리 메모리(비연속 가능)를 **가상의 연속 블록**으로 등록하고 access right 정의(CCN=Local Read, SCN=Remote Write).
- 키 생성: **L_Key**(로컬 접근), **R_Key**(원격 접근). SCN의 R_Key는 management connection으로 CCN에 전달.

### ② Queue Pair 생성
- **QP** = Send Queue + Receive Queue. **CQ(Completion Queue)** 로 완료 통지.
- Service Type: 신뢰성(Reliable/Unreliable) + 유형(Connection/Datagram). AI/ML·NCCL·스토리지는 신뢰성이 필요해 **RC(Reliable Connection)** 를 자주 쓴다.
- **P_Key(Partition Key)**: 두 NIC port가 같은 partition에 속해야 통신. VXLAN VNI처럼 가상 연결 식별자. QP 상태를 INIT으로 두고 P_Key 설정.

### ③ Connection 초기화
- CCN→SCN **REQ**(LID, CA GUID, QP number, service type, 시작 **PSN**, P_Key, payload size) → SCN→**REP** → CCN→**RTU**로 완료.
- QP 상태: `INIT → Ready to Send / Ready to Receive`.
- **PSN(Packet Sequence Number)**: 순서 확인, ACK/NAK/retry, out-of-order/loss 감지.

### ④ Work Request와 RDMA Write
- 앱이 **WR**을 Send Queue에 **WRE**로 posting(OpCode, Local Buffer+L_Key, Remote Buffer+R_Key, Send Flag).
- NIC가 헤더 구성: **IB BTH**(P_Key, Dest QP, PSN, Ack Required, OpCode) + **RETH**(Remote Address, R_Key, Length — 수신 NIC가 "payload를 어디에 쓸지" 판단하는 핵심).
- 캡슐화: `Ethernet / IP / UDP(dst 4791) / IB BTH / RETH / Data`.
- **수신(SCN)**: P_Key 확인 → R_Key 검증 → 가상→물리 주소 변환 → Receive Queue → CQ 통지. **서버 CPU가 payload를 복사하지 않고** NIC가 메모리에 직접 write.

## 2. AI Fabric 설계의 도전 과제

딥러닝은 여러 GPU가 동시에 forward/backward 후 gradient를 교환해 동일 weight update를 해야 한다. **한 GPU·한 링크만 느려도 전체 iteration이 지연**된다. 그래서 평균 대역폭보다 "동기화 통신이 막히지 않는가"가 더 중요하다.

- **Egress Congestion**: gradient sync 시 여러 GPU가 한 host의 NIC로 몰려(예 200G 링크에 최대 800G egress) 버퍼 초과 → drop.
- **Single Point of Failure**: 통신 빈도는 병렬화 전략에 좌우(Data parallel=backward, Model/Pipeline=forward에도). **Rail switch 장애 = 그 rail의 모든 GPU 고립**. 기존 DC보다 SPOF에 민감.
- **HOL Blocking**: 한 GPU/NIC가 자기 트래픽 + 남의 forwarding을 함께 하면 queueing delay → collective 지연.
- **Hash Polarization**: 두 GPU가 QP를 열면 모든 sync가 하나의 큰 flow(**elephant flow**, 수백MB~수GB). 여러 elephant flow가 같은 uplink/spine으로 hash되면 한 링크 과부하·다른 링크 유휴.

→ 설계는 **Topology-aware**(GPU-NIC-rail 정렬), **Congestion-aware**(fan-in/out, PFC/ECN), **Failure-domain 최소화**, **Flow 분산 개선**(adaptive routing/spraying/flowlet), **Network-aware scheduling**으로 대응.

## 3. 혼잡 제어 심화 — ECN / PFC / DCQCN

> Day2에서 개념을 다뤘고, 여기서는 **임계값·헤더·설정 순서**를 깊게 본다.

### ECN — egress, marking, 점진적 감속

- RDMA NIC가 IP ToS에 **DSCP=24, ECN=10(ECN-capable)** 설정. 스위치가 DSCP 24 → **QoS-group 3 → egress queue 3**.
- **WRED 기반 marking**:

```
depth < WRED Min           : 정상 전달
WRED Min ≤ depth < WRED Max : 일부(무작위) packet을 ECN CE(11) 마킹
depth ≥ WRED Max           : 모든 packet CE 마킹
depth ≥ Drop Threshold     : drop
```

- 수신 NIC가 CE(11) 감지 → **CNP**(OpCode 0x81, **DSCP=48**) 생성 → 스위치가 DSCP 48 → **QoS-group 7 → strict-priority queue 7**(피드백이 안 막히게) → sender NIC가 **inter-packet delay 증가**로 해당 QP 감속. 혼잡 해소 후 **천천히** 회복.

| 트래픽 | DSCP | QoS Group | Queue |
| --- | --- | --- | --- |
| RoCEv2 Data | 24 | 3 | Queue 3 |
| CNP | 48 | 7 | Queue 7(strict) |

### PFC — ingress, 즉시 pause

- ingress queue가 **xOFF** 초과 시 upstream에 pause frame → 해당 priority만 즉시 정지, **xON** 아래로 내려가면 재개.
- **DSCP-based PFC**: 원래 802.1Qbb는 L2 PCP 기반이나, AI 백엔드는 대규모 **L3 routed fabric**이라 IP **DSCP**로 클래스 식별.
- **Pause Frame**: Ethertype **0x8808**, Class Enable Vector(어느 queue를 멈출지 8비트), **Quanta**(pause 기간, 512-bit-time; 400G에서 최대값이면 ≈ 83.9µs). **quanta≠0=pause, quanta=0=resume**.
- **Buffer headroom**: pause가 도착해 멈출 때까지의 **in-flight 트래픽**을 흡수할 여유 버퍼. 부족하면 pause해도 drop.
- **Backpressure Cascade**: pause가 상류로 hop-by-hop 전파되어 넓은 영역이 멈출 수 있다. PFC는 loss 방지 장치일 뿐 **혼잡 자체(hotspot/oversubscription)를 없애지 못한다**.
- 협상은 LLDP의 **DCBX**(IEEE DCBXv2 표준 / CEE 구형)로. 양단이 같은 DSCP-priority 매핑을 공유해야 한다.

### DCQCN — ECN 먼저, PFC 백업

핵심 임계값 순서: **`xON < WRED Min < WRED Max < xOFF`**

| Threshold | 의미 |
| --- | --- |
| xON | PFC resume |
| WRED Min | ECN 마킹 시작 |
| WRED Max | 모든 packet CE |
| xOFF | PFC pause 시작 |

- ECN이 먼저 동작해야 하는 이유: ① CNP로 **점진적 감속**(graceful) ② 불필요한 PFC 회피 ③ fabric 안정성. **xOFF가 ECN보다 먼저 도달하면** → Sudden Pause → backpressure/HOL/GPU jitter → xON/xOFF를 오가는 **stop-and-go 진동**.
- 설정 6단계 요약: Classification(DSCP) → Internal QoS Label → Scheduling/WRED/**ECN 옵션 필수**(없으면 drop으로 동작, RDMA에 치명) → Network-QoS+PFC(MTU 9216 jumbo, `pause pfc-cos 3`, **PFC Watchdog**로 deadlock 감시) → System/Interface bind.
- 모니터링: ECN CE count, CNP send/recv, Queue 3 occupancy, WRED marking, PFC xOFF/xON, watchdog trigger, RoCE retransmission, **GPU step-time jitter, NCCL allreduce latency**.

## 4. 로드밸런싱 — Flow / Flowlet / Packet Spraying

RoCEv2 백엔드에서 기존 L3 ECMP만으로는 대용량 RDMA를 고르게 못 나눈다.

- **Flow-based ECMP**: 5-tuple(Src/Dst IP, Src/Dst Port, Protocol) 해시로 한 flow를 한 경로에 고정. AI의 line-rate elephant flow는 소수라 **hash polarization**(여러 큰 flow가 같은 uplink)으로 불균형.
- **Flowlet(Adaptive Routing)**: 한 flow를 비활성 gap으로 구분되는 **flowlet**으로 쪼개, 링크 사용률이 threshold 초과하면 일부 flowlet을 다른 spine으로. **재정렬 위험이 낮으면서** 재분산 — 가장 균형 잡힌 절충.
- **Packet Spraying(per-packet)**: 같은 flow의 packet을 여러 경로로 흩뿌림. 활용률 최고지만 경로별 지연차로 **out-of-order** 도착 → 수신 NIC 재정렬 부담.

### RDMA Write First/Middle/Last와 Spraying

큰 RDMA Write는 `First → Middle … → Last`로 분할된다. **First만 RETH**(R_Key, Length, Virtual Address)를 갖고 Middle/Last는 First의 context+PSN 순서에 의존한다. 따라서 순서가 중요해 spraying 시 재정렬이 문제다.

→ **RDMA Write Only**(NVIDIA ConnectX-5 이후): **모든 packet이 자체 RETH를 포함(self-contained)** → 앞 packet 의존이 줄어 per-packet 분산이 현실화. (Cisco Nexus DLB의 `mode per-packet` 등에서 활용. 단 DLB flow는 egress QoS/ACL/SPAN 제약이 있을 수 있어 observability 검증 필요.)

| 방식 | 단위 | 장점 | 위험 |
| --- | --- | --- | --- |
| Flow ECMP | Flow | 순서 보장·단순 | elephant flow 편중 |
| Flowlet AR | Flowlet | 활용률↑, 재정렬 낮음 | gap/threshold 판단 |
| Packet Spraying | Packet | 경로 분산 최고 | 재정렬 → self-contained 전송 필요 |

## 5. 백엔드 토폴로지

> **정정**: PCIe는 full-duplex — **PCIe 5.0 x16 ≈ 방향당 64 GB/s, 양방향 합계 128 GB/s.**

### GPU 서버 NIC — Shared vs Dedicated

- **Shared NIC**: 8×H100이 NVSwitch로 내부 연결(GPU-to-GPU ~900 GB/s)하고 외부는 공유 ConnectX-7 200GbE(~25 GB/s) → **shared NIC가 병목**. 개발·소모델·비용 민감용.
- **Dedicated NIC per GPU**: GPU마다 전용 NIC를 rail(1~8)에 → 대역폭 분리·예측 가능, GPUDirect RDMA 안정. NIC/포트/케이블·전력·비용 증가. **production 학습의 표준**.

### 모델 크기와 GPU 수 (8 GPU × 80GB = 640GB 서버)

| 모델 | 필요 메모리 | 의미 |
| --- | --- | --- |
| 8B | ~16GB | 단일 GPU |
| 70B | ~140GB | 최소 2 GPU(intra-host NVLink) |
| 405B | ~810GB | 단일 서버 초과 → multi-server |

### 확장 단계

```
Single Rail(SPOF) → Dual-Rail(dual-port NIC, MLAG/EVPN ESI로 HA)
 → Pod(2-tier / 3-stage Clos: Rail→Spine→Rail)
 → Multi-Pod(3-tier / 5-stage Clos: Rail→Spine→Super-Spine→Spine→Rail)
```

- **Dual-Rail**: 각 GPU dual-port NIC를 독립 rail switch 2대에 → 장애 시 고립 방지. 두 스위치를 하나로 보이게 하려면 MLAG(vPC/Virtual Chassis/Arista MLAG) 또는 **BGP EVPN ESI Multihoming**(vendor-neutral, control plane 복잡).
- **Rail-only cross-rail**: rail 간 spine이 없으면, 다른 rail GPU 통신은 **host 내부 NVLink로 목적 rail의 NIC를 가진 GPU에 복사 후** 그 NIC로 송신. **NCCL**이 경로 선택과 NVLink copy·NIC 전송을 스케줄.
- **non-blocking 조건**: Rail downlink 총량 = Spine uplink 총량(예 32×100G=3.2Tbps → 800G 또는 2×400G port-channel). GPU 클러스터는 all-reduce east-west 때문에 oversubscription을 피한다.

## 6. Rail 설계 — Multi / Dual / Single per Switch

Rail Layer(GPU NIC ↔ Rail Switch)는 **확장성·장애 도메인·비용·운영 복잡도**를 좌우한다.

| 설계 | Isolation | Failure Domain | CapEx | NVIDIA SU 적합성 | 추천 |
| --- | --- | --- | --- | --- | --- |
| **Multi-Rail-per-Switch** | Logical only | 가장 큼 | 가장 낮음 | 낮음 | lab/PoC/small |
| **Dual-Rail-per-Switch** | Logical only | 중간~큼 | 중간 | 부분 | intermediate |
| **Single-Rail-per-Switch** | **Physical** | 가장 작음 | 가장 높음 | 높음 | production/mission-critical |

- **Multi-Rail**: 한 스위치에 여러 logical rail을 VLAN/subnet으로 분리. 스위치 수·비용 최저지만 **스위치 장애가 모든 rail에 영향**.
- **Dual-Rail**: 한 스위치에 2 logical rail. 절충. 그래도 **물리 failure domain은 하나**.
- **Single-Rail**: 물리 스위치 하나 = rail 하나. **물리 격리**로 장애 도메인이 작고 예측 가능, 설정 단순, **NVIDIA Scalable Unit과 가장 잘 맞음**. 대신 스위치/케이블/전력/냉각 비용↑.

> 세 방식 모두 dual-NIC-per-GPU로 GPU를 두 스위치에 이중 연결하면 rail switch SPOF를 없앨 수 있다. 핵심은 rail이 **논리 분리**만인지 **물리 분리**까지인지 구분하는 것. AI 학습에서 rail 장애는 단순 링크 장애가 아니라 **GPU collective 지연·NCCL timeout·job failure**로 번진다.

---
---

# Part 3. 전원 · 냉각 · AI 데이터센터

## 1. 부지 선정 — 전기가 핵심

데이터센터 필수 환경: **온도 25℃ 이하, 습도 RH 60% 이하**, 전원, 인터넷. 저장 단위는 **Rack(랙)**, 이송은 무진동 물류체인.

- **전기를 확보할 수 있는 땅**이 핵심. 수전 전압은 초고압(한국 22,900V "투투나인" 또는 154,000V "일오사").
- 전력 인입은 도로를 파고 지중 매립 → 공사비·거리 비례. 지자체 **3년 내 재굴착 금지** 규정 때문에 인입 대기가 길어질 수 있고, 변전소가 멀면 유지보수 굴착 리스크↑.
- 수도권은 발전소 건설이 어렵고 송전탑 민원으로 **상시 전력 부족**.
- 그 외: Network 이중화 인입(고객은 통상 **3개 이상 POE** 요구), 침수·폭발 위험·민원 회피.

### 용량·면적 산출

- 용량 = **IT 전력량**. 랙 밀도: 일반 서버 ~15kW/랙, **AI 서버 ~72kW/랙**.
- 예: (15kW×1,200) + (72kW×100) = 25,200kW = **25.2MW**.
- 연면적: 개략 **MW당 ~300평** → 25.2MW × 300 = 7,560평. 건폐율·용적률 규제 고려.

## 2. 공간 구성

4대 구성: 토지/건물 · 전원 · 냉방 · Network.

### 데이터홀 — 하중과 기둥

| 공간 | 바닥 하중 |
| --- | --- |
| 일반 사무실 | ~250~300 kg/m² |
| **데이터홀** | **1,200~2,400 kg/m²** |

- 1U 서버 ~15~20kg × 42U ≈ 랙당 630~840kg + 랙·배관·케이블. 하중은 슬라브→보→기둥으로 전달, 통상 **기둥 간격 ~11m**.

### 물류

```
로딩독 → Unpacking → Staging → Test → 화물 E/V → 데이터홀
```

- **Staging Room**: 서버를 데이터홀 환경에 적응시켜 온도차 **결로 방지**(없으면 입고 후 최소 6시간 적응 후 전원).

## 3. 전원시설 — 무정전이 원칙

전력계통(Grid): 발전소 → 변전소(승/강압) → 송전 → 배전. 데이터센터 인입도 **주/예비로 이중화**(2개 이상 발전소+변전소).

```
발전소A→변전소A ─(주)→ 수전(154kV) → 변압기(154kV→6.6kV→380V) → UPS → 서버
발전소B→변전소B ─(예비)→                                        ↑
                            비상발전기(EDG) ─(정전 시)────────────┘
```

- **UPS (Uninterrupted Power Supply)** — 가장 중요. 서버는 정전 후 **~20ms 이내** 재공급이면 무탈(내부 캐패시터 버퍼). UPS는 **~3~5ms** 내 배터리로 공급(통상 10분 용량)하면서 동시에 **비상발전기(EDG)** 시동. 발전기는 정격 도달까지 **15~30초** 걸리므로 UPS가 그때까지 버틴다.
- **배터리**: 과거 납배터리 → 현재 대용량은 부피·무게·수명 유리한 **리튬이온(Li-ion)**.
- **변압기**: 수전 **40MW 초과** 시 154kV 초고압 수전. 이유는 **송전효율** — `P=V×I`에서 전압↑→전류↓, 손실 `I²R`이 줄어 더 얇은 케이블로 같은 전력 전송(경제적). 154kV를 저압 380V(또는 중압 6,600V 경유)로 감압.

## 4. 냉방(공조)의 기본

- 서버는 **Cold Aisle의 25℃ 공기**를 냉각팬으로 전면 흡입 → 내부 열 제거 → **~35℃** 배기 → 냉방시스템이 24℃로 냉각해 다시 공급(=공기조화).
- **냉방부하**: 서버 전력 1kW당 **860 kcal/h**(전력 대부분이 열로 전환).
- **냉방부하 공식**: `냉방부하(kcal/h) = 0.29 × 온도차(℃) × 풍량(m³/h)`.
  - 부하 고정 시 온도차↑→풍량↓, 풍량↑→온도차↓.
  - 예(10kW 랙, ΔT 10℃): 8,600 ÷ (0.29×10) ≈ 2,965 m³/h ≈ **4.94 m³/min/kW**. 통상 서버는 kW당 **4.5 m³/min 이상** 요구.
  - **핵심: 서버 입장에서 온도차보다 풍량(흡입 공기의 양)이 더 중요.**
- **CRAH**(Computer Room Air Handler)와 그 벽체형 **FWU**(Fan Wall Unit). FWU 사용 시 데이터홀 층고 ~8m 필요(일반 오피스의 2배+).
- **Chiller(냉동기)**: 더워진 냉수(28℃)를 20℃로 재냉각. **Buffer Tank**: 정전으로 냉동기가 멈춰도 재기동까지 미리 보관한 냉수를 공급(냉방 연속성). **냉각탑(Cooling Tower)**: 최종 열을 외부로 배출.

## 5. AI DC — 왜 전력·냉각이 핵심인가

- **DGX H200**: 8U에 **10.2kW** → 랙에 4대면 **40.8kW/랙**. 국내 일반 코로케이션은 8~12kW.
- 코로케이션 랙 밀도 진화: 20kW는 이제 기본, **GB200 NVL72 = 120kW**, **Vera Rubin NVL144 = 랙당 600kW 목표(2026)**. 액냉 도입률 22%, Direct-to-chip이 점유율 47%.
- 잘못된 코로케이션 선택 시 열 차단·전력 장애로 GPU 투자가 고립. "AI-ready"라며 실제로 80kW 랙을 못 식히는 사례 존재. NVIDIA DGX-Ready 인증 시설은 소수 → 판매자 우위 시장.

### 공랭의 물리적 한계

- 40.8kW 랙은 풍량 ≈ 183.6 m³/min 필요 — 15kW 랙 대비 ~2.7배 → FWU 면적/높이를 2.7배로 키워야 하나 **물리적으로 어려움**. 서버 팬을 키우면 팬 소비전력이 늘어 냉기가 더 필요한 악순환.

### 물이 공기보다 압도적인 이유

**물은 공기보다 같은 부피 기준 약 3,000배 이상 열을 운반**한다. 핵심은 밀도와 비열.

| 항목 | 공기 | 물 |
| --- | --- | --- |
| 밀도 | ~1.2 kg/m³ | ~1,000 kg/m³ |
| 비열 | ~1.0 kJ/kg·K | ~4.18 kJ/kg·K |
| 열전도율 | ~0.026 W/m·K | ~0.6 W/m·K |
| 누수 위험 | 없음 | 있음 |

열 운반 `Q = 질량유량 × 비열 × ΔT` → 밀도·비열이 큰 **물이 압도적으로 유리**.

## 6. 액체 냉각 기술

NVIDIA는 **액침(immersion)은 GPU 품질 보증 불가**를 선언 → 서버업체는 **DLC** 방식으로 제작.

| 방식 | 냉각 위치 | 개조 필요 | 효율 | 감각 |
| --- | --- | --- | --- | --- |
| **Immersion** | 장비 전체(절연 유체) | 매우 높음 | 매우 높음 | 데이터센터/장비 구조가 바뀜. Single/Two-Phase |
| **Cold Plate (DLC/D2C)** | 칩(GPU/CPU) 위 | 중간 | 높음 | **AI GPU 서버에서 가장 현실적**. 발열 큰 부품=액냉, 나머지=공냉(Hybrid) |
| **RDHx(후면문)** | 랙 후면 배기 | 낮음 | 중간 | 장비 내부는 여전히 공랭, 랙 열만 줄이는 보조형 |
| **Sprayed** | 부품 표면 노즐 분사 | 높음 | 높음 | 냉각액 적게 사용, 공랭 대비 총에너지 25.8%↓ |

### DLC (Direct Liquid Cooling to Chip)

- GPU/CPU 위에 **Cold plate**를 붙이고 냉수 배관 연결. 발열 큰 부품=액냉, NIC·DIMM·SSD·PSU 등=공냉 → **Hybrid Cooling**. 외부 **CDU** 또는 facility water 연동, 누수 감지·quick disconnect 필요. DLC 랙은 열순환 위해 상단 1.5m+ 공간.

### 온수(고온) 냉각 — Free Cooling 365일

NVIDIA 신규 AI 서버는 **45℃ 고온 액체 냉각**으로 물 사용량을 거의 제로화, 팬리스(Fanless).

1. **발상 전환**: 차가운 물(10~15℃)이 아니라 **45℃ 미지근한 냉각수**를 유입.
2. **열역학(ΔT)**: 최신 GPU 한계 온도가 **85~95℃**라 45℃ 물로도 40℃ 이상 온도차 → 열 흡수 문제없음.
3. **Chiller 제거 → 100% Free Cooling**: 랙을 빠져나온 물이 **60~65℃**라 한여름 외기 40℃로도 식힐 수 있어 **기계식 냉동기 없이 자연냉각이 연중 가능** → PUE↓.

## 7. 열·전력 효율과 PUE

냉각 시스템은 데이터센터에서 **두 번째로 큰 전력 소비원**. AI 클러스터 설계는 "몇 U 남았나"보다 **"몇 kW·몇 kW의 열을 감당하나"** 를 먼저 본다.

| 영역 | 전력 비중 |
| --- | --- |
| IT 장비 | 40~45% |
| 냉각 시스템 | 35~40% |
| 전력분배/UPS/조명/기타 | ~17% |

- **PUE = 전체 전력 ÷ IT 전력**. 이상적 1.0, 현실은 그 이상. AI는 IT 전력 자체가 커서 **PUE를 조금만 낮춰도 막대한 절감**.
- 광모듈 전력: QSFP28 ~3.2W, QSFP56-DD 최대 ~12W, **OSFP 최대 ~15W**. OSFP 64개 = ~960W → 광모듈만으로 거의 1kW.
- AI 랙 전력 예: DGX H100(13.2kW) + 64×400G 스위치(~3kW) + OSFP 64개(~0.96kW) ≈ **17.2kW** → 이미 일반 랙 소비 수준이라 "빈 공간에 서버 더 넣기"가 불가능.

## 8. Airflow 설계

AI 랙은 발열이 커 **airflow 방향을 장비 주문 전에 먼저 결정**하고 **모든 장비 방향을 일치**시켜야 한다.

| 방식 | 흐름 | 특징 |
| --- | --- | --- |
| **Front-to-Back** | 앞→뒤 | optics를 먼저 냉각 → 일반적으로 선호 |
| Back-to-Front | 뒤→앞 | PSU 먼저 냉각, optics는 데워진 공기로 → heat sink 추가 가능 |
| Bidirectional Fan | 양방향 | 유연하나 설계·비용 부담 |

- 스위치 전면에 고속 optics(포트당 10W+)가 몰려 있어 **Front-to-Back이 optics 냉각에 유리**. Hot/Cold Aisle·랙 배치·케이블링·냉각기 위치가 모두 airflow와 연동된다.
- 악순환: 전력↑ → 열↑ → 팬 속도↑ → 냉각 전력↑ → 전체 비용↑.

---
---

# 📌 추가 고려사항

## 추가-1. "케이블 굵기 = 냉각"이라는 숨은 제약

케이블 선택은 신호·거리 문제로만 보기 쉽지만, 실제로는 **냉각 제약**이 크다. H100 세대에서 서버↔NVLink Switch를 구리로 연결할 수 있었음에도 **굵은 케이블이 장비 전면을 막아 냉기 순환을 방해**해 광(AOC)으로 갔다. AEC가 뜬 이유도 절반은 "얇아서 공기 흐름을 안 막아서"다. 즉 케이블 도면은 airflow 도면과 함께 봐야 한다.

## 추가-2. CPO는 성능을 얻고 유연성을 내준다

CPO는 전력·밀도·신호 품질에서 크게 유리하지만 **장애 도메인이 스위치 단위로 커진다**. Pluggable은 고장 난 모듈만 뽑아 갈면 되지만, CPO는 optics 장애가 스위치 교체/수리로 번질 수 있다. 예비품 전략(모듈 spare vs 스위치 spare)과 운영 절차가 근본적으로 달라지므로, CPO 도입은 성능 이득만이 아니라 **운영·가용성 설계**를 함께 바꿔야 한다.

## 추가-3. Packet Spraying은 전송 계층이 전제

per-packet 로드밸런싱은 활용률을 최대화하지만 재정렬을 유발한다. RDMA Write First/Middle/Last 구조는 순서에 의존하므로, **RDMA Write Only(self-contained RETH)** 같은 NIC 지원이나 재정렬 내성 전송(UEC 등)이 없으면 오히려 성능이 급락한다. "spraying = 무조건 좋다"가 아니라 "전송 계층이 out-of-order를 감당하는가"가 조건이다.

## 추가-4. 임계값 순서 하나가 패브릭 안정성을 가른다

DCQCN에서 `xON < WRED Min < WRED Max < xOFF` 순서가 어긋나 xOFF가 ECN보다 먼저 도달하면, 점진적 감속 없이 급정지(Sudden Pause)가 걸려 backpressure·HOL·GPU jitter와 stop-and-go 진동이 생긴다. 그리고 WRED에 **`ecn` 옵션을 빼면 마킹이 아니라 drop으로 동작**해 RDMA에 치명적이다. 혼잡제어는 "켰다/껐다"가 아니라 임계값의 상대 순서와 옵션 디테일이 성패를 가른다.

## 추가-5. rail은 "논리 분리"인지 "물리 분리"인지가 핵심

Multi/Dual-Rail-per-Switch는 rail을 VLAN으로 나눠도 **물리 failure domain은 하나**라, 스위치 한 대 장애가 여러 rail을 죽인다. Single-Rail-per-Switch만 물리 격리를 제공하며 NVIDIA Scalable Unit과 맞는다. 비용을 아끼려 논리 분리를 택하면, 그 대가는 **장애 시 GPU collective 지연·NCCL timeout·job 재시작**이라는 점을 값으로 환산해 판단해야 한다.

## 추가-6. 전력·물·부지가 진짜 병목

GPU를 사도 넣을 곳이 없을 수 있다. 수도권 상시 전력 부족, 3년 재굴착 금지에 따른 인입 대기, 154kV 수전·변전 설비, MW당 ~300평 부지, 데이터홀 1,200~2,400 kg/m² 바닥하중 — 이 물리·행정 제약이 GPU 도입 속도를 실제로 좌우한다. 그리고 냉각이 IT 전력의 35~40%를 먹으므로, 온수 액냉(Free Cooling)으로 PUE를 낮추는 것이 곧 비용·지속가능성 전략이다.

## 추가-7. Day2와의 연결

RDMA·ECN/PFC/DCQCN·ROD/RUD·DLB/ALB/GLB의 개념은 Day2에서 다뤘고, Part 2는 그 **구현 디테일**(RDMA Write 4단계, DCQCN 임계값 순서·헤더·설정, Write First/Middle/Last와 spraying, shared vs dedicated NIC, rail per-switch 설계)로 한 단계 내려간 것이다. 개념(Day2) → 구현·운영(Day3)로 이어 읽으면 된다.
