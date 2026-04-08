# 스레드 상태 머신 — 6가지 상태 전환 조건

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 스레드의 6가지 상태는 각각 어떤 조건에서 전환되는가?
- `BLOCKED`와 `WAITING`은 어떻게 다른가?
- `Thread.sleep()`과 `Object.wait()`과 `LockSupport.park()`는 각각 어떤 상태를 만드는가?
- `jstack`으로 Thread Dump를 찍었을 때 각 상태를 어떻게 해석하는가?
- RUNNABLE이지만 실제로 CPU를 쓰지 않는 경우는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

장애 상황에서 `jstack`으로 Thread Dump를 찍었을 때 스레드 상태를 해석하지 못하면 원인 분석이 불가능하다. "왜 모든 스레드가 BLOCKED인가", "WAITING 중인 스레드가 왜 이렇게 많은가"를 이해하는 것이 장애 대응의 시작이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: RUNNABLE = CPU 사용 중이라고 착각
  Thread Dump에서 모든 스레드가 RUNNABLE로 보임
  "스레드들이 다 열심히 일하고 있다"
  → 실제로는 I/O 대기 중인 스레드도 RUNNABLE로 표시됨
  → CPU 사용률은 낮지만 스레드는 RUNNABLE → 오해 유발

실수 2: BLOCKED와 WAITING을 혼동
  "스레드가 WAITING 상태다 → 데드락 의심"
  → WAITING은 정상적인 상태 (wait(), park() 호출)
  → 데드락은 BLOCKED 스레드들이 순환 대기하는 것
  → Thread Dump에서 "Found one Java-level deadlock" 메시지로 확인

실수 3: sleep과 wait의 차이를 모름
  synchronized 블록 안에서 Thread.sleep() 사용
  → 락을 쥔 채로 슬리핑 → 다른 스레드가 락 못 획득
  Object.wait()을 써야 할 자리에 sleep 사용
  → 락을 반납하지 않아 교착 발생
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```
상태별 올바른 해석:

BLOCKED (스레드가 많다면):
  → synchronized 경합 발생 중
  → 어떤 락을 기다리는지 Thread Dump에서 확인
  → "waiting to lock <0x...>" 문자열 추적

WAITING (대다수라면):
  → 정상적인 wait/park 대기 (생산자-소비자 패턴 등)
  → 특정 조건 충족 시 깨어날 준비

TIMED_WAITING (대다수라면):
  → sleep, timed wait 중
  → 의도된 대기인지, 과도한 sleep인지 확인

RUNNABLE (CPU 낮은데 전부 RUNNABLE):
  → I/O 대기 중 (소켓 read, 파일 read 등)
  → OS 레벨에서는 블로킹이지만 JVM에서 RUNNABLE로 표시
  → 이 경우 스레드 수 늘려도 처리량 미증가 (I/O가 병목)

생산자-소비자 올바른 패턴:
  synchronized (queue) {
      while (queue.isEmpty()) {  // while (if 아님! Spurious Wakeup 방지)
          queue.wait();          // 락 반납 + WAITING 진입
      }
      process(queue.poll());
  }
```

---

## 🔬 내부 동작 원리

### 1. 6가지 상태 전체 지도

```
Java Thread 상태 (Thread.State enum):

                    ┌─────────────────────┐
      new Thread()  │                     │
         ──────────▶│        NEW          │
                    │                     │
                    └──────────┬──────────┘
                               │ start()
                               ▼
                    ┌─────────────────────┐
                    │                     │◀──────────────────────────────┐
                    │      RUNNABLE       │   notify() / notifyAll()      │
                    │  (실행 중 또는         │   unpark()                    │
                    │   실행 대기 중)        │   interrupt()                 │
                    └──┬────────┬─────────┘   timeout 만료                 │
                       │        │              I/O 완료                    │
          synchronized │        │ Object.wait()                           │
          락 획득 실패    │        │ LockSupport.park()                      │
                       │        │ Thread.join()                           │
                       ▼        ▼                                         │
             ┌──────────┐  ┌──────────────────────┐                       │
             │          │  │                      │                       │
             │ BLOCKED  │  │      WAITING         │────────────────────▶  │
             │          │  │                      │
             └──────────┘  └──────────────────────┘
             락 획득 시         ↕ wait(ms), sleep(ms),
             RUNNABLE로         join(timeout) 시
                          ┌──────────────────────┐
                          │                      │
                          │   TIMED_WAITING      │
                          │                      │
                          └──────────────────────┘

                    ┌─────────────────────┐
                    │                     │
                    │     TERMINATED      │◀── run() 메서드 종료
                    │                     │
                    └─────────────────────┘
```

