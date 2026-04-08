# 조건 변수 — wait/notify vs Condition, Spurious Wakeup

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Object.wait()` / `notify()` / `notifyAll()`의 모니터 락 연결 구조는?
- `Condition.await()` / `signal()`이 `wait()`/`notify()`보다 세밀한 제어를 하는 원리는?
- Spurious Wakeup이 왜 발생하며 `while` 루프로 어떻게 방어하는가?
- `ArrayBlockingQueue`가 두 개의 `Condition`을 사용하는 이유는?
- 생산자-소비자 패턴을 `Condition`으로 구현하는 올바른 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`BlockingQueue`, `Semaphore`, `CountDownLatch`의 내부는 결국 "어떤 조건이 충족될 때까지 기다리는" 패턴이다. 이 패턴의 올바른 구현이 조건 변수다. `wait()`/`notify()`의 모니터 락 연결과 Spurious Wakeup 방어를 이해하지 못하면, 직접 구현한 큐나 동기화 코드에 미묘한 버그가 잠복한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: wait()를 if로 감싸기 (Spurious Wakeup 미방어)
  synchronized (queue) {
      if (queue.isEmpty()) {  // if: 한 번만 확인
          queue.wait();
      }
      process(queue.poll());  // Spurious Wakeup 시 isEmpty일 수 있음!
  }

실수 2: synchronized 밖에서 wait() 호출
  queue.wait();  // IllegalMonitorStateException!
  // wait()는 반드시 해당 객체의 synchronized 블록 내에서 호출

실수 3: 단일 조건에 notifyAll() 남발
  // 생산자/소비자가 같은 객체로 wait
  notifyAll();  // 생산자도, 소비자도 전부 깨움
  // 깨어난 스레드 대부분이 조건 불충족 → 다시 wait
  // 불필요한 컨텍스트 스위칭 폭발

실수 4: Condition 미사용으로 notify 대상 제어 불가
  // 생산자: notFull 조건을 기다려야 함
  // 소비자: notEmpty 조건을 기다려야 함
  // 하지만 단일 모니터로 관리하면 분리 불가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 생산자-소비자: ReentrantLock + 두 Condition

class BoundedBuffer<T> {
    private final Object[] items;
    private int putIndex, takeIndex, count;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();  // 생산자 대기
    private final Condition notEmpty = lock.newCondition();  // 소비자 대기

    BoundedBuffer(int capacity) {
        items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();  // 버퍼 가득 참 → 생산자 대기
            }
            items[putIndex] = item;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            notEmpty.signal();  // 소비자만 깨움 (생산자 깨우지 않음)
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 버퍼 비어있음 → 소비자 대기
            }
            T item = (T) items[takeIndex];
            items[takeIndex] = null;
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            notFull.signal();  // 생산자만 깨움 (소비자 깨우지 않음)
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Object.wait() / notify()의 모니터 연결 구조

```
모니터 락과 wait/notify의 관계:

모든 Java 객체는 하나의 모니터(Monitor)를 가짐
  모니터 = synchronized 진입을 위한 락 + wait set

Object.wait() 동작:
  ① 현재 스레드가 해당 객체의 모니터 락을 보유해야 함
     (없으면 IllegalMonitorStateException)
  ② 모니터 락을 해제 (다른 스레드가 synchronized 진입 가능)
  ③ 현재 스레드를 Wait Set에 추가
  ④ WAITING 상태 진입 (park())
  ⑤ notify() 또는 notifyAll() 수신 시
  ⑥ Wait Set에서 Entry Set으로 이동 (락 획득 대기)
  ⑦ 다시 모니터 락 획득 (다른 스레드와 경쟁)
  ⑧ 락 획득 후 wait() 호출 지점부터 실행 재개

Object.notify():
  Wait Set에서 임의의 스레드 1개를 Entry Set으로 이동
  → 이동된 스레드가 락 획득 경쟁 참여

Object.notifyAll():
  Wait Set의 모든 스레드를 Entry Set으로 이동
  → 모두가 락 획득 경쟁 참여

