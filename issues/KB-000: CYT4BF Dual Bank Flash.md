---
ID: KB-000
date: 2026-02-03
status: solved  # or pending
category: flash 
priority: high  # high/medium/low
tags: [flash, cyt4bf, traveo, dual bank, cyt]
flags: self-study # or user-issue
---

### CYT4BF Dual Bank Flash: Map B 선택 및 프로그래밍 방법

#### **1줄 요약**  
CYT4BF는 리셋 후 기본 Single Bank (Map A)로 부팅되며, Map B 실행 또는 프로그래밍을 위해선 **Dual Bank Enable** 및 **Bank Swap** 레지스터 설정 필요.

#### **문제**  
- CYT4BF에서 Dual Bank Mode 사용 시, **리셋 후 항상 Map A**(Single Bank 기준)  
- Map B 영역을 실행하거나 프로그래밍하려면 추가 설정이 필요하지만, 관련 레지스터 및 주소 정보가 문서에서 흩어져 있음

#### **원인**  
CYT4BF의 flash controller는 다음 두 레지스터로 bank 동작 제어:
- **FLASHC_FLASH_CTL register**: Dual Bank Mode 활성화 여부 설정  
- **FLASHC_BK_SWAP register**: 현재 active bank를 Map A ↔ Map B로 전환  

리셋 직후 이들 레지스터는 기본값(= Dual Bank 비활성, Map A 고정)으로 초기화됨.

#### **해결**  
1. **Dual Bank Mode 활성화**:
bash
```
	Data.Set AD:0x40240000 %Long 0x00111000  ;//Enable dual bank mode for 0x1200 0000
	Data.Set AD:0x40240000 %Long 0x00111100  ;//Enable bank swap
```
Flash 프로그래밍 방법 선택:
✅ 방법 A (레지스터 활용):
   Bank swap 후 동일 물리 주소(0x10000000～)에 프로그램 → 실제 Map B 영역에 기록
✅ 방법 B (직접 접근):
   Map B 물리 주소에 직접 프로그램 → swap 불필요

💡 코멘트: 레지스터 설정 없이 Map B에 직접 쓰는 것도 가능하지만 CVD flash programming 은 bankswap 방식 사용. 
