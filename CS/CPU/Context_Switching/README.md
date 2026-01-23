# Context Switching

CPU가 **현재 실행 중인 실행 단위의 Context를 저장하고, 다른 실행 단위의 상태를 복원하여 CPU를 넘기는 과정** \
기본적으로, OS Scheduling의 실행 단위는 Process가 아니라, **Thread**이다.

<br>

따라서, Thread를 기반으로 Scheduling 및 Context Switch가 진행 되는데, \
이 때 **현재 실행 중인 Thread가 동일한 Process 내의 Thread로 전환하는지, 다른 Process의 Thread로 전환하는 지**에 따라 \
**Switching에 필요한 작업과 overhead가 달라진다.**

<br>

> 💡 Process와 Thread가 각각 다른 Context Switching을 수행하는 것이 아니라 \
> Thread 단위로 수행되지만 **다음 전환된 Thread와의 관계에 따라 수행할 작업의 차이가 있는 것 뿐이다.**

<br>

Context Switching은 다음과 같은 경우에 발생한다.
1. **높은 우선순위 프로세스가 실행 가능 상태**로 준비됨
2. Timer, I/O 등 **Interrupt 발생**
3. System Call 등으로 **User ↔ Kernel mode 간 전환**
    > 항상 Context Switching을 발생 시키는 것은 아니며, 필요할 때 발생
4. 선점형 스케줄링의 경우 **Time Slice(Time Quantum) 만료로 강제 교체**

<br>
<br>

### Process Context Switch

**CPU가 현재 실행 중인 Process의 실행 상태(Context)를 PCB에 저장하고, 다른 Process의 상태를 복원하여 실행을 전환하는 과정**
- CPU는 한 순간에 하나의 작업만 실행할 수 있기 때문에, 여러 Process를 동시에 실행하는 것처럼 보이기 위해 **빠른 전환**이 필요하다.
    > 이러한 CPU 가상화를 구현하는 것이 **Context Switching**
- OS의 Scheduler가 다음으로 실행할 Process를 결정하면, **Dispatcher가 실제로 Context Switching을 수행한다.**

<br>
<br>

### Thread Context Switch

**OS가 하나의 Process 안에서 Thread간의 Context Switching을 수행한다.**
- 실행 중인 Thread의 Context를 TCB에 저장하고, 다른 Thread의 Context를 복원해서 CPU를 넘기는 방식

- Thread들은 하나의 Process에 대해 **같은 가상 메모리 공간을 공유한다.**
    - **Code, Data, Heap 영역을 서로 공유**
        > Stack은 Thread가 각각 하나씩 가진다.
    - Page table 등 **Memory map 전환이 필요 없다.**
    - Register, Thread의 TCB 정보... *TODO*

<br>
<br>

### 다른 프로세스 간 Switch 과정 (p0 → p1)

- 위에서 설명한 상황들 중 하나로 인해 Context Switching이 발생
- CPU와 OS Kernel code를 통해 p0의 Register(PC, SP, …) 값들을 Kernel Stack에 저장
    - CPU가 Kernel mode로 전환될 때, 일부 Register는 자동으로 Kernel Stack에 저장되기도 한다
    - 그러나 모든 Register가 그렇게 저장되진 않으므로, 저장되지 않은 나머지 Register는 Context Switching을 위한 OS Kernel code를 통해 저장된다
    - 결과적으로 CPU와 OS Kernel code의 두 과정을 통해 전체 Context가 Kernel Stack에 저장되는 것
- Kernel Stack에 저장된 데이터 주소를 p0 PCB에 기록
    - 경우에 따라, PCB에 실제로 Register 값을 저장할 수도 있고, Kernel Stack의 주소 값 포인터만 저장할 수도 있다
- PCB에서 p0 Process의 상태 변경
    - RUNNING → READY / BLOCKED / WAITING
    - 만약 READY라면 Run Queue에 넣어둔다
- Scheduler가 다음 실행 후보를 선택
    - OS Scheduling 정책에 따라 선택
    - 여기서 p1이 선택됨
- 선택된 p1 Process의 PCB를 꺼내 CPU의 Register로 복원
- p0과 p1은 다른 Process이므로,
    - 새로운 Page Table Pointer(CR3) 로드
    - TLB flush 진행
- Kernel이 CPU의 SP Register를 p0의 Kernel Stack에서 p1의 Kernel Stack 위치로 교체
- p1 Kernel Stack에 저장된 p1의 Register 데이터들을 CPU Register에 복원
- 복원이 완료되면, p1의 이전 중단 지점으로 실행할 준비 완료

<br>
<br>

### Overhead

- Context Switching 자체는 실제 연산을 하는 것이 아니라 Context 저장 및 복원 작업을 수행한다
- 이 때 CPU Cycle을 사용하지만, 연산 작업이 아니므로 CPU Cycle을 효율적으로 활용하지 못하게 된다
    - 이로 인해 Overhead가 발생

- User mode ↔ Kernel mode 간의 전환이 발생할 때, System Call이나 Interrupt가 발생하게 되고,
- 이 때 기존 진행하던 Process 처리를 멈추고 CPU Cycle을 Mode 전환에 소비하게 된다
    - 이로 인해 Overhead 발생

- 그러나, Context Switch(특히 Process) 발생 시, Mode 전환보다 훨씬 overhead가 크다
- 다음과 같이 Context Switch가 일어나지 않았을 때 하지 않아도 될 일을, Context Switch가 일어날 때에는 해야 하기 때문이다
    - 현재 Process의 Register Context를 저장하고, 다음 Process의 Context를 복원해야 한다
    - 현재 Process의 가상 메모리를 저장하고, 다음 Process의 가상 메모리를 로드해야 한다
    - 다른 Process로 전환될 때, CPU cache를 무효화해야 한다
    - Scheduling을 위해 CPU의 시간과 리소스가 필요하다
    - Process가 I/O 작업을 수행 중일 때, Kernel 호출 및 I/O Device와 상호 작용이 필요하다

- 결과적으로 Overhead가 큰 Context Switch가 자주 발생하면 시스템 성능에 부정적인 영향을 미칠 수 있다

<br>
<br>

----
### 참고
- [Context Switching in Operating System](https://www.geeksforgeeks.org/operating-systems/context-switch-in-operating-system)
- [Context Switching이란?](https://nesoy.github.io/blog/Context-Switching)
- [컨텍스트 스위칭(Context Switching)](https://s7won.tistory.com/11)
- [Thread와 Process의 Context Switching](https://hwan-shell.tistory.com/197)
- [Context Switch가 일어날 때는 왜 overhead가 클까?](https://dev-ellachoi.tistory.com/106)