---
{"dg-publish":true,"permalink":"/CS/운영체제/OS_04_Operating_Systems/","created":"2025-04-09T20:00:55.231+09:00"}
---

# Contents
- ## 프로그램이 만들어지는 과정 
- ## Process State
- ## Process Control Block
- ## 프로세스 생성 관리
---
# 1. 프로그램이 만들어지는 과정
## [[CS/시스템 보안/ELF 파일 구조-1\|ELF 파일 구조-1]]
---
# 2. Process State
## Porcess State
- ### New : the process is being created
	- process가 막 생성된 상태 (`fork()`등으로 생성)
- ### Running : Instructions are being executed
	- CPU를 점유해 명령어를 실행 중인 상태
- ### Waiting : the process is waiting for some event to occur
	- ### (e.g., an I/O completion or reception of a signal)
	- I/O 등 외부 이벤트가 끝나길 기다리는 상태 (CPU 양보 상태)
		- I/O를 위해 CPU를 자발적으로 양보한다.
		- I/O가 끝날때까지 기다린다.
- ### Ready : the process is waiting to be assigned to a processor
	- CPU 할당을 기다리는 상태 (I/O 작업은 없음)
- ### Terminated : the process has finished execution
	- 실행이 종료되어 자원이 회수된 상태 (`exit()` 또는 `return`)
---
## 프로세스 상태 다이어그램
### 커널 내에 Ready Queue, Waiting Queue, Running Queue 를 두고 프로세스들을 상태에 따라 관리한다.
![](https://i.imgur.com/juLGN6z.png)
1. **`fork()`를 통해 새로운 프로세스가 생성되면** → `New` 상태로 진입한다.
2. **CPU 할당을 기다리기 위해** → `Ready` 상태로 이동하며, **Ready Queue에 삽입**된다.
3. **스케줄러가 CPU를 할당하면** → `Running` 상태로 전환되어 **명령어를 실행**한다.
4. CPU 스케줄링에 따라 **CPU를 너무 오래 점유하면** → 인터럽트가 발생하여 다시 `Ready` 상태로 이동된다.
5. **I/O 작업이 필요하면**, 프로세스는 자발적으로 CPU를 반납하고 `Waiting` 상태로 전환된다.  (예: 디스크 읽기, 키보드 입력 등)
6. **I/O 작업이 완료되면** → 다시 `Ready` 상태로 돌아가 **CPU 할당을 기다린다.**
7. **다시 스케줄러에 의해 CPU를 할당받으면**(`scheduler dispatch`) → `Running` 상태가 되어 작업을 계속한다.
8. **작업이 모두 끝나거나**, `exit()` 혹은 `main()` 함수 종료 시 → `Terminated` 상태로 진입하고,  운영체제가 해당 프로세스의 **메모리와 자원을 회수**한다.

### **New → Ready → Running → (Waiting ↔ Ready) → Running → Terminated**
---
# 3. Process Control Block
## PCB(Process Control Block)
  ![left|300](https://i.imgur.com/HTF6Yq4.png)
- ### 운영체제가 프로세스를 관리하기 위해 유지하는 정보들의 구조체
	-  각 프로세스는 고유한 PCB를 하나씩 가지며,  운영체제는 이를 통해 프로세스의 상태를 추적하고 제어한다.
	- 운영체제는 이 PCB들을 큐(Ready Queue, Waiting Queue 등)로 관리하면서  **스케줄링, Context Switch, 자원 할당 등을 수행**한다.

| 항목                            | 설명                                                     |
| ----------------------------- | ------------------------------------------------------ |
| **Process State**             | 현재 프로세스의 상태 (New, Ready, Running, Waiting, Terminated) |
| **Program Counter**           | 다음에 실행할 명령어의 메모리 주소 → CPU가 해당 위치에서 명령어를 fetch한다.       |
| **CPU Registers**([[CS/시스템 보안/메모리_구조\|메모리_구조]]) | 실행 중이던 레지스터 값 (레지스터 스냅샷 저장) → Context Switch 시 필요 하다.  |
| **CPU Scheduling Info**       | 우선순위, 스케줄링 큐 포인터 등                                     |
| **Memory Management Info**    | 페이지 테이블, 세그먼트 테이블 등                                    |
| **Accounting Info**           | 어떤 user가 creation 했는지 등                                |
| **I/O Status Info**           | 연관된 I/O 장치, 어떤 자원을 열었고 lock을 걸었는지 등                    |

---
## Program Counter
![](https://i.imgur.com/e9sLrb6.png)
- ### 프로그램 실행 중 **다음에 실행할 명령어의 주소**를 담고 있는 레지스터  
	1. **Fetch(가져오기) : 다음에 실행할 명령어를 메모리에서 읽어온다.**
		- Program Counter(PC)는 다음에 실행할 명령어의 메모리 주소를 가리킨다.
		- 해당 주소의 명령어를 메모리에서 가져와서(Fetch),  Instruction Queue(IQ)에 저장한다.
	2. **Decode(해석하기) : 무슨 명령인지 해석해서 실행을 준비한다.**
		- IQ에 저장된 명령어를 분석하여 어떤 연산인지 파악한다.
	3. **Fetch Operands(피연산자 가져오기) : 실제로 연산할 값들을 메모리/레지스터에서 읽어온다.**
		- 명령어가 참조하는 값들을 메모리나 레지스터에서 불러온다.
	4. **Execute(실행) : 명령어에 따라 연산을 수행한다.**
		- 산술/논리 연산, 메모리 접근, 분기 등 연산 유닛(ALU 등)이 동작한다.
	5. **Store Output(결과 저장) : 연산 결과를 저장한다.**
		- 연산 결과를 레지스터 또는 메모리에 저장한다.
- ### 이 사이클은 각 명령어 단위로 반복되며,  Program Counter는 다음 명령어 주소를 가리키도록 자동으로 증가한다.
---
## Process Concept
- ### Process
	- a program that performs a **single thread of execution**
	- process는 한번에 하나의 일만 수행할 수 있다.
	- 현대의 운영체제는 여러개의 일을 동시에 실행(멀티 프로세싱)할 수 있다.![left|400](https://i.imgur.com/RKvMokC.png)
		1. Kernel이 P0의 PCB0를 참조해서 PC, Register 정보 등을 CPU에 로드한다.
		2. interrupt나 I/O 요청이 발생하면 현재 PC,  Register정보 등을 PCB0에 저장한다.(**Context Save**)
		3. 다음에 실행될 process인 P1의 PCB를 참조해서 상태를 CPU에 복원한다.(**Context Restore**)
---
## Context Switch
- ### Context
	- the context of a process is represented in the **PCB**
	- 모든 프로세스는 코드, 데이터, 스택의 주소 공간과 프로세스가 동작할 때의 레지스터 값 등의 ==context==를 가지고 있다.
- ### Context Switch
	- Process가 바뀌는 과정
	- when CPU switches to a new process, kernel must ==save the state of the old process== and ==load the saved stated for the new process==![](https://i.imgur.com/x9WUdUR.png)
		- Context switching time is **overhead**
			- 시스템은 switching하는동안 아무것도 할 수 없다.
		- Context switching time은 hardware support(scheduling을 얼마나 잘하는가)에 의존된다.
---
# 4. Process Creation
## process creation이란?
- ### process descriptor
- ### Linux Task(Process Task, Thread Task)
- ### MultiProcess VS Multithread
- ### Context Switching(SW / HW)
	- #### S/W 방식![](https://i.imgur.com/7uZAPTP.png)
	- #### H/W 방식![](https://i.imgur.com/SUBIo6U.png)
## fork()
## execve()
## Process Termination