### 2. 각 상태의 상세 전환 조건

```
NEW:
  new Thread() 직후, start() 호출 전
  JVM 내부: Thread 객체는 생성됐지만 OS 스레드는 아직 없음

RUNNABLE:
  start() 호출 후
  run() 메서드 실행 중 (CPU 점유)
  I/O 대기 중 (소켓 read 등) ← 여기가 주의 포인트!
    → OS 레벨: 블로킹 상태지만 JVM에서는 RUNNABLE로 표시
    → Thread Dump에서 "waiting on condition" 없이 그냥 RUNNABLE로 보임
  실행 준비 완료 (스케줄러 대기)

  RUNNABLE 진입:
    start() 호출
    BLOCKED 상태에서 락 획득
    WAITING/TIMED_WAITING에서 깨어남

BLOCKED:
  synchronized 블록/메서드 진입을 위해 모니터 락 대기 중
  락을 다른 스레드가 보유 중일 때만 발생
  Thread.State.BLOCKED는 오직 synchronized 대기만 표시
  (ReentrantLock 대기는 WAITING으로 표시)

  Thread Dump에서:
    "java.lang.Thread.State: BLOCKED (on object monitor)"
    "waiting to lock <0x00000007815c7498> (a java.lang.Object)"
    "locked by: Thread-1 ..."

WAITING (무기한 대기):
  깨어나는 조건이 외부 신호에 의존

  진입 조건:
    Object.wait()        → notify() / notifyAll() 까지 대기
    Thread.join()        → 대상 스레드 종료까지 대기
    LockSupport.park()   → unpark() / interrupt() 까지 대기

  Thread Dump에서:
    "java.lang.Thread.State: WAITING (on object monitor)"
    "waiting on <0x00000007815c7498>"

  ReentrantLock.lock() 대기:
    "java.lang.Thread.State: WAITING (parking)"
    "parking to wait for <0x...> (a java.util.concurrent.locks.ReentrantLock)"
    ← BLOCKED가 아닌 WAITING! (LockSupport.park() 기반)

TIMED_WAITING (시간 제한 대기):
  진입 조건:
    Thread.sleep(ms)        → 지정 시간 후 RUNNABLE
    Object.wait(ms)         → 시간 초과 또는 notify()
    Thread.join(ms)         → 시간 초과 또는 대상 종료
    LockSupport.parkNanos() → 나노초 단위 대기
    LockSupport.parkUntil() → 절대 시간까지 대기

  Thread Dump에서:
    "java.lang.Thread.State: TIMED_WAITING (sleeping)"
    "java.lang.Thread.State: TIMED_WAITING (on object monitor)"

TERMINATED:
  run() 메서드 정상 종료
  run() 메서드에서 예외 발생 (UncaughtExceptionHandler 호출 후)
  interrupt()로 강제 종료 (InterruptedException 처리 후 종료)
  TERMINATED 스레드는 재사용 불가 (start()를 다시 호출하면 IllegalThreadStateException)
```

### 3. sleep vs wait vs park 비교

```
Thread.sleep(ms):
  현재 스레드를 TIMED_WAITING으로 전환
  락을 보유 중이면 → 락 유지한 채로 대기 (!)
  → synchronized 블록 안에서 sleep 시 다른 스레드가 진입 불가
  InterruptedException 발생 가능
  인터럽트 시 즉시 깨어남

  synchronized (obj) {
      Thread.sleep(1000);  // 1초 동안 obj 락을 유지
      // 다른 스레드는 이 1초 동안 obj를 사용 못 함
  }

Object.wait():
  반드시 synchronized 블록 안에서 호출 (아니면 IllegalMonitorStateException)
  호출 시 락 반납 + WAITING 상태
  notify()/notifyAll() 또는 인터럽트로 깨어남
  깨어난 후 다시 락 획득 시도 (BLOCKED 상태 거침 → 락 획득 후 RUNNABLE)

  synchronized (obj) {
      while (condition) {
          obj.wait();  // obj 락 반납 + 대기 → 다른 스레드가 obj 사용 가능
      }
      // 깨어난 후 여기서 실행 재개
  }

LockSupport.park():
  WAITING 상태 (또는 parkNanos로 TIMED_WAITING)
  락과 무관 — synchronized 밖에서도 호출 가능
  LockSupport.unpark(thread)로 특정 스레드를 깨울 수 있음
  인터럽트로도 깨어남 (InterruptedException 아닌 interrupt 상태만 set)
  AQS(AbstractQueuedSynchronizer) 내부 구현에 사용
  ReentrantLock.lock() 대기 시 내부적으로 park() 호출

  LockSupport.park();  // 누군가 unpark(this) 또는 interrupt() 호출까지 대기
  // 깨어난 후 인터럽트 상태 확인 필요
  if (Thread.interrupted()) { /* 인터럽트로 깨어남 */ }
```

