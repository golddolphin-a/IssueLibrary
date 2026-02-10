---
ID: ISS-002
date: 2026-02-03
status: solved # or pending
category: CVD function
Architecture: all
priority: medium
tags: [memory read, timeout, slow clock, data dump]
flags: user-issue # or self-study
---

### Memory Read Timeout with Slow Target Clock

#### **1줄 요약**  
CVD 디버거에서 low JTAG clock인 경우 memory read 시간이 길어져 response timeout으로 인해 CVD가 연결을 끊어버리는 문제. 필요한 영역만 dump하거나 timeout/segment size 조절로 해결.

---
#### **문제/증상**  

**발생 조건:**
- Target clock: 0.05 이하 (매우 느린 클럭)
- Debug mode 진입: 정상
- Memory read 시도: Timeout 발생

**에러 메시지 예시:**
```
Error - Memory read failed from 0x[ADDRESS] - unknown error (0x00000002)
```
```
Fail, Target does not system down !
```

## 🔍 에러 메시지별 이해

### "Memory read failed - unknown error (0x00000002)"
- 일반적인 timeout 에러
- 1차/2차 해결책 적용

### "Fail, Target does not system down!"
**의미:**
- Memory read가 timeout으로 먼저 실패
- CVD가 cleanup을 위해 debug mode 종료(sysdown) 시도
- 하지만 target이 이미 정상 상태이거나 state 불일치
- Sysdown 명령 실패로 인한 2차 에러 메시지

**대응:**
- 이 메시지는 근본 원인이 아닌 **결과**
- 실제 원인: Memory read timeout
- **동일하게 1차/2차 해결책 적용**

---

## ⚡ 즉시 시도할 해결책

### 1차 해결책: 최소 범위 Memory Dump (가장 효과적)

**핵심 원리:**
- 전체 메모리 읽기 대신 **필요한 주소만 선택적으로 dump**
- 전송 데이터량 최소화 → timeout 위험 감소

**명령어 형식:**
```
data.dump <memory_class>:<address> | <range> /format
```

**실전 예시:**
```bash
# 특정 주소 범위만 읽기 (0x0부터 0x1F까지)
data.dump EAPB:0x0++0x1f /long

# Word 단위로 읽기
data.dump UM:0x0--0x1f /word

# 단일 주소만 확인
data.dump EAPB:0x1000 /long
```

**사용 시나리오:**
- 특정 레지스터 값만 확인하고 싶을 때
- 구조체 일부 필드만 필요할 때
- Large memory 영역 중 관심 영역이 명확할 때

**장점:**
- Timeout 설정과 무관하게 빠르게 성공
- CVD 사용성 저하 없음

---

### 2차 해결책: Response Timeout + Memory Segment Size 조절

**적용 시점:**
- 넓은 메모리 영역을 반드시 읽어야 하는 경우
- 1차 해결책으로 충분하지 않을 때

#### 2-1. Response Timeout 증가

**설정 방법:**
```
DEBUG.ResponseTimeOut <value_in_ms>
```

**GUI 설정:**
1. CVD 메뉴: `Menu -> Config -> Debugger`
2. Response Timeout 항목 확인 및 수정

**권장값:**
- Target clock 0.05 이하: 10000ms (10초)부터 시작
- 실패 시 점진적 증가: 20000ms, 30000ms...
- 최대값: 10000000ms (약 2.7시간) - **실용적이지 않음**

**주의사항:**
- 너무 큰 값(예: 10000000ms)으로 설정 시:
  - Memory read는 성공하지만
  - **CVD가 너무 느려져 사실상 사용 불가능**
  - 적절한 균형점 찾기 필요

#### 2-2. Memory Segment Size 조정

**설정 방법:**
```
OPTION.MemorySegmentSize S<size_in_bytes>
```
**실전 예시:**
```bash
# CVD 명령어
DEBUG.MemorySegmentSize S<segmentsize>
# segment size : 512(default), 1024, 2048, 4096, 8192
```

**동작 원리:**
- CVD가 한 번에 전송하는 메모리 블록 크기 제어
- Slow clock 환경: 큰 segment = 단일 transaction 시간 과다 → 실패
- 작은 segment = 안정성 향상 (단, 전송 횟수 증가)

**권장값:**
- Target clock 0.05 이하: 512 또는 256 bytes부터 시작
- Default가 큰 경우 점진적으로 감소 테스트

**Trade-off:**
- ✅ 안정성 향상 (timeout 회피)
- ❌ 전체 전송 시간 증가 (오버헤드)
- **Slow clock 환경: "느리지만 성공" > "빠르지만 실패"**

---

## 🌳 트러블슈팅 트리

```
Target clock 0.05 이하 & Memory read timeout 발생
    ↓
[1차] 필요한 주소만 data.dump로 선택적 읽기
    ↓
    ├─ 성공 → 문제 해결 ✅
    └─ 넓은 영역 필수 → [2차]로 이동
        ↓
    [2차-1] Response Timeout 증가 (10000ms부터)
        ↓
        ├─ 성공하지만 너무 느림 → Segment size 조정
        └─ 여전히 실패 → Timeout 추가 증가
            ↓
        [2차-2] Memory Segment Size 감소 (512 → 256)
            ↓
            ├─ 성공 → 문제 해결 ✅
            └─ 지속 실패 → 에스컬레이션 📧
```

---

## 🚨 에스컬레이션 기준

**메일 문의가 필요한 경우:**
1. 1차 해결책으로 필요한 데이터 접근 불가능
2. 2차 해결책 모두 시도했으나 (Timeout 30000ms+, Segment 256 이하) 지속 실패
3. 다른 memory address/class에서도 동일 증상

**메일 문의 시 필수 정보:**
- [ ] CVD 버전
- [ ] Firmware 버전  
- [ ] Target mode (sysup / attach / prepare)
- [ ] Target clock 정확한 값
- [ ] 시도한 timeout 값들
- [ ] 시도한 segment size 값들
- [ ] 실패한 memory address/class 범위
- [ ] 사용한 data.dump 명령어 예시

---

## 🔗 관련 문서

- CVD Script Reference Manual: 
  - `DEBUG.ResponseTimeOut` 명령어
  - `OPTION.MemorySegmentSize` 명령어
  - `data.dump` 명령어
- JTAG-RISCV-002.md: JTAG clock 설정 관련

---

## 💡 추가 팁

**Memory class 종류:**
- `EAPB`: External APB bus
- `UM`: User Memory
- 기타 target-specific memory class는 CVD 문서 참조

**Format 옵션:**
- `/long`: 32-bit (4 bytes)
- `/word`: 16-bit (2 bytes)
- `/byte`: 8-bit (1 byte)

**범위 지정 방식:**
- `++`: 시작 주소 + 크기 (예: `0x0++0x1f` = 0x0부터 31 bytes)
- `--`: 시작 주소 -- 끝 주소 (예: `0x0--0x1f` = 0x0부터 0x1f까지)
