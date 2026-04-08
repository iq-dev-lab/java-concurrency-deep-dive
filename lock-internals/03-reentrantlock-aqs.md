# ReentrantLock 내부 — AQS 대기 큐 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AbstractQueuedSynchronizer`의 CLH 변형 큐는 어떤 구조인가?
- `state` 필드를 CAS로 업데이트하며 락을 획득/해제하는 과정은?
- Fair Lock이 큐 순서를 보장하는 방식과 Non-Fair Lock보다 느린 이유는?
- `LockSupport.park()` / `unpark()`로 스레드를 멈추고 깨우는 원리는?
- `synchronized`와 `ReentrantLock`의 설계 목표는 어디서 갈리는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`ReentrantLock`은 `synchronized`가 제공하지 않는 `tryLock()`, `lockInterruptibly()`, `Condition`, Fair/NonFair 선택을 제공한다. 이 기능들이 AQS 구조 위에서 어떻게 구현되는지 이해하면, 어떤 상황에서 어느 것을 선택해야 할지 근거 있게 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: ReentrantLock을 항상 synchronized보다 낫다고 오해
  "ReentrantLock이 더 유연하니까 항상 이걸 쓴다"
  → 경쟁 없는 상황: synchronized Thin Lock ≈ ReentrantLock (성능 유사)
  → JIT Lock Elision: synchronized에 더 공격적으로 적용됨
  → 코드 복잡성: try-finally 필수, 실수 시 락 영구 점유

실수 2: Fair Lock이 공정하면서도 빠를 거라는 기대
  new ReentrantLock(true);  // Fair Lock
  → 공정하지만 처리량이 Non-Fair 대비 최대 6배 감소
  → unpark된 스레드가 스케줄러에 의해 깨어나는 동안
    다른 스레드가 CPU를 놀릴 수밖에 없음 (Barging 불가)
  → 특별한 이유 없으면 Non-Fair 사용

실수 3: finally 블록 누락
  lock.lock();
  doWork();       // 예외 발생 시 unlock 영원히 안 됨
  lock.unlock();  // 도달 못 함 → 락 영구 점유 → 데드락

  올바른 패턴:
  lock.lock();
  try { doWork(); }
  finally { lock.unlock(); }  // 반드시 finally에
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// ReentrantLock 올바른 사용 패턴

ReentrantLock lock = new ReentrantLock();  // Non-Fair (기본값)

// 패턴 1: 기본 락/언락
lock.lock();
try {
    // 임계 구역
} finally {
    lock.unlock();  // 반드시 finally
}

// 패턴 2: 타임아웃 tryLock (대기 시간 제한 → 데드락 회피)
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        // 임계 구역
    } finally {
        lock.unlock();
    }
} else {
    // 타임아웃: 다른 처리 또는 재시도
}