### 4. Thread Dump 읽는 방법

```
jstack <pid> 출력 예시 (각 상태 포함):

"http-nio-8080-exec-1" #25 daemon prio=5 os_prio=0 cpu=123.45ms
   java.lang.Thread.State: RUNNABLE         ← 상태
    at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)   ← I/O 대기 중
    at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
    at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
    ...
    at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(...)
    
→ RUNNABLE이지만 실제로는 epoll I/O 대기 중 (CPU 사용 없음)

"worker-1" #30 prio=5 os_prio=0 cpu=890.12ms
   java.lang.Thread.State: BLOCKED (on object monitor)
    at com.example.Service.process(Service.java:45)
    - waiting to lock <0x00000007815c7498> (a com.example.Resource)
    - locked by "worker-2" ...
    
→ worker-2가 Resource 락을 보유 중, worker-1이 대기

"async-1" #35 prio=5 os_prio=0 cpu=0.23ms
   java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    - parking to wait for <0x...> (a java.util.concurrent.locks.ReentrantLock)
    at java.util.concurrent.locks.LockSupport.park(...)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(...)
    
→ ReentrantLock 획득 대기 (WAITING, BLOCKED 아님)

"pool-1-thread-1" #40 prio=5 os_prio=0 cpu=2.34ms
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at com.example.Scheduler.run(Scheduler.java:30)
    
→ Thread.sleep() 중
```

---

## 💻 실전 실험

### 실험 1: 모든 상태를 직접 유발하고 jstack으로 관찰

```java
import java.util.concurrent.locks.LockSupport;

public class ThreadStateDemo {
    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        // WAITING 스레드
        Thread waiting = new Thread(() -> {
            synchronized (lock) {
                try {
                    lock.wait();  // WAITING
                } catch (InterruptedException e) {}
            }
        }, "demo-WAITING");

        // TIMED_WAITING 스레드
        Thread timedWaiting = new Thread(() -> {
            try {
                Thread.sleep(60000);  // TIMED_WAITING
            } catch (InterruptedException e) {}
        }, "demo-TIMED_WAITING");

        // BLOCKED 스레드 (lock을 먼저 잡는 스레드가 필요)
        Thread lockHolder = new Thread(() -> {
            synchronized (lock) {
                try {
                    Thread.sleep(60000);  // 락을 쥐고 슬리핑
                } catch (InterruptedException e) {}
            }
        }, "demo-LOCK_HOLDER");

        Thread blocked = new Thread(() -> {
            synchronized (lock) {}  // BLOCKED (lockHolder가 락 보유 중)
        }, "demo-BLOCKED");

        // WAITING (park)
        Thread parked = new Thread(() -> {
            LockSupport.park();  // WAITING (parking)
        }, "demo-PARKED");

        waiting.start();
        timedWaiting.start();
        lockHolder.start();
        Thread.sleep(100);
        blocked.start();
        parked.start();

        Thread.sleep(500);

        // 상태 출력
        System.out.println("demo-WAITING: " + waiting.getState());
        System.out.println("demo-TIMED_WAITING: " + timedWaiting.getState());
        System.out.println("demo-LOCK_HOLDER: " + lockHolder.getState());
        System.out.println("demo-BLOCKED: " + blocked.getState());
        System.out.println("demo-PARKED: " + parked.getState());

        long pid = ProcessHandle.current().pid();
        System.out.println("\n# jstack으로 확인:");
        System.out.println("jstack " + pid + " | grep -A 10 'demo-'");
    }
}
```

### 실험 2: sleep vs wait 차이 확인 (락 유지 여부)

