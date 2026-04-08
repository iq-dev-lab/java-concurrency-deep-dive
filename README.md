<div align="center">

# ⚙️ Java Concurrency Deep Dive

**"synchronized를 쓰는 것과, JVM이 Object Header의 Mark Word를 어떻게 바꾸며 락을 확장하는지 아는 것은 다르다"**

<br/>

> *"Virtual Thread를 켜면 성능이 좋아지겠지 — 와 — Pinning이 왜 발생하고, Continuation이 캐리어 스레드에서 언마운트되는 과정과, synchronized 안에서 블로킹이 왜 문제인지 아는 것의 차이를 만드는 레포"*

JVM이 Thin Lock을 Fat Lock으로 확장하는 과정, CAS가 `LOCK CMPXCHG` CPU 명령어 수준에서 원자성을 보장하는 원리, Virtual Thread의 Continuation이 힙에 저장되고 ForkJoinPool 캐리어 스레드 위에서 스케줄링되는 방식, JMM의 happens-before가 실제로 무엇을 보장하고 volatile만으로 충분하지 않은 케이스까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Java 동시성의 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-21-ED8B00?style=flat-square&logo=openjdk&logoColor=white)](https://openjdk.org/projects/jdk/21/)
[![Virtual Thread](https://img.shields.io/badge/Virtual_Thread-JEP_444-5382a1?style=flat-square&logo=openjdk&logoColor=white)](https://openjdk.org/jeps/444)
[![JMH](https://img.shields.io/badge/JMH-Benchmark-orange?style=flat-square)](https://github.com/openjdk/jmh)
[![Docs](https://img.shields.io/badge/Docs-40개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Java 동시성에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`synchronized`를 쓰면 스레드 안전합니다" | Thin Lock → Fat Lock 확장 과정, Object Header Mark Word가 어떻게 바뀌는지, JIT가 Lock Elision으로 락을 제거하는 조건 |
| "`volatile`을 쓰면 가시성이 보장됩니다" | CPU L1/L2/L3 캐시와 Write Buffer가 가시성 문제를 일으키는 원리, volatile이 메모리 펜스를 삽입하는 방식, volatile만으로 충분하지 않은 check-then-act 케이스 |
| "Virtual Thread를 쓰면 처리량이 늘어납니다" | Continuation 객체가 힙에 실행 상태를 저장하는 방식, 블로킹 I/O 시 캐리어 스레드에서 언마운트되는 과정, synchronized Pinning이 왜 발생하고 어떻게 진단하는가 |
| "`AtomicInteger`가 `synchronized`보다 빠릅니다" | `LOCK CMPXCHG` CPU 명령어로 원자성을 보장하는 원리, ABA 문제와 ABA 해결, `LongAdder`가 Cell 분산으로 고경쟁 환경에서 CAS 실패를 줄이는 스트라이프 기법 |
| "`ConcurrentHashMap`을 쓰세요" | Java 7 세그먼트 락 → Java 8 버킷 레벨 `synchronized` + CAS 진화, `computeIfAbsent` 내부 동작, 쓰기 경합이 없는 버킷 읽기가 락 없이 안전한 이유 |
| "데드락이 발생했어요" | Thread Dump에서 데드락 4가지 조건을 찾는 방법, `jstack` / `jcmd` 실전 분석, Lock Contention을 JFR과 async-profiler로 측정하는 방법 |
| 이론 나열 | 재현 가능한 JMH 벤치마크 + JVM 플래그(`-Djdk.tracePinnedThreads=full`, `-XX:+PrintAssembly`) + Docker Compose 실험 환경 + Spring 연결 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Ch1](https://img.shields.io/badge/🔹_Ch1-OS_스레드와_Java_스레드-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./os-threads-and-java-threads/01-os-thread-model.md)
[![Ch2](https://img.shields.io/badge/🔹_Ch2-Java_Memory_Model-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./java-memory-model/01-hardware-memory-model.md)
[![Ch3](https://img.shields.io/badge/🔹_Ch3-락_내부_구현-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./lock-internals/01-object-header-mark-word.md)
[![Ch4](https://img.shields.io/badge/🔹_Ch4-Lock--Free_알고리즘-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./lock-free-algorithms/01-cas-cpu-level.md)
[![Ch5](https://img.shields.io/badge/🔹_Ch5-동시성_자료구조-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./concurrent-data-structures/01-concurrent-hashmap.md)
[![Ch6](https://img.shields.io/badge/🔹_Ch6-Virtual_Thread-5382a1?style=for-the-badge&logo=openjdk&logoColor=white)](./virtual-threads/01-why-virtual-thread.md)
[![Ch7](https://img.shields.io/badge/🔹_Ch7-실전_패턴과_장애_분석-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./real-world-patterns/01-deadlock-analysis.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: OS 스레드와 Java 스레드 — 기반 원리

> **핵심 질문:** Java 스레드는 OS 스레드와 어떻게 연결되는가? 스레드를 만드는 비용은 얼마나 되고, ThreadPoolExecutor 내부에서 스레드는 어떻게 관리되는가?

<details>
<summary><b>OS 스레드 1:1 매핑부터 데몬 스레드와 JVM 종료까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. OS 스레드 모델 — 커널 스레드와 1:1 매핑 비용](./os-threads-and-java-threads/01-os-thread-model.md) | 커널 스레드와 사용자 스레드의 차이, Java 스레드가 OS 스레드에 1:1 매핑되는 이유와 그 비용(1MB 스택, OS 자원), `strace`로 `new Thread()` 시 `clone()` 시스템콜을 관찰하는 실험, Virtual Thread가 왜 이 구조를 탈피했는가 |
| [02. 컨텍스트 스위칭 비용의 실체 — 레지스터·TLB·CPU 캐시](./os-threads-and-java-threads/02-context-switching-cost.md) | 컨텍스트 스위칭 시 레지스터 저장/복원, TLB 플러시, CPU 캐시(L1/L2) 무효화가 처리량에 미치는 영향, 자발적 vs 비자발적 컨텍스트 스위칭 차이, JMH로 스레드 수 증가에 따른 처리량 저하를 측정하는 벤치마크 |
| [03. 스레드 생성과 Thread Pool — ThreadPoolExecutor 내부](./os-threads-and-java-threads/03-thread-pool-executor.md) | `new Thread()` 생성 비용, `ThreadPoolExecutor`의 corePool / maxPool / 작업 큐 / RejectedExecutionHandler 내부 동작, `CallerRunsPolicy`가 백프레셔를 구현하는 방식, `ForkJoinPool`과의 차이 |
| [04. 스레드 상태 머신 — 6가지 상태 전환 조건](./os-threads-and-java-threads/04-thread-state-machine.md) | `NEW` / `RUNNABLE` / `WAITING` / `TIMED_WAITING` / `BLOCKED` / `TERMINATED` 6가지 상태 전환 조건과 원인, `Thread.sleep()` vs `Object.wait()` vs `LockSupport.park()` 의 차이, `jstack`으로 각 상태를 실시간으로 관찰하는 방법 |
| [05. 데몬 스레드와 JVM 종료 — ShutdownHook 실행 보장](./os-threads-and-java-threads/05-daemon-thread-jvm-shutdown.md) | JVM이 종료되는 조건(모든 non-daemon 스레드 완료), 데몬 스레드가 강제 종료되는 방식과 리소스 정리 문제, `Runtime.addShutdownHook()`이 실행되는 타이밍과 보장 범위, Graceful Shutdown 설계 패턴 |

</details>

<br/>

### 🔹 Chapter 2: Java Memory Model — 가시성과 순서의 보장

> **핵심 질문:** CPU 캐시와 Write Buffer가 왜 가시성 문제를 일으키는가? happens-before는 실제로 무엇을 보장하고, volatile만으로 충분하지 않은 경우는 언제인가?

<details>
<summary><b>하드웨어 메모리 모델부터 Double-Checked Locking 함정까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 하드웨어 메모리 모델 — CPU 캐시와 Write Buffer](./java-memory-model/01-hardware-memory-model.md) | CPU L1/L2/L3 캐시 계층 구조, Write Buffer와 스토어 포워딩이 가시성 문제를 일으키는 원리, MESI 캐시 일관성 프로토콜, 멀티코어에서 같은 변수를 다른 코어가 다른 값으로 보는 상황 재현 실험 |
| [02. JMM — 주 메모리와 작업 메모리 추상화](./java-memory-model/02-jmm-main-working-memory.md) | JVM이 하드웨어 위에 만드는 주 메모리 / 작업 메모리 추상화 모델, JMM이 정의하는 8가지 원자적 연산(read/load/use/assign/store/write/lock/unlock), JMM이 하드웨어를 추상화하면서 생기는 허용 재정렬 범위 |
| [03. happens-before 관계 — 8가지 규칙과 전이적 특성](./java-memory-model/03-happens-before.md) | 프로그램 순서 규칙, 모니터 락 규칙, volatile 규칙, 스레드 시작/종료 규칙, `interrupt()` 규칙 등 8가지 happens-before 규칙 상세 분석, 전이적 특성으로 인해 의도치 않게 보장이 연결되는 케이스, `volatile` 쓰기 전의 모든 쓰기가 왜 보이는가 |
| [04. volatile 완전 분해 — 메모리 펜스와 한계](./java-memory-model/04-volatile-deep-dive.md) | `volatile` 읽기/쓰기 시 삽입되는 LoadLoad / LoadStore / StoreLoad / StoreStore 메모리 펜스, `-XX:+PrintAssembly`로 어셈블리 코드에서 `MFENCE` / `LOCK ADDL` 확인, volatile만으로 충분하지 않은 check-then-act 케이스와 해결책 |
| [05. 명령어 재정렬 — 컴파일러·CPU·JIT의 3단계 재정렬](./java-memory-model/05-instruction-reordering.md) | 컴파일러 재정렬(JIT 최적화), CPU Out-of-Order Execution, CPU 투기적 실행이 관찰 순서를 바꾸는 방식, `@Contended`로 False Sharing을 방지하는 원리, Dekker 알고리즘이 x86에서 의도대로 동작하지 않는 이유 |
| [06. Double-Checked Locking의 함정 — Java 5 이전과 이후](./java-memory-model/06-double-checked-locking.md) | Java 5 이전 DCL이 왜 깨지는가(부분 초기화된 객체를 다른 스레드가 보는 경위), `volatile` 추가가 어떻게 수정하는가, `Initialization-on-demand Holder` 패턴으로 완전히 회피하는 방법, JVM 클래스 초기화 락의 보장 범위 |

</details>

<br/>

### 🔹 Chapter 3: 락의 내부 구현 — synchronized에서 StampedLock까지

> **핵심 질문:** synchronized는 Object Header를 어떻게 바꾸며 락을 확장하는가? AQS 대기 큐는 어떻게 구현되고, StampedLock의 낙관적 읽기는 어떻게 동작하는가?

<details>
<summary><b>Object Header Mark Word부터 락 성능 벤치마크까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Object Header와 Mark Word — 락 상태 저장 구조](./lock-internals/01-object-header-mark-word.md) | 모든 Java 객체가 가진 Object Header의 Mark Word(8바이트) 구조, Thin / Fat Lock 상태별 Mark Word 비트 레이아웃, `jol-core`(Java Object Layout)로 Object Header를 직접 출력하는 실험, 락 상태 전환 시 Mark Word 변화 관찰 |
| [02. synchronized 내부 — Thin→Fat Lock 확장](./lock-internals/02-synchronized-lock-escalation.md) | 최초 진입 시 Thin Lock(CAS로 Lock Record 주소 교체), 경쟁 심화 시 Fat Lock(OS Mutex, 컨텍스트 스위칭 발생) 전 과정, `jol-core`로 락 확장 단계별 Mark Word 상태 관찰 |
| [03. ReentrantLock 내부 — AQS 대기 큐 구조](./lock-internals/03-reentrantlock-aqs.md) | `AbstractQueuedSynchronizer`의 CLH 변형 대기 큐 구조, `state` 필드를 CAS로 업데이트하는 락 획득/해제 과정, Fair Lock이 큐 순서를 보장하는 방식과 Non-Fair Lock보다 느린 이유, `LockSupport.park()` / `unpark()`로 스레드를 멈추고 깨우는 원리 |
| [04. ReadWriteLock — 읽기/쓰기 분리와 Write Starvation](./lock-internals/04-read-write-lock.md) | `ReentrantReadWriteLock`이 단일 `state` 필드의 상위 16비트(읽기 홀드 수) / 하위 16비트(쓰기 홀드 수)로 읽기/쓰기를 분리 관리하는 방식, 읽기가 많을 때 쓰기 스레드가 영원히 대기하는 Write Starvation 발생 조건 |
| [05. StampedLock — 낙관적 읽기와 validateStamp](./lock-internals/05-stamped-lock.md) | `StampedLock`의 낙관적 읽기(락 없이 읽기 시도 → `validateStamp()`로 쓰기 발생 여부 확인)가 읽기 성능을 높이는 원리, 낙관적 읽기 실패 시 비관적 읽기로 전환하는 패턴, `StampedLock`이 재진입 불가인 이유와 데드락 주의사항 |
| [06. 조건 변수 — wait/notify vs Condition, Spurious Wakeup](./lock-internals/06-condition-variables.md) | `Object.wait()` / `notify()` / `notifyAll()`의 모니터 락 연결 구조, `Condition.await()` / `signal()`이 더 세밀한 제어를 가능하게 하는 방식, Spurious Wakeup(가짜 깨어남)이 발생하는 이유와 `while` 루프로 방어하는 패턴, `ArrayBlockingQueue` 내부의 두 Condition 활용 |
| [07. 락 성능 비교 — JMH 벤치마크와 JIT 최적화](./lock-internals/07-lock-performance-benchmark.md) | `synchronized` vs `ReentrantLock` vs `StampedLock` vs `AtomicLong` JMH 처리량 벤치마크, 경쟁 수준(스레드 수)별 성능 역전 조건, JIT의 Lock Elision(탈출 분석으로 락 제거) / Lock Coarsening(락 범위 통합) 최적화가 적용되는 조건 |

</details>

<br/>

### 🔹 Chapter 4: Lock-Free 알고리즘과 원자적 연산

> **핵심 질문:** CAS는 CPU 명령어 수준에서 어떻게 원자성을 보장하는가? ABA 문제는 왜 발생하고, LongAdder는 어떻게 AtomicLong보다 빠른가?

<details>
<summary><b>LOCK CMPXCHG부터 VarHandle 메모리 오더링까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. CAS — LOCK CMPXCHG와 ABA 문제](./lock-free-algorithms/01-cas-cpu-level.md) | `Compare-And-Swap`의 x86 구현(`LOCK CMPXCHG` 명령어)이 메모리 버스를 잠금 없이 원자성을 보장하는 원리, `-XX:+PrintAssembly`로 어셈블리 출력 확인, ABA 문제(값이 A→B→A로 바뀌었지만 CAS는 변경을 감지하지 못함) 발생 시나리오 |
| [02. AtomicInteger/AtomicLong 내부 — Unsafe와 VarHandle](./lock-free-algorithms/02-atomic-integer-varhandle.md) | Java 8 이하의 `sun.misc.Unsafe.compareAndSwapInt()` 사용 방식, Java 9+에서 `VarHandle`로 전환한 이유(모듈 시스템 호환), `VarHandle.compareAndSet()`이 `Unsafe`와 동일한 성능을 내는 이유, `AtomicInteger` 소스코드 단계별 분석 |
| [03. AtomicReference와 AtomicStampedReference — ABA 해결](./lock-free-algorithms/03-atomic-reference-stamped.md) | `AtomicReference`로 참조 타입을 원자적으로 교체하는 방식, ABA 문제가 참조 타입에서 실제로 버그를 일으키는 시나리오, `AtomicStampedReference`의 (참조, 스탬프) 쌍으로 ABA를 방지하는 메커니즘, `AtomicMarkableReference`와의 차이 |
| [04. LongAdder vs AtomicLong — 고경쟁 환경의 Cell 분산](./lock-free-algorithms/04-longadder-striped64.md) | `Striped64` 기반 `Cell[]` 배열로 CAS 충돌을 스레드별로 분산하는 스트라이프 기법, 고경쟁 환경에서 `AtomicLong` CAS 실패율이 높아지는 원리, `sum()`이 모든 Cell을 합산할 때 일관성이 보장되지 않는 이유, JMH로 스레드 수별 처리량 비교 |
| [05. Lock-Free 자료구조 — ConcurrentLinkedQueue 내부](./lock-free-algorithms/05-lock-free-data-structures.md) | Michael-Scott Queue 알고리즘으로 구현된 `ConcurrentLinkedQueue`의 head / tail CAS 기반 삽입·삭제 과정, 동시에 여러 스레드가 enqueue할 때 tail 포인터를 따라가는 "helping" 메커니즘, Non-Blocking 알고리즘 설계 시 ABA · 메모리 재사용 주의사항 |
| [06. 메모리 오더링 — VarHandle의 Acquire/Release/Opaque](./lock-free-algorithms/06-varhandle-memory-ordering.md) | `VarHandle.getAcquire()` / `setRelease()` / `getOpaque()` / `setOpaque()` 각각이 보장하는 메모리 순서 수준(full fence → acquire/release → opaque → plain), 순서 보장 약화에 따른 성능 향상 폭과 안전하게 사용할 수 있는 조건, JMH로 각 모드의 처리량 비교 |

</details>

<br/>

### 🔹 Chapter 5: 동시성 자료구조 내부 구현

> **핵심 질문:** ConcurrentHashMap은 Java 7과 Java 8에서 어떻게 다르게 구현됐는가? BlockingQueue는 어떻게 생산자-소비자를 블로킹하고, CopyOnWriteArrayList는 락 없이 읽기가 안전한 이유는 무엇인가?

<details>
<summary><b>ConcurrentHashMap 진화부터 Phaser 다단계 동기화까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. ConcurrentHashMap 진화 — 세그먼트 락에서 CAS로](./concurrent-data-structures/01-concurrent-hashmap.md) | Java 7의 16개 세그먼트(Segment) ReentrantLock 구조, Java 8에서 버킷 레벨 `synchronized` + CAS로 전환하고 세그먼트를 제거한 이유, `computeIfAbsent` 내부 동작과 락 범위, 트리 변환(TREEIFY_THRESHOLD=8) 조건과 Red-Black Tree 노드 구조 |
| [02. CopyOnWriteArrayList — 쓰기 복사와 읽기 안전성](./concurrent-data-structures/02-copy-on-write-arraylist.md) | 쓰기마다 배열 전체를 복사하고 참조를 원자적으로 교체하는 구조, 읽기 스레드가 락 없이 안전한 이유(스냅샷 참조 유지), 쓰기 비용(O(N) 복사)이 감당 가능한 사용 사례와 부적합한 사용 사례, `iterator()`가 생성 시점의 스냅샷을 유지하는 원리 |
| [03. BlockingQueue — ArrayBlockingQueue vs LinkedBlockingQueue](./concurrent-data-structures/03-blocking-queue-internals.md) | `ArrayBlockingQueue`의 단일 ReentrantLock + 두 Condition(notFull/notEmpty)으로 put/take를 블로킹하는 방식, `LinkedBlockingQueue`의 head/tail 분리 락으로 생산자·소비자 경합을 줄이는 구조, `SynchronousQueue`의 직접 전달 메커니즘, 실전 생산자-소비자 패턴 |
| [04. ConcurrentSkipListMap — Lock-Free 정렬 맵](./concurrent-data-structures/04-concurrent-skip-list-map.md) | 스킵 리스트가 확률적 레이어로 O(log N) 검색·삽입을 보장하는 구조, `ConcurrentSkipListMap`이 CAS로 노드를 삽입하고 "마커 노드"로 삭제를 표시하는 Non-Blocking 알고리즘, `TreeMap`(Red-Black Tree) 대비 멀티스레드 환경 성능 비교 |
| [05. Exchanger와 Phaser — 데이터 교환과 다단계 동기화](./concurrent-data-structures/05-exchanger-phaser.md) | `Exchanger`가 두 스레드 간 데이터를 교환하는 원리와 파이프라인 패턴 활용, `Phaser`로 CyclicBarrier를 대체하고 동적으로 참가자를 추가/제거하는 방법, `Phaser`의 arriveAndAwaitAdvance() / arriveAndDeregister() 차이, 다단계 병렬 계산에서의 활용 |

</details>

<br/>

### 🔹 Chapter 6: Virtual Thread (Project Loom) — 패러다임의 전환

> **핵심 질문:** Virtual Thread는 OS 스레드와 무엇이 다른가? Continuation이 어떻게 블로킹 없이 컨텍스트 스위칭하고, Pinning은 왜 발생하며 어떻게 해결하는가?

<details>
<summary><b>OS 스레드 한계부터 Spring Boot Virtual Thread 통합까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 왜 Virtual Thread인가 — OS 스레드의 한계](./virtual-threads/01-why-virtual-thread.md) | OS 스레드 1:1 모델의 비용(1MB 스택, 컨텍스트 스위칭), I/O 바운드 작업에서 스레드 풀 스레드가 블로킹 대기 중 낭비되는 구조, "Thread per Request" 모델의 처리량 상한선, Virtual Thread가 이 문제를 어떻게 해결하는가 |
| [02. Continuation과 스케줄링 — 힙 저장과 ForkJoinPool](./virtual-threads/02-continuation-scheduling.md) | `Continuation` 객체가 JVM 힙에 Virtual Thread의 스택 프레임을 저장하는 방식(수 KB), ForkJoinPool 캐리어 스레드(CPU 코어 수)가 Virtual Thread를 스케줄링하는 원리, Virtual Thread가 Mount(캐리어에 할당) / Unmount(캐리어에서 해제)되는 과정 |
| [03. 블로킹 시 동작 — Unmount와 NIO 재작성](./virtual-threads/03-blocking-unmount.md) | Virtual Thread가 `Socket.read()` 같은 블로킹 I/O를 만났을 때 JDK 내부에서 NIO non-blocking으로 재작성되는 과정, 데이터 없으면 Continuation 저장 후 Unmount, 데이터 도착 시 Mount 재개하는 전체 흐름, `Thread.sleep()`이 Virtual Thread에서 캐리어를 블로킹하지 않는 이유 |
| [04. Pinning 문제 — synchronized와 캐리어 스레드 고착](./virtual-threads/04-pinning-problem.md) | `synchronized` 블록 안에서 블로킹 I/O 호출 시 캐리어 스레드도 함께 블로킹되는 "Pinning" 발생 원인(JVM 모니터 구현 제약), `-Djdk.tracePinnedThreads=full` 플래그와 JFR `jdk.VirtualThreadPinned` 이벤트로 진단하는 방법, `ReentrantLock`으로 교체해 Pinning을 해결하는 패턴 |
| [05. ThreadLocal과 ScopedValue — 메모리 누수와 Java 21 해결책](./virtual-threads/05-threadlocal-scopedvalue.md) | Virtual Thread를 100만 개 생성할 때 각각에 ThreadLocal 값이 복사되어 메모리 누수가 발생하는 원리, 풀링(pooling)이 Virtual Thread에서 안티패턴인 이유, Java 21 `ScopedValue`가 불변 상태를 스코프 내에서 공유하는 설계 원리, ThreadLocal vs ScopedValue 비교 |
| [06. Spring Boot + Virtual Thread — 실전 통합](./virtual-threads/06-spring-boot-virtual-thread.md) | `spring.threads.virtual.enabled=true` 설정으로 Tomcat 요청 처리 스레드를 Virtual Thread로 전환하는 방법, `@Async`와 Virtual Thread 조합, JPA / DataSource 커넥션 풀(HikariCP)이 Virtual Thread와 호환되는 방식, Pinning 발생 지점 진단 및 수정 |

</details>

<br/>

### 🔹 Chapter 7: 실전 패턴과 장애 분석

> **핵심 질문:** Thread Dump에서 데드락을 어떻게 찾는가? Lock Contention은 어떻게 측정하고, CompletableFuture / WebFlux / Virtual Thread는 실제로 무엇이 다른가?

<details>
<summary><b>데드락 분석부터 Spring 동시성 이슈 패턴까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 데드락 발생과 분석 — jstack과 Thread Dump](./real-world-patterns/01-deadlock-analysis.md) | 데드락 4가지 조건(상호 배제, 점유 대기, 비선점, 순환 대기), 의도적으로 데드락을 재현하는 코드, `jstack <pid>` / `jcmd <pid> Thread.print`로 Thread Dump 수집, "Found one Java-level deadlock" 섹션에서 락 소유자와 대기자를 추적하는 방법 |
| [02. 라이브락과 기아 — 공정한 락의 트레이드오프](./real-world-patterns/02-livelock-starvation.md) | 스핀 락에서 두 스레드가 서로 양보하며 진행하지 못하는 라이브락 발생 조건, Non-Fair Lock에서 특정 스레드가 영원히 락을 획득하지 못하는 기아(Starvation) 시나리오, Fair Lock으로 기아를 해결하지만 처리량이 저하되는 트레이드오프, `ThreadMXBean`으로 스레드 CPU 사용량을 모니터링하는 방법 |
| [03. 성능 병목 패턴 — Lock Contention과 JFR 진단](./real-world-patterns/03-lock-contention-profiling.md) | Lock Contention(락 경합)이 처리량을 저하시키는 원리, JFR(Java Flight Recorder)로 `jdk.JavaMonitorEnter` 이벤트를 수집해 경합 발생 지점 찾기, `async-profiler`의 `-e lock` 모드로 락 대기 시간 Flame Graph 생성, 과도한 `synchronized` 범위를 줄이는 리팩토링 패턴 |
| [04. 비동기 프로그래밍 비교 — CompletableFuture vs WebFlux vs Virtual Thread](./real-world-patterns/04-async-comparison.md) | `CompletableFuture`(스레드 풀 작업 체인) vs Spring WebFlux(Reactor 이벤트 루프, Non-Blocking) vs Virtual Thread(블로킹 코드 그대로, OS 스레드 절약)의 실제 동작 차이, I/O 바운드 100만 요청 시나리오에서 처리량과 메모리 사용량 비교, 각각이 적합한 사용 사례 |
| [05. Spring 동시성 이슈 — @Transactional·@Async·싱글톤 Bean](./real-world-patterns/05-spring-concurrency-issues.md) | `@Transactional`과 멀티스레드 조합 시 트랜잭션이 스레드에 바인딩되는 원리와 주의사항, `@Async` 스레드 풀 설정이 누락될 때 `SimpleAsyncTaskExecutor`가 스레드를 무제한 생성하는 문제, 싱글톤 Bean의 인스턴스 변수 공유로 발생하는 Race Condition 패턴과 진단 |

</details>

---

## 🔬 실험 환경

```yaml
# docker-compose.yml
services:
  java-lab:
    image: eclipse-temurin:21-jdk
    volumes:
      - ./src:/workspace/src
      - ./benchmarks:/workspace/benchmarks
    working_dir: /workspace
    command: /bin/bash
    stdin_open: true
    tty: true
    environment:
      - JAVA_OPTS=-XX:+UseZGC -Xmx2g

  jmh-runner:
    image: maven:3.9-eclipse-temurin-21
    volumes:
      - ./benchmarks:/workspace
    working_dir: /workspace
    command: mvn clean package && java -jar target/benchmarks.jar
```

```bash
# ── JVM 내부 동작 확인 ──────────────────────────────────────────────

# JIT 컴파일 로그 + 어셈블리 출력
# 주의: -XX:+PrintAssembly는 hsdis 라이브러리가 필요합니다
#       설치: https://github.com/openjdk/jdk/tree/master/src/utils/hsdis
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogCompilation \
     -XX:+PrintAssembly \
     MyApp

# ── Virtual Thread 진단 ─────────────────────────────────────────────

# Pinning 발생 추적 (synchronized 안에서 블로킹 시 출력)
java -Djdk.tracePinnedThreads=full MyVirtualThreadApp

# ── Thread Dump (데드락 분석) ───────────────────────────────────────

jstack <pid>
jcmd <pid> Thread.print

# ── Java Flight Recorder (성능 프로파일링) ──────────────────────────

# 60초 기록
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Virtual Thread Pinning 이벤트만 추출
jfr print --events jdk.VirtualThreadPinned recording.jfr

# 락 경합 이벤트 추출
jfr print --events jdk.JavaMonitorEnter recording.jfr

# ── JMH 벤치마크 실행 ──────────────────────────────────────────────

# 3 fork / 5 warmup iterations / 10 measurement iterations / 4 threads
java -jar benchmarks.jar -f 3 -wi 5 -i 10 -t 4 ".*LockBenchmark.*"
```

```java
// 기본 JMH 벤치마크 구조 (모든 챕터에서 활용)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(3)
@Warmup(iterations = 5)
@Measurement(iterations = 10)
public class ConcurrencyBenchmark {

    @Benchmark
    public void synchronizedMethod() { /* ... */ }

    @Benchmark
    public void reentrantLock() { /* ... */ }

    @Benchmark
    public void stampedLockOptimistic() { /* ... */ }
}
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 실무에서 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — 원리를 모를 때의 접근과 그 결과 |
| ✨ **올바른 접근** | After — 원리를 알고 난 후의 설계/구현 |
| 🔬 **내부 동작 원리** | JVM / JIT / CPU 수준 분석, ASCII 구조도 |
| 💻 **실전 실험** | JVM 플래그, JMH 벤치마크, Thread Dump, JFR |
| 📊 **성능/비용 비교** | 벤치마크 수치, OS 스레드 vs Virtual Thread 등 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "synchronized를 쓰지만 왜 느린지 모른다" — 락 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch3-01  Object Header Mark Word → 락이 어디에 저장되는지 이해
       Ch3-02  Thin→Fat Lock 확장 → synchronized 내부 동작 전 과정
Day 2  Ch2-03  happens-before 규칙 → 가시성 보장 범위 이해
       Ch2-04  volatile 완전 분해 → volatile만으로 부족한 케이스
Day 3  Ch3-07  락 성능 벤치마크 → JMH로 실제 수치 확인
       Ch7-03  Lock Contention JFR 진단 → 실무 병목 찾기
```

</details>

<details>
<summary><b>🟡 "Virtual Thread를 도입하려는데 내부가 궁금하다" — Virtual Thread 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  OS 스레드 모델 → Java 스레드 1:1 매핑 비용 이해
       Ch1-02  컨텍스트 스위칭 비용 → Virtual Thread가 해결하는 문제
Day 2  Ch6-01  왜 Virtual Thread인가 → 설계 동기
       Ch6-02  Continuation과 스케줄링 → 힙에 저장되는 스택 프레임
Day 3  Ch6-03  블로킹 시 Unmount 동작 → NIO 재작성 전 과정
       Ch6-04  Pinning 문제 → synchronized 안 블로킹의 위험
Day 4  Ch6-05  ThreadLocal과 ScopedValue → 메모리 누수 방지
       Ch6-06  Spring Boot + Virtual Thread → 실전 통합
Day 5  Ch4-01  CAS 원리 → Lock-Free가 Virtual Thread와 궁합이 좋은 이유
Day 6  Ch7-04  비동기 비교 → CompletableFuture / WebFlux / Virtual Thread 실제 차이
Day 7  Ch7-05  Spring 동시성 이슈 → @Async, @Transactional 주의사항
```

</details>

<details>
<summary><b>🔴 "JVM 동시성 내부를 CPU 명령어 수준까지 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — OS 스레드와 Java 스레드 기반
        → strace로 clone() 시스템콜 관찰, ThreadPoolExecutor 큐 동작 실험

2주차  Chapter 2 전체 — Java Memory Model
        → volatile 어셈블리 출력(-XX:+PrintAssembly), DCL 버그 재현

3주차  Chapter 3 전체 — 락의 내부 구현
        → jol-core로 Mark Word 상태 출력, JMH 락 벤치마크 직접 실행

4주차  Chapter 4 전체 — Lock-Free 알고리즘
        → AtomicInteger 어셈블리 확인, LongAdder vs AtomicLong JMH 비교

5주차  Chapter 5 전체 — 동시성 자료구조
        → ConcurrentHashMap 버킷 레벨 락 범위 실험, BlockingQueue 생산자-소비자

6주차  Chapter 6 전체 — Virtual Thread
        → Pinning 재현(-Djdk.tracePinnedThreads=full), JFR VirtualThreadPinned 분석

7주차  Chapter 7 전체 — 실전 패턴과 장애 분석
        → 데드락 의도적 재현 → Thread Dump 분석, async-profiler Flame Graph
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [jvm-deep-dive](https://github.com/dev-book-lab/jvm-deep-dive) | JVM 구조, 클래스로딩, GC, JIT 컴파일 | Ch2(JMM이 JVM 위에서 동작하는 방식), Ch3-02(JIT Lock Elision 최적화), Ch4-02(VarHandle이 Unsafe를 대체한 이유) |
| [linux-for-backend-deep-dive](https://github.com/dev-book-lab/linux-for-backend-deep-dive) | OS 프로세스·스레드, 컨텍스트 스위칭, epoll | Ch1(Java 스레드와 OS 스레드 1:1 매핑), Ch6-03(Virtual Thread가 epoll 위에서 동작하는 방식) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Spring MVC 요청 처리, DispatcherServlet | Ch6-06(Tomcat 스레드 모델을 Virtual Thread로 전환하는 설정) |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | @Transactional, 트랜잭션 전파, JDBC | Ch7-05(@Transactional과 멀티스레드 조합 주의사항, 커넥션 풀과 Virtual Thread) |
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | AOP Proxy, 싱글톤 Bean 생명주기 | Ch7-05(싱글톤 Bean 인스턴스 변수 Race Condition, @Async 스레드 풀 설정) |
| [redis-deep-dive](https://github.com/dev-book-lab/redis-deep-dive) | Redis 단일 스레드 이벤트 루프, I/O | Ch6-01(단일 스레드 Redis가 빠른 이유 vs 멀티스레드 Java 서버의 차이) |

> 💡 이 레포는 **JVM 동시성 내부**에 집중합니다. `jvm-deep-dive`로 JVM 구조를 먼저 이해하면 Chapter 2(JMM)와 Chapter 3(락 확장)의 깊이가 배가됩니다. `linux-for-backend-deep-dive`로 OS 스레드와 epoll을 이해하면 Chapter 1과 Chapter 6(Virtual Thread)의 연결이 명확해집니다.

---

## 📚 Reference

- [Java Concurrency in Practice — Brian Goetz et al.](https://jcip.net/) — 동시성 바이블
- [The Art of Multiprocessor Programming — Herlihy & Shavit](https://www.elsevier.com/books/the-art-of-multiprocessor-programming/herlihy/978-0-12-415950-1) — Lock-Free 알고리즘 원전
- [Inside the Java Virtual Machine — Bill Venners](https://www.artima.com/insidejvm/ed2/) — JVM 내부 구조
- [JEP 374 (Biased Locking Deprecation)](https://openjdk.org/jeps/374)
- [JEP 425 (Virtual Threads Preview)](https://openjdk.org/jeps/425)
- [JEP 444 (Virtual Threads in Java 21)](https://openjdk.org/jeps/444)
- [Aleksey Shipilëv's Blog](https://shipilev.net/) — JMM, JIT, 성능 분석의 최고 자료
- [Nitsan Wakart's Blog](https://psy-lob-saw.blogspot.com/) — Lock-Free, 성능 측정

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"synchronized를 쓰는 것과, JVM이 Object Header의 Mark Word를 어떻게 바꾸며 락을 확장하는지 아는 것은 다르다"*

</div>