┌──────────────────────────────────────────────────┐
│         Object obj 의 모니터                      │
│                                                  │
│  [Entry Set]          [Wait Set]                 │
│  Thread B (락 대기)    Thread C (wait 중)          │
│  Thread D (락 대기)    Thread E (wait 중)          │
│                                                  │
│  현재 락 소유: Thread A                            │
└──────────────────────────────────────────────────┘
  notify() → Wait Set의 하나(C 또는 E) → Entry Set 이동
  notifyAll() → C, E 모두 → Entry Set 이동
```

### 2. Condition — wait/notify의 세밀한 버전

```
Condition은 ReentrantLock에 연결된 조건 변수:
  Lock lock = new ReentrantLock();
  Condition cond = lock.newCondition();

  Lock 하나에 여러 Condition 생성 가능
  각 Condition은 독립적인 대기 큐(AQS의 Condition Queue)를 가짐

Condition.await() 동작:
  ① lock.lock()으로 획득한 Lock이 있어야 함
  ② AQS의 Condition Queue에 현재 스레드 추가
  ③ Lock 해제 (LockSupport.park() 기반)
  ④ signal() 수신 시 Condition Queue → Sync Queue 이동
  ⑤ Lock 재획득 후 실행 재개

Object.wait() vs Condition.await():

비교 항목          | Object.wait()         | Condition.await()
──────────────────┼──────────────────────┼──────────────────────────
연결 락            | synchronized (Object) | ReentrantLock
하나의 객체에 조건 수 | 1개 (모니터 Wait Set)  | 여러 개 (newCondition)
타임아웃            | wait(ms)              | await(ms, unit)
인터럽트 처리       | InterruptedException  | awaitUninterruptibly() 지원
Spurious Wakeup   | 발생 가능              | 발생 가능 (while 루프 필요)

예시: 생산자/소비자에서 notify 대상 선택 불가 문제
  // Object.wait()/notify() 방식
  Object lock = new Object();
  synchronized (lock) {
      lock.notifyAll();  // 생산자도 소비자도 전부 깨움
      // 깨어난 생산자가 조건 불충족 → 다시 wait
      // → 불필요한 컨텍스트 스위칭
  }

  // Condition 방식
  Condition notFull  = lock.newCondition();
  Condition notEmpty = lock.newCondition();
  notFull.signal();   // 생산자만 깨움
  notEmpty.signal();  // 소비자만 깨움
  → 불필요한 컨텍스트 스위칭 없음
```

### 3. Spurious Wakeup — 왜 발생하고 어떻게 방어하는가

```
Spurious Wakeup (가짜 깨어남):
  notify()나 notifyAll() 없이 wait()에서 깨어나는 현상

발생 원인 (OS/하드웨어 수준):
  ① POSIX pthread_cond_wait의 스펙:
     "spurious wakeups may occur even without a signal"
     → OS의 조건 변수 구현이 허용하는 동작
  ② Linux futex의 특성:
     신호 처리, 스레드 이동 등 OS 내부 이벤트로 깨어날 수 있음
  ③ JVM 구현 상의 특성:
     특정 하드웨어/OS에서 park() 자체가 간헐적으로 반환

얼마나 자주 발생하는가:
  x86 Linux에서 실제로 매우 드뭄 (경험적으로 거의 발생 안 함)
  하지만 JVM 스펙이 허용하므로 방어해야 함
  ARM, SPARC 등 다른 아키텍처에서 더 자주 발생

방어: while 루프로 조건 재확인
  // 잘못된 패턴 (if 사용):
  synchronized (queue) {
      if (queue.isEmpty()) {
          queue.wait();
      }
      // Spurious Wakeup 시 isEmpty일 수 있음 → NPE 또는 오작동
      Object item = queue.poll();
  }

  // 올바른 패턴 (while 사용):
  synchronized (queue) {
      while (queue.isEmpty()) {  // 깨어날 때마다 조건 재확인
          queue.wait();
      }
      Object item = queue.poll();  // isEmpty가 아님이 보장됨
  }

  while 루프의 부가 효과:
  ① Spurious Wakeup 방어
  ② notifyAll() 후 경쟁에서 진 스레드 재대기
     (여러 스레드가 깨어나도 실제로 실행할 수 있는 것은 락 획득한 하나)
  ③ 조건이 변경됐다가 다시 false가 된 경우 재대기