```java
public class SleepVsWait {
    private static final Object resource = new Object();

    public static void main(String[] args) throws InterruptedException {
        // 실험 1: synchronized 안에서 sleep (락 유지)
        Thread sleeper = new Thread(() -> {
            synchronized (resource) {
                System.out.println("Sleeper: 락 획득, sleep 시작");
                try {
                    Thread.sleep(3000);  // 락 유지한 채로 대기
                } catch (InterruptedException e) {}
                System.out.println("Sleeper: 완료");
            }
        }, "Sleeper");

        Thread waiter = new Thread(() -> {
            System.out.println("Waiter: 락 획득 시도 중...");
            long start = System.currentTimeMillis();
            synchronized (resource) {  // Sleeper가 락 해제할 때까지 대기
                System.out.printf("Waiter: 락 획득 (%.1fs 대기)%n",
                    (System.currentTimeMillis() - start) / 1000.0);
            }
        }, "Waiter");

        sleeper.start();
        Thread.sleep(100);
        waiter.start();
        waiter.join();

        System.out.println("\n--- wait 실험 ---");

        // 실험 2: synchronized 안에서 wait (락 반납)
        Thread producer = new Thread(() -> {
            synchronized (resource) {
                System.out.println("Producer: 락 반납하고 wait 시작");
                try {
                    resource.wait(3000);  // 락 반납 + 대기
                } catch (InterruptedException e) {}
                System.out.println("Producer: 깨어남");
            }
        }, "Producer");

        Thread consumer = new Thread(() -> {
            try { Thread.sleep(500); } catch (InterruptedException e) {}
            System.out.println("Consumer: 락 획득 시도...");
            long start = System.currentTimeMillis();
            synchronized (resource) {  // Producer가 wait 중이면 즉시 획득 가능
                System.out.printf("Consumer: 락 획득 (%.1fs 대기)%n",
                    (System.currentTimeMillis() - start) / 1000.0);
                resource.notify();  // Producer 깨우기
            }
        }, "Consumer");

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

---

## 📊 성능/비용 비교

```
대기 메커니즘별 특성 비교:

메커니즘               | 상태           | 락 반납  | 깨우기 방법           | 사용처
─────────────────────┼───────────────┼────────┼────────────────────┼──────────────────
Thread.sleep(ms)     | TIMED_WAITING | ❌     | timeout / interrupt| 단순 지연
Object.wait()        | WAITING       | ✅     | notify/interrupt   | 모니터 기반 협력
Object.wait(ms)      | TIMED_WAITING | ✅     | timeout/notify/int | 타임아웃 wait
LockSupport.park()   | WAITING       | N/A    | unpark/interrupt   | AQS, 커스텀 동기화
LockSupport.parkNanos| TIMED_WAITING | N/A    | timeout/unpark/int | 정밀 타임아웃

상태별 CPU 영향:

상태           | CPU 소비  | OS 스케줄러 관리
──────────────┼──────────┼───────────────────────
RUNNABLE      | 가능      | 실행 큐에 포함
BLOCKED       | ❌       | 락 해제 시 깨어남
WAITING       | ❌       | 신호 수신 시 깨어남
TIMED_WAITING | ❌       | 타이머 만료 시 깨어남
TERMINATED    | ❌       | 관리 대상 아님
```

---

## ⚖️ 트레이드오프

```
BLOCKED vs WAITING 선택:

synchronized + wait/notify:
  장점: 코드 단순, 추가 의존성 없음
  단점: 정밀 제어 어려움 (어느 스레드를 깨울지 선택 불가)
        notify() 는 임의의 대기 스레드 하나만 깨움

ReentrantLock + Condition + park:
  장점: 다수의 Condition 사용 가능 (생산자/소비자 분리)
        fairness 설정 가능
        tryLock() 타임아웃 지원
  단점: 명시적 lock/unlock (finally 블록 필수)
  → Virtual Thread 환경에서 synchronized 대신 권장 (Pinning 방지)

Spurious Wakeup 처리:
  wait()는 notify 없이도 깨어날 수 있음 (spurious wakeup)
  → 항상 while 루프로 조건 재확인
  
  잘못된 패턴:
    if (queue.isEmpty()) { queue.wait(); }  // if: 한 번만 확인
  
  올바른 패턴:
    while (queue.isEmpty()) { queue.wait(); }  // while: 깨어날 때마다 재확인
```

---

## 📌 핵심 정리

```
Java Thread 상태 핵심:

6가지 상태:
  NEW        → start() 전
  RUNNABLE   → 실행 중 또는 실행 대기 (I/O 대기도 RUNNABLE)
  BLOCKED    → synchronized 락 대기 (오직 모니터 락)
  WAITING    → wait() / park() / join() 무기한 대기
  TIMED_WAITING → sleep() / wait(ms) / park(ns) 시간 제한 대기
  TERMINATED → 종료