// 패턴 3: 인터럽트 가능한 락 (취소 가능 작업)
try {
    lock.lockInterruptibly();
    try {
        // 임계 구역
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 플래그 복원
    // 인터럽트 처리
}

// 패턴 4: 비블로킹 시도
if (lock.tryLock()) {
    try { /* 임계 구역 */ }
    finally { lock.unlock(); }
} else {
    // 즉시 실패 처리
}
```

---

## 🔬 내부 동작 원리

### 1. AQS 전체 구조

```
AbstractQueuedSynchronizer (AQS):
  java.util.concurrent.locks 패키지의 골격 프레임워크
  ReentrantLock, Semaphore, CountDownLatch, CyclicBarrier,
  ReentrantReadWriteLock 등의 공통 기반

핵심 필드:
  volatile int state;     // 동기화 상태
                          //   ReentrantLock: 0=미보유, 1+=재진입 횟수
                          //   Semaphore: 남은 허가 수
                          //   CountDownLatch: 남은 카운트
  volatile Node head;     // CLH 큐 헤드 (더미 노드)
  volatile Node tail;     // CLH 큐 테일

Node 구조체:
  volatile int waitStatus;
    CANCELLED( 1): 타임아웃/인터럽트로 취소된 노드
    SIGNAL   (-1): 다음 노드(후임)를 깨워야 함
    CONDITION(-2): Condition 큐에 있는 노드
    PROPAGATE(-3): 공유 락 상태 전파 (ReadLock, Semaphore)
    0:             초기값

  volatile Node prev;     // 이전 노드
  volatile Node next;     // 다음 노드
  volatile Thread thread; // 이 노드를 소유한 대기 스레드

CLH 변형 큐 시각화:
  [head(dummy)] ←→ [Node(T1,SIGNAL)] ←→ [Node(T2,SIGNAL)] ←→ [tail=Node(T3,0)]

  head: 현재 락을 보유 중이거나 방금 해제한 스레드 (더미)
  head.next: 다음으로 락을 받을 첫 번째 대기 스레드
  tail: 가장 최근에 큐에 삽입된 노드
```

### 2. 락 획득 상세 과정 (Non-Fair)

```
ReentrantLock.lock()
  → NonfairSync.lock()
  → 내부적으로 AQS.acquire(1)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1단계: 즉시 획득 시도 (Barging)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CAS(state, 0, 1)
  → 성공: setExclusiveOwnerThread(currentThread) → 락 획득 완료
  → 실패: 2단계

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2단계: 재진입 확인
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  getExclusiveOwnerThread() == currentThread?
  → YES: state++ (재진입 카운터 증가) → 락 획득 완료
  → NO: 3단계

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
3단계: 큐에 삽입 후 대기
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ① 새 Node 생성: Node(currentThread, EXCLUSIVE)
  ② CAS로 테일에 추가 (실패 시 루프 재시도)
  ③ 앞 노드의 waitStatus = SIGNAL 설정 ("나를 깨워줘")
  ④ LockSupport.park(this) → Thread: WAITING(parking)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
4단계: 깨어난 후 CAS 재시도
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  LockSupport.unpark(this) → 깨어남
  head.next == this? → CAS(state, 0, 1) 시도
  → 성공: 노드를 새로운 head로 설정, 락 획득
  → 실패: 다시 park

락 해제 흐름:
  ReentrantLock.unlock()
  → AQS.release(1)
  → state-- (재진입이면 반복)
  → state == 0: setExclusiveOwnerThread(null)
  → head.waitStatus == SIGNAL: LockSupport.unpark(head.next.thread)
```

### 3. Fair Lock vs Non-Fair Lock

```
Non-Fair Lock (기본값, new ReentrantLock()):

  lock() 진입 시:
    큐를 확인하지 않고 즉시 CAS 시도 (Barging)
    큐에 대기자가 있어도 새 스레드가 먼저 락 획득 가능

  왜 처리량이 높은가:
    unpark(T1) 호출 → OS가 T1을 깨우는 데 ~수 μs 소요
    그 사이에 T_new가 lock() 호출 → 즉시 CAS 성공
    T1이 깨어났을 때: CAS 재시도 → 실패 → 다시 park
    결과: 불필요한 스케줄러 개입 없이 CPU 사용 지속

  단점: 대기 큐의 스레드가 계속 새 스레드에 밀릴 수 있음 (기아)

Fair Lock (new ReentrantLock(true)):

  lock() 진입 시:
    hasQueuedPredecessors() 호출
    큐에 앞선 대기 스레드 있으면 → 즉시 큐 삽입 (Barging 불가)
    큐가 비었고 CAS 성공 시에만 즉시 획득

  결과: FIFO 순서 보장
  단점:
    hasQueuedPredecessors() 자체 비용
    모든 스레드가 park/unpark 사이클을 거침
    → Non-Fair 대비 처리량 ~4~6배 낮음 (JMH 실측)

  tryLock() 예외:
    Fair Lock 설정에서도 tryLock()은 Non-Fair 방식
    (큐 앞선 대기자 있어도 즉시 획득 시도)
    → 명세에 명시: "공정성 정책을 따르지 않는다"
    → tryLock(0, NANOSECONDS)는 Fair 정책 적용

실측 처리량 비교 (8스레드, JMH):
  Non-Fair: ~3,200,000 ops/ms
  Fair:     ~550,000 ops/ms  (약 5.8배 차이)
```

### 4. LockSupport.park() / unpark() 내부 구조

```
LockSupport.park(Object blocker):
  내부: UNSAFE.park(false, 0L)
  OS 레벨: Linux futex_wait() 시스템콜
  스레드 상태: RUNNABLE → WAITING (parking)
  Thread Dump 출력: "WAITING (parking)"
                    "parking to wait for <lock address>"

  blocker 파라미터:
    jstack Thread Dump에 표시되는 힌트 객체
    null이면 "WAITING (parking)" (어디서 기다리는지 모름)
    this 전달하면 AQS 주소가 표시 → 디버깅 용이

LockSupport.unpark(Thread t):
  내부: UNSAFE.unpark(t)
  OS 레벨: Linux futex_wake() 시스템콜
  특징: "permit" 방식
    unpark(t)를 미리 호출해두면
    다음 park(t) 호출이 즉시 반환 (permit 소비)
    permit은 최대 1개만 적립 (unpark 여러 번 호출 = 1회만 효과)

  Object.notify() 와의 차이:
    notify: 특정 스레드 지정 불가 (임의 선택)
    unpark: 특정 스레드 직접 지정 → AQS가 큐 순서 제어 가능

park에서 깨어나는 3가지 경우:
  ① LockSupport.unpark(thisThread) 호출
  ② thisThread.interrupt() 호출 (InterruptedException 없음, 플래그만 set)
  ③ Spurious Wakeup (허위 깨어남, 드물지만 존재)
  → AQS는 깨어난 후 반드시 조건(CAS 가능 여부) 재확인
```

---

## 💻 실전 실험

### 실험 1: Fair vs Non-Fair 처리량 비교

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Group)
@Fork(2)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class FairVsNonFairBenchmark {

    ReentrantLock fair    = new ReentrantLock(true);
    ReentrantLock nonFair = new ReentrantLock(false);
    int counter = 0;

    @Benchmark @Group("fair") @GroupThreads(8)
    public int fairLock() {
        fair.lock();
        try { return ++counter; }
        finally { fair.unlock(); }
    }

    @Benchmark @Group("nonfair") @GroupThreads(8)
    public int nonFairLock() {
        nonFair.lock();
        try { return ++counter; }
        finally { nonFair.unlock(); }
    }
}
// 결과: nonfair가 fair보다 ~5~6배 처리량 높음
```

### 실험 2: AQS 큐 상태 실시간 모니터링

```java
import java.util.concurrent.locks.ReentrantLock;

public class AqsQueueMonitor {
    static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {
        // 10개 스레드 경쟁
        for (int i = 0; i < 10; i++) {
            final int id = i;
            new Thread(() -> {
                System.out.printf("T%d: 시도, 큐길이=%d%n",
                    id, lock.getQueueLength());
                lock.lock();
                try {
                    System.out.printf("T%d: 획득, 큐길이=%d, 재진입=%d%n",
                        id, lock.getQueueLength(), lock.getHoldCount());
                    Thread.sleep(80);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    lock.unlock();
                }
            }, "T-" + i).start();
        }

        // 모니터링
        for (int i = 0; i < 10; i++) {
            Thread.sleep(100);
            System.out.printf("[Monitor] 큐길이=%d, 잠금=%b, 대기스레드=%s%n",
                lock.getQueueLength(),
                lock.isLocked(),
                lock.getQueuedThreads().stream()
                    .map(Thread::getName).toList());
        }
    }
}
```

### 실험 3: tryLock 타임아웃으로 데드락 회피

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class DeadlockAvoidance {
    static final ReentrantLock lock1 = new ReentrantLock();
    static final ReentrantLock lock2 = new ReentrantLock();

    static void safeTransfer(ReentrantLock from, ReentrantLock to,
                              String name) throws InterruptedException {
        while (true) {
            if (from.tryLock(50, TimeUnit.MILLISECONDS)) {
                try {
                    if (to.tryLock(50, TimeUnit.MILLISECONDS)) {
                        try {
                            System.out.println(name + ": 두 락 모두 획득, 작업 수행");
                            Thread.sleep(10);
                            return;
                        } finally { to.unlock(); }
                    }
                } finally { from.unlock(); }
            }
            // 둘 다 못 얻음 → 잠깐 대기 후 재시도 (데드락 없음)
            Thread.sleep(10 + (int)(Math.random() * 10));
            System.out.println(name + ": 재시도...");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // 일반 lock()으로는 데드락 가능한 패턴
        // tryLock으로 안전하게 처리
        Thread t1 = new Thread(() -> {
            try { safeTransfer(lock1, lock2, "T1"); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        Thread t2 = new Thread(() -> {
            try { safeTransfer(lock2, lock1, "T2"); }  // 순서 반대
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println("완료 (데드락 없음)");
    }
}
```

---

## 📊 성능/비용 비교

```
ReentrantLock vs synchronized 기능/성능 비교:

기능                     | synchronized | ReentrantLock
────────────────────────┼─────────────┼─────────────────────────
타임아웃 tryLock          | ❌           | ✅ tryLock(t, unit)
인터럽트 가능 대기         | ❌           | ✅ lockInterruptibly()
공정성 선택              | ❌ (항상 NF)  | ✅ new ReentrantLock(fair)
다수 Condition           | ❌ (1개)      | ✅ newCondition() 여러 개
락 상태 조회             | ❌           | ✅ isLocked(), getQueueLength()
재진입                   | ✅           | ✅
JIT Lock Elision         | ✅ 적극 적용  | 제한적
코드 단순성              | ✅           | ❌ try-finally 필수

성능 비교 (8스레드, JMH):
                        | synchronized | ReentrantLock(NF) | ReentrantLock(F)
────────────────────────┼─────────────┼───────────────────┼──────────────────
무경쟁 (1스레드)          | 100%        | ~95%              | ~90%
약한 경쟁 (2~4스레드)     | 100%        | ~100~110%         | ~40~60%
강한 경쟁 (8+스레드)      | 100%        | ~100~115%         | ~15~25%

→ 무경쟁: synchronized가 JIT Elision으로 가장 빠를 수 있음
→ 약한~강한 경쟁: 거의 비슷하거나 NF가 약간 유리
→ Fair Lock은 모든 구간에서 처리량 낮음
```

---

## ⚖️ 트레이드오프

```
synchronized vs ReentrantLock 선택 기준:

synchronized 선택:
  단순 상호 배제, 추가 기능 불필요
  JIT Lock Elision 최대 활용 원할 때
  Virtual Thread 없는 환경 (Pinning 무관)
  코드 가독성/단순성 중요

ReentrantLock 선택:
  tryLock(timeout): 데드락 회피, 응답성 우선 서비스
  lockInterruptibly(): 취소 가능한 장기 작업
  여러 Condition: 생산자/소비자 큐 분리
  락 상태 모니터링: 운영 진단 지표 필요
  Virtual Thread 환경 (Pinning 방지)

Fair Lock은 언제:
  특정 스레드가 계속 밀려 SLA를 못 맞추는 경우
  처리량보다 응답시간 편차 최소화가 중요한 경우
  그 외 대부분: Non-Fair

Virtual Thread (Java 21) 관점:
  synchronized 블록 내 블로킹 I/O → 캐리어 스레드 Pinning
  → 해당 Virtual Thread가 Unmount 불가 → 효율 저하
  ReentrantLock → park/unpark → Virtual Thread Unmount 가능
  → I/O 집약적 Virtual Thread 환경: ReentrantLock 권장
  (Chapter 6에서 상세 분석)
```

---

## 📌 핵심 정리

```
ReentrantLock / AQS 핵심:

AQS 구조:
  volatile int state (락 상태: 0=미보유, N=재진입 횟수)
  CLH 변형 큐: head(dummy) ↔ Node(T1) ↔ Node(T2) ↔ tail
  각 Node: thread, prev/next, waitStatus(SIGNAL/CANCELLED/...)

락 획득 흐름:
  CAS(state, 0, 1) 성공 → 즉시 획득
  실패 → 큐 삽입 → LockSupport.park() → unpark → CAS 재시도

Fair vs Non-Fair:
  Non-Fair: 새 스레드가 Barging 가능 → 처리량 높음, 기아 가능
  Fair:     FIFO 순서 보장 → 처리량 ~5배 낮음

park/unpark:
  park(blocker): 특정 객체를 힌트로 WAITING 상태
  unpark(thread): 특정 스레드 직접 지정해서 깨움
  permit 방식: 미리 unpark → 다음 park 즉시 반환

사용 원칙:
  기본: synchronized (단순)
  타임아웃/인터럽트/Condition/VirtualThread: ReentrantLock
  Fair Lock: 특별한 이유 없으면 사용 안 함
  ReentrantLock: try-finally 반드시 사용
```

---

## 🤔 생각해볼 문제

**Q1.** `ReentrantLock.tryLock()`은 Fair Lock 설정에서도 큐를 무시하고 즉시 획득 시도를 하는가?

<details>
<summary>해설 보기</summary>

그렇다. `tryLock()` (타임아웃 없는 버전)은 Fair Lock 설정과 무관하게 `nonfairTryAcquire()`를 직접 호출한다. 큐에 대기자가 있어도 즉시 CAS를 시도하고, 성공하면 바로 락을 획득한다.

이것은 의도된 설계다. `tryLock()`의 계약이 "지금 당장 락을 얻을 수 있으면 얻고, 아니면 즉시 false"이기 때문에, 공정성을 지키면서 대기하는 것 자체가 의미에 부합하지 않는다.

공정한 방식의 tryLock을 원하면 `tryLock(0, TimeUnit.NANOSECONDS)`를 사용해야 한다. 이 버전은 `hasQueuedPredecessors()`를 확인하는 Fair 로직을 거친다.

</details>

---

**Q2.** `CountDownLatch`와 `Semaphore`도 AQS 기반이다. `state` 필드를 어떻게 다르게 사용하는가?

<details>
<summary>해설 보기</summary>

AQS는 `state`의 의미를 하위 구현체가 자유롭게 정의하는 Template Method 패턴을 따른다.

- **`ReentrantLock`**: state=0이면 미획득, state≥1이면 재진입 횟수
- **`CountDownLatch(n)`**: state=n으로 시작. `countDown()`마다 CAS로 state-1. state=0이 되면 모든 `await()` 스레드를 PROPAGATE로 한꺼번에 해제
- **`Semaphore(n)`**: state=n (허가 수). `acquire()`마다 state-1, `release()`마다 state+1. state=0이면 `acquire()`가 park
- **`ReentrantReadWriteLock`**: state의 상위 16비트=읽기 보유 수, 하위 16비트=쓰기 재진입 카운터

AQS 자체는 큐 관리(enqueue/dequeue), park/unpark 인프라, CAS 헬퍼만 제공한다. "어떤 상태가 락 획득을 허용하는가"는 하위 클래스의 `tryAcquire()`/`tryRelease()` 오버라이드가 결정한다.

</details>

---

**Q3.** `lockInterruptibly()`로 대기 중 인터럽트가 오면, AQS 큐의 노드는 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

1. `park()`에서 인터럽트로 깨어남
2. `Thread.interrupted()`로 인터럽트 플래그 확인
3. `cancelAcquire(node)` 호출:
   - 해당 노드의 `waitStatus`를 `CANCELLED(1)`로 설정
   - 큐에서 노드를 논리적으로 제거 (앞뒤 연결 재구성)
   - CANCELLED 노드는 즉시 물리 제거되지 않고, 이후 다른 스레드의 큐 탐색 시 건너뜀
4. `InterruptedException` throw

이와 달리 `lock()` (인터럽트 불가)은 park에서 인터럽트로 깨어나도 인터럽트 상태를 로컬에 기억해두고 다시 park 한다. 락을 최종적으로 획득한 후 `Thread.currentThread().interrupt()`로 인터럽트 상태를 복원한다.

CANCELLED 노드의 lazy 제거는 AQS 큐가 완벽한 FIFO가 아닌 "CLH 변형"이라 불리는 이유 중 하나다.

</details>

---

<div align="center">

**[⬅️ 이전: synchronized 락 확장](./02-synchronized-lock-escalation.md)** | **[홈으로 🏠](../README.md)** | **[다음: ReadWriteLock ➡️](./04-read-write-lock.md)**

</div>