```

### 4. ArrayBlockingQueue 내부 구현

```
ArrayBlockingQueue는 Condition 두 개를 사용하는 대표적 예시:

class ArrayBlockingQueue<E> {
    final Object[] items;
    int takeIndex, putIndex, count;
    
    final ReentrantLock lock;
    private final Condition notEmpty;  // 소비자 대기 큐
    private final Condition notFull;   // 생산자 대기 큐

    ArrayBlockingQueue(int capacity, boolean fair) {
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull  = lock.newCondition();
    }

    public void put(E e) throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();   // 가득 참 → 생산자 대기
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }

    private void enqueue(E e) {
        items[putIndex] = e;
        if (++putIndex == items.length) putIndex = 0;
        count++;
        notEmpty.signal();  // 소비자 한 명만 깨움
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();  // 비어있음 → 소비자 대기
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    private E dequeue() {
        E e = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        notFull.signal();  // 생산자 한 명만 깨움
        return e;
    }
}

핵심: notFull.signal()은 생산자만, notEmpty.signal()은 소비자만 깨움
→ Object.notifyAll()처럼 모두를 깨우는 낭비 없음
→ 불필요한 컨텍스트 스위칭 최소화
```

---

## 💻 실전 실험

### 실험 1: Spurious Wakeup 방어 필요성 확인

```java
import java.util.LinkedList;
import java.util.Queue;

public class SpuriousWakeupTest {
    static final Queue<Integer> queue = new LinkedList<>();
    static final Object lock = new Object();
    static volatile boolean done = false;

