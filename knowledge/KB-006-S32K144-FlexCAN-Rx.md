---
created: 2026-02-13
last_reviewed: null
review_count: 0
next_review: null
tags: [FlexCAN, CAN-RX, MB, message-buffer, NVIC-iqr, IRQ-trigger-sequence]
---

## S32K144 CAN 통신 RX ERROR 확인 

## 나의 현시점 이해
- 02-13
이제 문제 발생 시 동작하는 sequence 부터 파악을 하고 어떤 단계에서 문제가 생겼는지 파악하는 것이 효율적일 것 같다.
can 수신 할 때 일단 message id를 확인하고 지정된 id 인 경우 mb 에 그 값을 쓸 수 있다.
그리고 나서 flag 를 설정하고 동시에 interrupt 을 관리하는 register 의 해당 mb interrupt 이 enable 된 경우 irq 가 발생하게 된다. vector table 을 확인해서 현재 수신한 mb 와 irq가 매치 되는지 확인이 필요하다. 
시퀀스는 이렇고 왜 이럴가?
rx data를 처리해야 되는데 interrupt 를 써야 되고, 그걸 부르는 방법이 flag고, flag만 설정이 되었다고 해서 interrup 이 호출되는게 아니라 interrupt 를 관리하는 별도의 register 에 설정도 해야 한다. 오동작을 방지하기 위함이라고 판단된다. 
 
---

## S32K144 FlexCAN related registers

### 1. FlexCAN Rx IRQ tirgger sequence 
```
CDS가 message 송신
    ↓
message 수신 (Rx Shifter): 
CAN 버스에 흐르는 신호가 FlexCAN module 의 수신 shifter 로 먼저 들어옵니다. 
    ↓
ID Matching Process : 
수신된 message ID 를 각 MB 에 설정된 ID 와 비교 (이때 마스크 RXIMR 또는 RXMGMASK register 를 참조해서 어떤 bit 를 검사하고 어떤 bit를 무시할 지 결정). Mask bit 가 1인 위치는 수신된 ID 와 MP 의 ID 가 반드시 이치해야 하며 0인 위치는 상관하지 않음.
    ↓
MB3/MB4에 data 저장 : 
위의 조건이 만족되는 경우 MB work2, 3 등에 실제 데이터 값을 씀. 
    ↓
상태 update : 
Data 기록이 완료되면 MB 의 CODE 필드가 0b0100 empty -> 0b0010 full 로 바뀜
    ↓
IFLAG1 bit set  ← 여기가 trigger, software에  수신알림
    ↓
IFLAG1 bit AND IMASK1 bit = 1 이면
    ↓
NVIC로 IRQ 전달 
MB 에 따라 IRQ 번호가 다르므로 반드시 확인 하여 설정 필요 
    ↓
ISR 호출
```
---
### 2. Message Buffer, MB
1. 메시지 버퍼 주소 (FlexCAN0 기준)
S32K144 FlexCAN0의 레지스터 베이스 주소는 **0x4002_4000**이며, 메시지 버퍼는 이 베이스 주소에서 0x0080 오프셋 지점부터 시작됩니다.
• 메시지 버퍼 전체 범위: 0x4002_4080 ~ 0x4002_427F (총 512바이트).
• 개별 MB 시작 주소 (8바이트 페이로드 기준): 각 MB는 16바이트를 차지합니다.
    ◦ MB0: 0x4002_4080
    ◦ MB1: 0x4002_4090
    ◦ MB4: 0x4002_40C0 (데모에서 수신용으로 자주 사용).
    ◦ MBn 주소 계산식: 0x4002_4080 + (n * 16).
2. 메시지 버퍼의 내부 구조 및 역할
하나의 8바이트 메시지 버퍼는 **4개의 32비트 워드(총 16바이트)**로 구성됩니다.


---
워드번호: Word0
offset: 0x0
name: C/S (Control and Status)
role: MB 의 상태 data 길이, message type (CAN/FD) 제어
---
워드번호: Word0
offset: 0x0
name: C/S (Control and Status)
role: MB 의 상태 data 길이, message type (CAN/FD) 제어
오프셋
이름
주요 역할
Word 0
0x0
Control and Status (C/S)
MB의 상태(활성/비활성), 데이터 길이(DLC), 메시지 종류(CAN/FD) 제어.
Word 1
`0x4**
Identifier (ID)
메시지의 고유 ID (표준 11비트 또는 확장 29비트) 설정 및 저장.
Word 2
0x8
Data Bytes 0-3
전송하거나 수신한 실제 데이터의 앞부분 4바이트.
Word 3
0xC
Data Bytes 4-7
전송하거나 수신한 실제 데이터의 뒷부분 4바이트.
---

3. 핵심 비트 필드의 역할
• CODE (Word 0, Bits 27-24): MB의 상태를 결정하는 가장 중요한 필드입니다.
    ◦ 0b0000: Rx 비활성(Inactive).
    ◦ 0b0100: Rx 수신 대기(Empty).
    ◦ 0b0010: Rx 데이터 수신 완료(Full).
    ◦ 0b1100: Tx 데이터 전송 준비 완료.
• DLC (Word 0, Bits 19-16): 전송하거나 수신된 데이터의 바이트 수(0~8)를 나타냅니다.
• IDE (Word 0, Bit 21): ID 형식을 지정합니다 (0: 표준 ID, 1: 확장 ID).
• TIME STAMP (Word 0, Bits 15-0): 메시지가 송수신된 시점의 타이머 값을 기록합니다.
• PRIO (Word 1, Bits 31-29): 송신 시 MB 간의 우선순위를 결정할 때 사용됩니다.
4. 주의사항 (CAN FD 사용 시)
CAN FD를 사용하여 8바이트보다 큰 데이터를 보낼 경우, 각 MB가 차지하는 크기가 24, 40, 72바이트 등으로 늘어납니다. 이 경우 전체 MB 개수가 줄어들며, 개별 MB의 시작 주소(Offset)도 CAN_FDCTRL[MBDSRx] 설정에 따라 재계산되어야 합니다.