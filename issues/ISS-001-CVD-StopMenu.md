---
ID: ISS-001
date: 2025-11-25
status: solved # or pending
category: CVD function
Architecture: all
priority: medium
tags: [system-go, go, go-icon, cvd-menu, run]
flags: user-issue # or self-study
---

### Go button functionality in different menu

#### **1줄 요약**  
1. system option view 에 있는 go 는 target 의 non-debug state 에서 run 시키면서 debug mode 로 진입
2. go icon 과 debug list 의 go 는 debug 상태에서 target 을 run 할 때 사용


---
#### **문제/증상**  

**유저보고:**
- system option view 에 있는 go 를 click 했는데 target 이 run 하지 않아요
- go icon 혹은 debug list 의 go 를 click 하면 run 합니다. 

**CVD 동작:**
- system option view 에 있는 go 는 target 이 non-debug state 에 있을 때 debug mode 진입 + target run 을 시키기 한 option 
    -> system option view 의 SysReset option 이 check 되어 있으면 debug mode 진입 후 target reset 하고 run 
- go icon 과 debug list 의 는 debug state 에서 target 을 run 


---

## ⚡ 즉시 시도할 해결책

### Debug state 에서는 go icon 혹은 debuglist 의 go 사용
---