중요 구분:
  BLOCKED vs WAITING:
    BLOCKED = synchronized 경합 (락 기다림)
    WAITING = 조건 기다림 (notify, unpark, 종료)
  RUNNABLE ≠ CPU 사용 중:
    I/O 대기 중 스레드도 RUNNABLE로 표시

sleep vs wait:
  sleep: 락 유지 + TIMED_WAITING
  wait:  락 반납 + WAITING (synchronized 안에서만)

Thread Dump 해석:
  BLOCKED 많음 → synchronized 경합 → 락 보유 스레드 추적
  WAITING 많음 → 조건 대기 (정상 가능)
  RUNNABLE 많고 CPU 낮음 → I/O 대기 중
```

---

## 🤔 생각해볼 문제

**Q1.** `Thread.interrupt()`를 호출하면 무슨 일이 일어나는가? `WAITING` 상태의 스레드와 `RUNNABLE` 상태의 스레드가 다르게 반응하는 이유는?

<details>
<summary>해설 보기</summary>

`Thread.interrupt()`는 해당 스레드의 **인터럽트 플래그를 set**한다.

**WAITING/TIMED_WAITING 상태의 스레드**: `wait()`, `sleep()`, `join()`, `park()` 중인 경우 즉시 깨어나고 `InterruptedException`이 발생한다(park는 예외 없이 인터럽트 상태만 set). 인터럽트 플래그는 clear된다.

**RUNNABLE 상태의 스레드**: 즉시 중단되지 않는다. 인터럽트 플래그만 set되며, 스레드 코드에서 `Thread.currentThread().isInterrupted()`로 주기적으로 확인해야 한다.

```java
// RUNNABLE 스레드에서 인터럽트를 처리하는 올바른 패턴
while (!Thread.currentThread().isInterrupted()) {
    // 작업 수행
}
// 또는 InterruptedException을 다시 던지거나 플래그 재설정
```

인터럽트는 "종료 요청"을 협력적으로 전달하는 메커니즘이며, 강제 종료가 아님을 항상 기억해야 한다.

</details>

---

**Q2.** Thread Dump에서 데드락을 찾는 방법은? `Found one Java-level deadlock` 메시지가 나타나는 조건은 무엇인가?

<details>
<summary>해설 보기</summary>

`jstack`은 스레드 간 락 보유/대기 관계를 분석하여 순환 대기 사이클을 자동으로 감지하고 "Found one Java-level deadlock:" 메시지를 출력한다. 이는 **synchronized 모니터 락에 대해서만** 자동 감지된다.

데드락 발생 조건 (4가지 모두 충족 시):
1. 상호 배제: 락은 한 스레드만 보유 가능
2. 점유 대기: 락을 보유한 채로 다른 락 대기
3. 비선점: 락을 강제로 빼앗을 수 없음
4. 순환 대기: A→B→C→A 방식의 순환

ReentrantLock으로 인한 데드락은 `jstack`이 자동으로 찾지 못할 수 있다. 이 경우 `WAITING (parking)` 스레드들의 대기 체인을 수동으로 추적해야 한다.

`jcmd <pid> Thread.print` 또는 JFR의 `jdk.JavaMonitorDeadlock` 이벤트로도 감지 가능하다.

</details>

---

**Q3.** `notifyAll()` 대신 `notify()`를 사용하면 어떤 문제가 생길 수 있는가?

<details>
<summary>해설 보기</summary>

`notify()`는 wait 중인 스레드 중 **임의의 하나만** 깨운다. 같은 모니터에서 서로 다른 조건으로 wait 중인 스레드가 섞여 있으면, 엉뚱한 스레드가 깨어나 조건이 맞지 않아 다시 wait하는 상황이 발생한다. 이 경우 깨어나야 할 스레드는 영원히 깨어나지 못하는 **기아(Starvation)** 문제가 생긴다.

`notifyAll()`은 모든 대기 스레드를 깨우므로 기아 문제를 방지하지만, 대부분의 스레드가 다시 wait로 돌아가는 "thundering herd" 현상이 발생할 수 있다.

해결책으로 `ReentrantLock`과 복수의 `Condition` 객체를 사용하면 "생산자만 깨우기", "소비자만 깨우기"처럼 정밀한 제어가 가능하다. `ArrayBlockingQueue`가 내부적으로 `notFull`과 `notEmpty` 두 Condition을 분리해 사용하는 이유다.

</details>

---

<div align="center">

**[⬅️ 이전: Thread Pool](./03-thread-pool-executor.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데몬 스레드와 JVM 종료 ➡️](./05-daemon-thread-jvm-shutdown.md)**

</div>