    public static void main(String[] args) throws InterruptedException {
        // 소비자 스레드
        Thread consumer = new Thread(() -> {
            synchronized (lock) {
                try {
                    while (!done || !queue.isEmpty()) {
                        while (queue.isEmpty() && !done) {
                            lock.wait();  // while 루프로 Spurious Wakeup 방어
                        }
                        if (!queue.isEmpty()) {
                            int item = queue.poll();
                            System.out.println("소비: " + item);
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });

        consumer.start();

        // 생산자
        for (int i = 0; i < 5; i++) {
            synchronized (lock) {
                queue.offer(i);
                lock.notify();  // 소비자 하나 깨움
            }
            Thread.sleep(100);
        }

        synchronized (lock) {
            done = true;
            lock.notifyAll();
        }
        consumer.join();
    }
}
```

### 실험 2: 단일 Condition vs 두 개 Condition 성능 비교

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class ConditionBenchmark {
    private static final int CAPACITY = 100;

    // 단일 Condition (notifyAll)
    final Object singleCondLock = new Object();
    final java.util.ArrayDeque<Integer> singleQueue = new java.util.ArrayDeque<>();

    @Benchmark @Group("single") @GroupThreads(4)
    public void singleCond_produce() throws InterruptedException {
        synchronized (singleCondLock) {
            while (singleQueue.size() >= CAPACITY) singleCondLock.wait();
            singleQueue.offer(1);
            singleCondLock.notifyAll();  // 생산자/소비자 모두 깨움
        }
    }

    @Benchmark @Group("single") @GroupThreads(4)
    public Integer singleCond_consume() throws InterruptedException {
        synchronized (singleCondLock) {
            while (singleQueue.isEmpty()) singleCondLock.wait();
            Integer v = singleQueue.poll();
            singleCondLock.notifyAll();
            return v;
        }
    }

    // 두 Condition (signal)
    final ReentrantLock dualLock = new ReentrantLock();
    final Condition notFull2  = dualLock.newCondition();
    final Condition notEmpty2 = dualLock.newCondition();
    final java.util.ArrayDeque<Integer> dualQueue = new java.util.ArrayDeque<>();

    @Benchmark @Group("dual") @GroupThreads(4)
    public void dualCond_produce() throws InterruptedException {
        dualLock.lock();
        try {
            while (dualQueue.size() >= CAPACITY) notFull2.await();
            dualQueue.offer(1);
            notEmpty2.signal();  // 소비자만 깨움
        } finally { dualLock.unlock(); }
    }

    @Benchmark @Group("dual") @GroupThreads(4)
    public Integer dualCond_consume() throws InterruptedException {
        dualLock.lock();
        try {
            while (dualQueue.isEmpty()) notEmpty2.await();
            Integer v = dualQueue.poll();
            notFull2.signal();  // 생산자만 깨움
            return v;
        } finally { dualLock.unlock(); }
    }
}
// 결과: dual Condition이 single Condition보다 높은 처리량
// notifyAll의 불필요한 컨텍스트 스위칭을 제거
```

---

## 📊 성능/비용 비교

```
notify() vs notifyAll() vs Condition.signal():

방법              | 깨우는 대상      | 비용   | 불필요한 CS  | 권장 상황
─────────────────┼────────────────┼───────┼────────────┼──────────────────
notify()          | Wait Set 임의 1 | 낮음  | 없음        | 단일 조건, 동질 대기자
notifyAll()       | Wait Set 전부   | 높음  | 많음        | 이기종 대기자 (안전)
Condition.signal()| Condition 큐 1개| 낮음  | 없음        | 다중 조건 분리
Condition.signalAll()| Condition 큐 전부| 중간| 있음        | 단일 Condition의 안전

notify() 사용 시 주의:
  Wait Set에 동질 대기자(같은 조건을 기다리는 스레드)만 있을 때 안전
  이기종 대기자(다른 조건을 기다리는 스레드)가 섞이면 잘못된 스레드 깨울 수 있음
  → 이기종 = Condition 분리, 동질 = notify() 사용

Condition을 통한 성능 개선:
  ArrayBlockingQueue(100), 4 생산자 + 4 소비자:
  notifyAll() 방식: ~100,000 ops/s
  signal() 방식:   ~400,000 ops/s (4배 처리량)
  → 불필요한 컨텍스트 스위칭 제거 효과
```

---

## ⚖️ 트레이드오프

```
Object.wait/notify vs Condition 선택:

Object.wait/notify:
  장점: 추가 의존성 없음, synchronized와 함께 단순
  단점: 단일 대기 큐 → 이기종 대기자 분리 불가
       타임아웃 단위: ms만 (Condition은 TimeUnit 지원)
       인터럽트 불가능한 대기 없음

Condition (ReentrantLock 기반):
  장점: 다수 Condition으로 이기종 대기자 분리
       awaitUninterruptibly(), awaitNanos() 지원
       더 정밀한 signal 제어
  단점: try-finally 필수, 코드 복잡

언제 무엇을 쓰는가:
  단순한 생산자-소비자, 단일 조건 → Object.wait/notify
  복잡한 상태 머신, 다중 조건 → Condition
  Virtual Thread 환경 → Condition 권장 (Pinning 방지)

설계 원칙:
  항상 while 루프로 wait/await 감싸기 (Spurious Wakeup 방어)
  동질 대기자: notify() / signal()
  이기종 대기자: notifyAll() 또는 Condition 분리 후 signal()
```

---

## 📌 핵심 정리

```
조건 변수 핵심:

Object.wait() 규칙:
  synchronized 블록 내에서만 호출
  모니터 락 해제 + Wait Set 진입
  notify/notifyAll로 Entry Set 이동 → 락 재획득 후 실행 재개

Spurious Wakeup 방어:
  항상 while 루프: while (조건 불충족) { wait(); }
  if 사용 금지: if (조건 불충족) { wait(); }

Condition vs Object.wait():
  Condition: ReentrantLock에 여러 개 연결 가능
  signal(): 해당 Condition 대기자만 깨움 (타겟 깨우기)
  awaitUninterruptibly(), awaitNanos() 추가 지원

ArrayBlockingQueue 패턴:
  notFull Condition: 생산자 대기
  notEmpty Condition: 소비자 대기
  put 완료 시 notEmpty.signal() (소비자만)
  take 완료 시 notFull.signal() (생산자만)
  → 불필요한 컨텍스트 스위칭 제거
```

---

## 🤔 생각해볼 문제

**Q1.** `Object.wait()`를 `while` 루프로 감싸면 Spurious Wakeup도 방어하고 notifyAll() 후 경쟁에서 진 스레드도 재대기한다. 그렇다면 `notify()` 대신 `notifyAll()`을 항상 쓰면 안전하지 않은가?

<details>
<summary>해설 보기</summary>

안전하지만 성능 비용이 있다. `notifyAll()`은 Wait Set의 모든 스레드를 깨우고, 깨어난 스레드들이 락 획득 경쟁을 한다. 하나만 조건을 충족하더라도 N개가 깨어나고 N-1개가 다시 `wait()`로 돌아간다.

`notifyAll()`을 안전하게 항상 쓸 수 있는 경우: 스레드 수가 적고 성능이 크게 중요하지 않을 때.

`notifyAll()` 대신 `notify()`가 적합한 경우: Wait Set의 모든 스레드가 동일한 조건을 기다리고(동질 대기자), 하나만 처리할 수 있을 때. 예: "아이템 1개 추가됨" → 소비자 1명만 처리 가능 → `notify()`.

`Condition`으로 완전히 해결: 생산자/소비자를 별도 Condition으로 분리하면 `signal()`로 정확히 원하는 대기자만 깨울 수 있다. 이것이 `ArrayBlockingQueue`의 접근 방식이다.

</details>

---

**Q2.** `Condition.awaitUninterruptibly()`는 언제 사용하는가? 인터럽트를 무시하는 것이 안전한 상황은?

<details>
<summary>해설 보기</summary>

`awaitUninterruptibly()`는 인터럽트가 발생해도 대기를 계속하고, 조건이 충족되어 깨어난 후 인터럽트 상태를 복원한다.

사용이 적합한 경우:
- 리소스 정리나 마무리 작업처럼 "어떤 상황에서도 완료해야 하는" 작업
- 인터럽트가 의미 없는 시스템 레벨 대기 (예: JVM shutdown hook 내부)
- 인터럽트 처리를 외부 계층에서 담당하는 경우

주의사항:
- 인터럽트 기반 취소 메커니즘이 동작하지 않음
- 오랫동안 블로킹될 수 있어 응답성 저하
- 일반 애플리케이션 코드에서는 `await()` (인터럽트 가능)를 권장

관련 패턴: 인터럽트 상태를 무시하지 않고, 현재 작업을 완료한 후 인터럽트를 처리하려면 `try-catch`로 `InterruptedException`을 잡고 `Thread.currentThread().interrupt()`로 상태를 복원하는 것이 올바르다.

</details>

---

**Q3.** `Condition.signal()` 후 즉시 `await()`를 다시 호출하면 어떻게 되는가? 신호를 받은 스레드가 실행될 기회가 있는가?

<details>
<summary>해설 보기</summary>

`signal()`은 대기 스레드를 깨우지만, 호출 스레드가 락을 보유한 상태이므로 깨어난 스레드는 즉시 실행될 수 없다. 깨어난 스레드는 AQS Sync Queue(락 대기 큐)에 진입하고, 현재 락을 보유한 스레드가 `unlock()`을 해야 실행될 수 있다.

```java
lock.lock();
try {
    notEmpty.signal();  // 소비자를 Condition 큐에서 Sync 큐로 이동
    notEmpty.await();   // 현재 스레드가 다시 Condition 큐로 진입, 락 해제
    // 이제 소비자가 락을 획득하고 실행
} finally {
    lock.unlock();
}
```

`signal()` 이후 `await()`를 연속 호출하면:
1. 깨어난 소비자가 Sync Queue에 대기
2. 현재 스레드가 `await()`로 Condition Queue에 진입하고 락 해제
3. 소비자가 락을 획득하고 실행
4. 소비자가 `signal()`로 현재 스레드를 깨우면 현재 스레드가 다시 실행

이는 생산자-소비자 패턴에서 자연스러운 흐름이다.

</details>

---

<div align="center">

**[⬅️ 이전: StampedLock](./05-stamped-lock.md)** | **[홈으로 🏠](../README.md)** | **[다음: 락 성능 비교 ➡️](./07-lock-performance-benchmark.md)**

</div>
