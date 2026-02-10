---
ID: ISS-004
category: connect
satus: solved #or pending
Architecture: RISC-V
priority: high
tags: [multi-dmi, cluster, hartid, attach-fail, FPGA, debug-status-register]
flags: user issue #or self-study
---

### RISC-V Multi-DMI 환경에서 Cluster 1 코어 attach 실패
#### **1줄 요약**  
- multi DMI 에서 cluster 1 core 연결 시 `system.config hartindex` command 로 설정 후 연결 가능. 
- jtag-sel (hardware switch) 로 multi dmi의 core 선택 시 `system.config hartindex` command 필요없음.
  jtag-sel 로 individual dmi 선택 하기 때문. 이런 design 에서 연결이 안될 경우 target 구현문제일 가능성 큼

## 증상

- **유저 보고**: "Cluster 1 attach fail"
- **CVD 동작**:
    - Cluster 0 코어는 정상 연결
    - RISC-V Cluster 1 코어 attach 시도 시 **연결 자체가 안 됨**
    - CVD는 코어 상태를 "Running"으로 표시하며 attach 실패 or sysdown 
---

## 즉시 시도할 해결책

### Step 1: hartid 명시적 지정 (해결률: 90%)

**실패 시 step 2.**

CVD는 초기화 시 DMI 0 영역의 코어만 자동 인식합니다. Cluster 1 (DMI 1)의 hartid를 수동으로 지정해야 합니다.

**시도:**
```bash
# 명령어
system.config hartindex <parameter>

#사용 예제 
sysdown
system.config hartindex 8 9 10 11 12 13 14 15
sysup

```

---

### Step 2: 디버그 레지스터 동기화 확인 

**문제 가능성**: Hardware debug status register 동기화 지연/버그

**진단 방법:**: dmcontrol register 로 hart halt 후 terminal 에서 hart 상태 확인 

1. **debug module register 로 hart halt request **:
   - 
3. **Serial Terminal로 실제 코어 상태 확인**:
   - UART console 또는 FPGA 내부 디버그 레지스터 직접 읽기

4. **CVD에서 DMSTATUS 레지스터 읽기**:
```bash
riscv read_dmi 0x43000110  # DMSTATUS 주소
# 출력 예: 0x000F0382
```

3. **비트 필드 분석**:
   - `allhalted`,`anyhalted` (bit 8-9): 코어가 멈췄는지 여부
   - `allrunning`,`anyrunning` (bit 10-11): 코어가 실행 중인지 여부

**패턴 확인**:
- Serial terminal: Core가 halt 상태
- DMSTATUS: `allrunning=1` (불일치!)
→ **HW 레벨 동기화 이슈** : target 이 halt 되었으나 cvd 가 읽어 온 값은 run 상태.

**해결 방안 **
- Target system 설정 확인 필요. 실제 core 상태를 정확하게 반영하는지.
- CVD 에서 할 수 있는건 없음. 사용자의 target 의 system 설계 담당자 검토 필요

#### A. 환경 확인 먼저:
```bash
help jtag_sel
```

---

### Step 3: HW 팀 에스컬레이션

**Step 1, 2가 모두 실패한 경우에만 필요합니다.**


**수집할 정보:**

1. CVD & Firmware version
2. prepare mode 에서 0x43000000 (debug module register) dump 값
3. target mode (sysup/attach/prepare)


**보고 대상**: HW 설계 팀 또는 FPGA 검증 팀

---

## 근본 원인 (Root Cause)

이 이슈는 **3가지 레이어**에서 발생할 수 있습니다:

### 1. SW 레이어: CVD 설정 문제 (90% 케이스)

- CVD는 초기화 시 DMI 0의 hartsel 필드만 스캔
- Cluster 1의 hartid를 수동으로 지정하지 않으면 인식 불가
- **해결**: `system.config hartindex` 사용

### 2. HW-SW 인터페이스: Debug Register 동기화 지연 (9% 케이스)

- FPGA 환경에서 clock domain crossing 타이밍 이슈 가능
- Core는 실제로 halt되었지만, DMSTATUS 업데이트 지연됨
- CVD는 레지스터 값만 보고 "Running"이라고 잘못 판단

### 3. HW 레이어: JTAG TAP 구조 문제 (1% 케이스)

- Multi-DMI 구조에서 TAP routing 이슈
- Secure Word Config 설정 오류
- **해결**: HW 설계 수정 필요

---


## 기술적 배경

### RISC-V Debug Spec (0.13+)

- 각 DMI는 독립적인 **hartid 네임스페이스** 보유
- `DMCONTROL` 레지스터의 `hartsello` 필드로 타겟 코어 선택
- CVD는 부팅 시 **DMI 0의 hartsellen 비트**로 최대 hartid 계산
→ 다른 DMI는 자동 감지 안 됨

### Multi-DMI 아키텍처

```
JTAG ─→ Secure Word Config ─┬─→ ARM Cluster (접근 차단)
                             │
                             ├─→ RISC-V Cluster 0 (DMI 0)
                             │   hartid 0~7
                             │
                             └─→ RISC-V Cluster 1 (DMI 1)
                                 hartid 8~15
```

---

## 관련 CVD 명령어

```bash
# hartid 설정 (Step 1)
system.config hartindex 8 9 10 11 12 13 14 15

# Debug 레지스터 읽기 (Step 2)
data.dump edgb:0x43000100 #dmcontrol register
data.dump edgb:0x43000110 #dmstatus register

```


---

## 참고 자료

- RISC-V Debug Spec 0.13.2 - Section 3.14.1 (Debug Module Status)


---

## 이슈 히스토리

- **2025-01-29**: 초기 작성 (NXP 고객 케이스 기반)
- **2025-02-02**: 구조 개선 (해결책 우선 배치, 해결률 추가)
- 해결 성공률: Step 1 (90%), Step 2 (9%), Step 3 에스컬레이션 (1%)

---

