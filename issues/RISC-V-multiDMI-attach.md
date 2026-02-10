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

### Step 1: software multi dmi 구현 경우 hartid 명시적 지정 (해결률: 90%)

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
riscv read_dmi 0x11  # DMSTATUS 주소
# 출력 예: 0x00400382
```

3. **비트 필드 분석**:
   - `allhalted` (bit 9): 코어가 멈췄는지 여부
   - `allrunning` (bit 10): 코어가 실행 중인지 여부

**패턴 확인**:
- Serial terminal: Core가 halt 상태
- DMSTATUS: `allhalted=0`, `allrunning=1` (불일치!)
→ **HW 레벨 동기화 이슈**

**해결 시도 - 환경별 분기:**

#### A. 환경 확인 먼저:
```bash
help jtag_sel
```

#### B-1. "Command not found" 출력 (일반 SoC/ASIC 환경):
```bash
jtag_reset
sysdown
sysup
attach
```

#### B-2. 명령어 설명 출력 (NXP FPGA 환경):
```bash
jtag_sel 1  # Cluster 1 직접 선택
attach
```

**⚠️ 중요:**
- `jtag_sel`은 **NXP FPGA 검증 환경 전용**
- 일반 ASIC에서는 사용 불가

**결과:**
- ✅ 성공 시: Debug status register 동기화 문제였음 → workaround 적용
- ❌ 여전히 실패 시: Step 3으로 이동

---

### Step 3: HW 팀 에스컬레이션

**Step 1, 2가 모두 실패한 경우에만 필요합니다.**

**문제 가능성**: FPGA 설계 이슈 또는 JTAG TAP 설정 문제

**수집할 정보:**

1. FPGA bitstream 버전
2. JTAG chain 구성 (IR 길이, IDCODE)
3. Secure Word Config 설정값
4. Serial terminal 로그 (실제 core 상태)
5. CVD 로그:
```bash
set debug on
# 위 시나리오 재현 후 로그 수집
```

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
- **Workaround**: `jtag_sel`로 cluster 직접 선택 (NXP FPGA만) 또는 `jtag_reset`

### 3. HW 레이어: JTAG TAP 구조 문제 (1% 케이스)

- Multi-DMI 구조에서 TAP routing 이슈
- Secure Word Config 설정 오류
- **해결**: HW 설계 수정 필요

---

## 환경 조건 (참고용)

이 이슈가 발생하는 환경:

- **SoC 구조**:
    - FPGA 기반 검증 환경
    - Multi-cluster 아키텍처 (ARM Cluster + RISC-V Cluster 0 + RISC-V Cluster 1)
    - JTAG는 Secure Word Config를 통해 RISC-V 영역에 연결됨

- **DMI 구조**:
    - DMI 0: RISC-V Cluster 0 (hartid 0~7)
    - DMI 1: RISC-V Cluster 1 (hartid 8~15)

- **디버거**: CVD-RISC-V 5.0+
- **JTAG 연결**: 정상 (IDCODE 읽기 성공, Secure Word Config 설정 완료)
- **고객**: NXP (FPGA validation)

---

## 기술적 배경

### RISC-V Debug Spec (0.13+)

- 각 DMI는 독립적인 **hartid 네임스페이스** 보유
- `DMCONTROL` 레지스터의 `hartsel` 필드로 타겟 코어 선택
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

### Debug Status Register 동기화

- FPGA 환경에서 **비동기 clock domain** 간 레지스터 업데이트 지연 발생 가능
- 일부 SoC는 debug module과 core 사이에 **2~3 cycle latency** 존재
- CVD는 즉시 레지스터 읽기 → 타이밍 이슈 발생

---

## 관련 CVD 명령어

```bash
# hartid 설정 (Step 1)
system.config hartindex 8 9 10 11 12 13 14 15

# 환경 확인 (Step 2)
help jtag_sel

# Cluster 직접 선택 (NXP FPGA 환경 전용, Step 2)
jtag_sel 1

# JTAG reset (일반 환경, Step 2)
jtag_reset

# Debug 레지스터 읽기 (Step 2)
riscv read_dmi 0x11

# 현재 연결된 코어 목록
info cores

# 디버그 로그 활성화 (Step 3)
set debug on
```

---

## Known Limitations

1. **CVD 버전 의존성**:
    - CVD 4.x: `system.config hartindex` 미지원
    - → CVD 5.0+ 업그레이드 필요

2. **FPGA 환경 특수성**:
    - `jtag_sel` 명령어는 **NXP FPGA 검증 환경 전용**
    - 실제 SoC에서는 사용 불가 (ASIC은 다른 메커니즘 사용)

3. **HW 설계 의존성**:
    - Debug status register 동기화는 SoC 설계에 따라 다름
    - 일부 칩은 JTAG reset 후 재시도 필요

---

## 참고 자료

- RISC-V Debug Spec 0.13.2 - Section 3.14 (hartsel 필드)
- CVD 명령어 레퍼런스: `help riscv`, `help system.config`
- NXP FPGA 검증 환경 매뉴얼 (내부 문서)

---

## 이슈 히스토리

- **2025-01-29**: 초기 작성 (NXP 고객 케이스 기반)
- **2025-02-02**: 구조 개선 (해결책 우선 배치, 해결률 추가)
- 해결 성공률: Step 1 (90%), Step 2 (9%), Step 3 에스컬레이션 (1%)

---

## AI 진단 제안 (메타 정보)

**이 이슈를 AI가 판단할 때:**

1. **즉시 Step 1 해결책 제시**
   - 환경 확인 질문 하지 말 것
   - CVD 버전은 Note로 언급

2. **Step 1 실패 시에만 Step 2 제시**
   - 환경별 분기 명확히 (`help jtag_sel`)

3. **jtag_sel 언급 시 반드시:**
   - 환경 확인 방법 먼저
   - NXP FPGA 전용임을 명시
   - 일반 환경의 대안 제공

**AI가 제시할 1차 조치:**
```
즉시 시도:
```bash
sysdown
system.config hartindex 8 9 10 11 12 13 14 15
sysup
attach
```

이것으로 90% 해결됩니다.
실패하면 알려주세요 (Step 2 안내)
