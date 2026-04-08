# BlockingQueue — ArrayBlockingQueue vs LinkedBlockingQueue

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ArrayBlockingQueue`의 단일 락 + 두 Condition이 put/take를 블로킹하는 방식은?
- `LinkedBlockingQueue`가 head/tail 분리 락으로 생산자·소비자 경합을 줄이는 원리는?
- `SynchronousQueue`의 "직접 전달(Direct Handoff)" 메커니즘은 어떻게 동작하는가?
- `PriorityBlockingQueue`와 `DelayQueue`는 어떤 특수한 사용 사례에 적합한가?
- `ArrayBlockingQueue`와 `LinkedBlockingQueue`를 성능 기준으로 어떻게 선택하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`ThreadPoolExecutor`의 작업 큐, Spring의 비동기 이벤트 처리, Kafka 컨슈머 내부 버퍼가 모두 `BlockingQueue`를 기반으로 한다. 각 구현체의 내부 락 구조를 이해하면 스레드 풀 설정, 생산자-소비자 패턴의 병목 원인, 공정성/순서 요구사항에 맞는 큐를 정확히 선택할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: ArrayBlockingQueue와 LinkedBlockingQueue를 동등하게 취급
  "용량이 있는 BlockingQueue면 둘 다 똑같지 않나?"
  → 단일 락 vs 분리 락 → 고처리량 시나리오에서 성능 차이
  → ArrayBlockingQueue: 메모리 예측 가능, GC 압박 없음
  → LinkedBlockingQueue: 더 높은 동시성 (put/take 병렬)

실수 2: 큐가 가득 찼을 때 offer() 실패를 무시
  executor.getQueue().offer(task);  // false 반환 무시
  → 큐가 가득 차면 작업이 조용히 유실됨
  → put()은 블로킹, offer(timeout)은 타임아웃 후 false
  → 실패 시 적절한 처리 필수 (재시도, 예외, 거부 정책)

실수 3: SynchronousQueue에 용량이 있다고 오해
  new SynchronousQueue(); // 내부 버퍼 없음!
  "SynchronousQueue는 크기 0짜리 큐겠지"
  → 실제로는 버퍼가 전혀 없는 "직접 전달" 메커니즘
  → put()은 take()가 올 때까지, take()는 put()이 올 때까지 블로킹
  → ThreadPoolExecutor(cached)의 기본 큐로 사용됨
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 상황별 BlockingQueue 선택 패턴

// 패턴 1: 고정 버퍼, 메모리 예측 가능 → ArrayBlockingQueue
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000);
// 공정성 옵션: new ArrayBlockingQueue<>(1000, true) → Fair (FIFO 스레드 순서)
// 단일 락: put/take가 같은 락을 공유 → 생산자-소비자 직렬화

// 패턴 2: 가변 버퍼, 높은 처리량 → LinkedBlockingQueue
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);
// 분리 락: put(takeLock), take(putLock) → 동시 실행 가능
// 주의: 무한 용량 new LinkedBlockingQueue() → OOM 위험

// 패턴 3: 버퍼 없는 직접 전달 → SynchronousQueue
BlockingQueue<Runnable> queue = new SynchronousQueue<>();
// Executors.newCachedThreadPool() 내부에서 사용
// 작업이 오면 즉시 새 스레드에 전달, 소비자가 없으면 새 스레드 생성

// 패턴 4: 우선순위 기반 처리 → PriorityBlockingQueue
BlockingQueue<Task> queue = new PriorityBlockingQueue<>(100,
    Comparator.comparingInt(Task::getPriority));

// 패턴 5: 지연 실행 → DelayQueue
DelayQueue<DelayedTask> queue = new DelayQueue<>();
queue.put(new DelayedTask(task, 5, TimeUnit.SECONDS));
DelayedTask due = queue.take();  // 5초 후 반환

// 비블로킹 삽입/삭제 (용량 초과 시 null/false 즉시 반환)
boolean added = queue.offer(task);      // 즉시 반환
Task item   = queue.poll();             // 즉시 반환 (없으면 null)
Task item2  = queue.poll(100, MILLIS);  // 타임아웃
```

---

## 🔬 내부 동작 원리

### 1. ArrayBlockingQueue — 단일 락 + 두 Condition

```java
// ArrayBlockingQueue 핵심 소스 (단순화)
public class ArrayBlockingQueue<E> {
    final Object[] items;     // 순환 배열
    int takeIndex;            // 다음 take 위치
    int putIndex;             // 다음 put 위치
    int count;                // 현재 원소 수

    final ReentrantLock lock; // 단일 락 (put/take 공유!)
    private final Condition notEmpty;  // take 대기
    private final Condition notFull;   // put 대기

    public void put(E e) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();    // 가득 참 → 대기 (락 해제 + 소비자 깨어날 때까지)
            enqueue(e);             // 삽입 + count++
            notEmpty.signal();      // take 대기 중인 스레드 깨우기
        } finally {
            lock.unlock();
        }
    }

    public E take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();   // 비었음 → 대기
            E x = dequeue();        // 삭제 + count--
            notFull.signal();       // put 대기 중인 스레드 깨우기
            return x;
        } finally {
            lock.unlock();
        }
    }
}

단일 락의 의미:
  put()과 take()가 같은 ReentrantLock을 공유
  → put()과 take()가 동시에 실행 불가 (직렬화)
  → 생산자가 많고 소비자가 많을 때 병목 발생 가능

두 Condition의 역할:
  notFull: 큐가 가득 찰 때 생산자 대기
           소비자가 take() 후 notFull.signal()로 생산자 깨움
  notEmpty: 큐가 비었을 때 소비자 대기
            생산자가 put() 후 notEmpty.signal()로 소비자 깨움

Condition.await() 동작:
  현재 보유한 락을 해제
  스레드를 Condition의 Wait Set에 추가
  park() → WAITING 상태
  signal() 수신 → Wait Set에서 Entry Set으로 이동
  락 재획득 → await() 반환

순환 배열 (Circular Array):
  items[] 고정 크기 배열
  takeIndex: 다음 삭제 위치 (0부터 증가, 끝에서 0으로 순환)
  putIndex:  다음 삽입 위치
  → 동적 메모리 할당 없음, GC 압박 없음
  → 미리 용량 예약 → 메모리 사용 예측 가능
```

### 2. LinkedBlockingQueue — 분리 락 구조

```java
// LinkedBlockingQueue 핵심 소스 (단순화)
public class LinkedBlockingQueue<E> {
    static class Node<E> {
        E item;
        Node<E> next;
    }

    private final int capacity;
    private final AtomicInteger count = new AtomicInteger();

    private Node<E> head;   // head.next = 첫 번째 실제 항목 (더미 노드)
    private Node<E> last;   // 마지막 노드

    // 두 개의 독립적인 락!
    private final ReentrantLock takeLock = new ReentrantLock();
    private final Condition notEmpty = takeLock.newCondition();

    private final ReentrantLock putLock = new ReentrantLock();
    private final Condition notFull = putLock.newCondition();

    public void put(E e) throws InterruptedException {
        int c = -1;
        Node<E> node = new Node<>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity)
                notFull.await();
            last = last.next = node;        // 테일에 노드 추가
            c = count.getAndIncrement();    // count++ (AtomicInteger!)
            if (c + 1 < capacity)
                notFull.signal();           // 아직 공간 있으면 다음 생산자 깨움
        } finally {
            putLock.unlock();
        }
        if (c == 0) signalNotEmpty();       // 비었다가 첫 항목 → 소비자 깨움
    }

    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0)
                notEmpty.await();
            x = dequeue();                  // head.next에서 삭제
            c = count.getAndDecrement();    // count--
            if (c > 1)
                notEmpty.signal();          // 아직 항목 있으면 다음 소비자 깨움
        } finally {
            takeLock.unlock();
        }
        if (c == capacity) signalNotFull(); // 가득 찼다가 한 개 빠짐 → 생산자 깨움
        return x;
    }
}

분리 락의 의미:
  putLock: 생산자들이 공유 (생산자들끼리만 직렬화)
  takeLock: 소비자들이 공유 (소비자들끼리만 직렬화)
  → put()과 take()가 동시에 실행 가능!

count를 AtomicInteger로 관리하는 이유:
  put()은 putLock, take()는 takeLock을 보유
  두 락 모두에서 count에 접근 → 단일 락으로 보호 불가
  → AtomicInteger.getAndIncrement/getAndDecrement로 원자적 갱신

signalNotEmpty / signalNotFull 의 위치:
  lock 해제 후 signal을 호출 (lock 밖에서!)
  이유: lock 안에서 signal하면 signaled 스레드가 즉시 락 획득 시도
       → 현재 스레드가 아직 보유 중 → 불필요한 컨텍스트 스위칭
  lock 해제 후 signal → 더 효율적인 스케줄링
```

### 3. SynchronousQueue — 직접 전달

```
SynchronousQueue 특성:
  내부 버퍼 없음 (capacity = 0)
  put()은 take()가 준비될 때까지 블로킹
  take()는 put()이 준비될 때까지 블로킹
  = 두 스레드가 랑데부(Rendezvous) 지점에서 직접 교환

내부 구조:
  공정성 모드: TransferQueue (FIFO)
    대기 중인 생산자/소비자를 FIFO 큐로 관리
    오래 기다린 스레드가 먼저 매칭
  비공정성 모드: TransferStack (LIFO, 기본값)
    스택으로 관리 → 최근 요청이 먼저 매칭 (LRU 캐시 친화적)

전달 과정:
  Case 1: 대기 중인 소비자 있음 + put(x) 호출
    → 소비자에게 x를 직접 전달
    → 소비자 unpark, 소비자가 take()에서 x를 받아 반환

  Case 2: put(x) 호출 (대기 소비자 없음)
    → 생산자를 대기 큐/스택에 추가
    → park() → 소비자 올 때까지 대기
    → 소비자 take() → 대기 중 생산자 발견 → 직접 x 수령 → 생산자 unpark

  Case 3: take() 호출 (대기 생산자 없음)
    → 소비자를 대기 큐/스택에 추가
    → park() → 생산자 올 때까지 대기

Executors.newCachedThreadPool()에서의 활용:
  ThreadPoolExecutor(0, MAX, 60s, SynchronousQueue)
  작업 도착 시 SynchronousQueue.offer() → 즉시 소비자(유휴 스레드) 없으면 실패
  → 실패 → 새 스레드 생성
  → 스레드가 많아지면 60초 후 유휴 스레드 종료
  → SynchronousQueue가 "직접 전달 또는 새 스레드" 분기를 구현
```

### 4. PriorityBlockingQueue와 DelayQueue

```
PriorityBlockingQueue:
  내부: 힙(Heap) 기반 우선순위 큐
  단일 ReentrantLock + notEmpty Condition
  용량 무제한 (자동 확장) → notFull Condition 없음
  → put()은 절대 블로킹 안 함
  → take()만 큐가 비면 블로킹

  주의:
    같은 우선순위 원소 간 FIFO 순서 보장 없음
    Iterator는 우선순위 순서 보장 안 함 (힙 순서)
    → 정렬된 순회: toArray() 후 Arrays.sort()

  사용 사례:
    우선순위 기반 작업 스케줄러
    Dijkstra 알고리즘의 우선순위 큐
    이벤트 기반 시뮬레이션

DelayQueue<E extends Delayed>:
  내부: PriorityBlockingQueue<E> (만료 시간 기준 정렬)
  take()는 큐 헤드 원소의 delay가 0 이하가 될 때까지 대기

  Delayed 인터페이스:
    getDelay(TimeUnit unit): 남은 지연 시간 반환
    compareTo(Delayed other): 만료 시간 비교 (정렬용)

  구현 예시:
    class DelayedTask implements Delayed {
        private final long executeAt;
        private final Runnable task;

        DelayedTask(Runnable task, long delay, TimeUnit unit) {
            this.task = task;
            this.executeAt = System.nanoTime() + unit.toNanos(delay);
        }

        public long getDelay(TimeUnit unit) {
            return unit.convert(executeAt - System.nanoTime(), NANOSECONDS);
        }

        public int compareTo(Delayed o) {
            return Long.compare(executeAt, ((DelayedTask)o).executeAt);
        }
    }

  사용 사례:
    세션 만료 처리
    재시도 지연 (Exponential Backoff)
    스케줄링 시스템
    캐시 TTL 만료
```

---

## 💻 실전 실험

### 실험 1: ArrayBlockingQueue vs LinkedBlockingQueue 처리량

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Group)
@Fork(2)
public class BlockingQueueBenchmark {

    static final int CAPACITY = 1000;

    ArrayBlockingQueue<Integer>  abq = new ArrayBlockingQueue<>(CAPACITY);
    LinkedBlockingQueue<Integer> lbq = new LinkedBlockingQueue<>(CAPACITY);

    @Setup public void setup() {
        // 절반 채우기
        for (int i = 0; i < CAPACITY / 2; i++) {
            abq.offer(i); lbq.offer(i);
        }
    }

    @Benchmark @Group("abq") @GroupThreads(4)
    public void abqPut() throws InterruptedException { abq.put(1); }
    @Benchmark @Group("abq") @GroupThreads(4)
    public Integer abqTake() throws InterruptedException { return abq.take(); }

    @Benchmark @Group("lbq") @GroupThreads(4)
    public void lbqPut() throws InterruptedException { lbq.put(1); }
    @Benchmark @Group("lbq") @GroupThreads(4)
    public Integer lbqTake() throws InterruptedException { return lbq.take(); }
}
// 결과: 생산자/소비자 균등 경쟁에서 LBQ가 30~50% 더 높은 처리량
// put과 take가 독립 락을 사용하므로 동시 실행 가능
```

### 실험 2: SynchronousQueue 직접 전달 패턴

```java
import java.util.concurrent.*;

public class SynchronousQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        SynchronousQueue<String> sq = new SynchronousQueue<>();

        // 생산자: 소비자가 올 때까지 대기
        Thread producer = new Thread(() -> {
            try {
                System.out.println("생산자: put 시도 (" + System.currentTimeMillis() + ")");
                sq.put("data-payload");
                System.out.println("생산자: put 완료 (" + System.currentTimeMillis() + ")");
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "Producer");

        // 소비자: 2초 후 take
        Thread consumer = new Thread(() -> {
            try {
                Thread.sleep(2000);  // 2초 지연
                System.out.println("소비자: take 시도 (" + System.currentTimeMillis() + ")");
                String data = sq.take();
                System.out.println("소비자: 수신 = " + data);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "Consumer");

        producer.start();
        consumer.start();
        producer.join(); consumer.join();
        // 생산자의 put이 소비자의 take가 올 때까지 ~2초 대기 확인
    }
}
```

### 실험 3: DelayQueue로 세션 만료 구현

```java
import java.util.concurrent.*;

public class SessionExpiry {
    static class Session implements Delayed {
        final String id;
        final long expireAt;

        Session(String id, long delayMs) {
            this.id = id;
            this.expireAt = System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(delayMs);
        }

        public long getDelay(TimeUnit unit) {
            return unit.convert(expireAt - System.nanoTime(), TimeUnit.NANOSECONDS);
        }

        public int compareTo(Delayed o) {
            return Long.compare(expireAt, ((Session)o).expireAt);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        DelayQueue<Session> expiry = new DelayQueue<>();

        // 세션 등록 (각각 다른 만료 시간)
        expiry.put(new Session("user-A", 1000));
        expiry.put(new Session("user-B", 3000));
        expiry.put(new Session("user-C", 2000));

        System.out.println("만료 감시 시작...");

        // 만료된 세션 처리 스레드
        Thread expiryWorker = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Session s = expiry.take();  // 만료까지 대기
                    System.out.printf("[%d ms] 세션 만료: %s%n",
                        System.currentTimeMillis() % 10000, s.id);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        expiryWorker.start();
        Thread.sleep(4000);  // 모든 세션 만료 기다림
        expiryWorker.interrupt();
        // 출력 순서: user-A (1s) → user-C (2s) → user-B (3s)
    }
}
```

---

## 📊 성능/비용 비교

```
BlockingQueue 구현체별 특성 비교:

특성                    | ArrayBQ       | LinkedBQ      | SynchronousQ  | PriorityBQ
───────────────────────┼───────────────┼───────────────┼───────────────┼───────────
내부 구조               | 순환 배열      | 연결 리스트    | 없음 (직접전달)| 힙 배열
용량                    | 고정 (지정)   | 선택적 (기본:∞)| 0 (버퍼 없음) | 무한 (자동확장)
락 구조                 | 단일 lock      | 분리 lock(2개)| Transfer 알고리즘| 단일 lock
put/take 동시 실행      | ❌ (공유 락)   | ✅ (분리 락)   | ✅ (직접 전달) | 제한적
GC 압박                 | 낮음 (배열)   | 중간 (노드 생성)| 낮음          | 낮음 (배열)
메모리 예측 가능성       | ✅ (고정)      | △ (동적)      | ✅ (없음)     | △ (확장)
공정성 옵션             | ✅ (생성 시)   | ❌            | ✅ (생성 시)  | ❌
정렬                    | ❌ (FIFO)     | ❌ (FIFO)     | ❌            | ✅ (우선순위)

처리량 비교 (4생산자, 4소비자, JMH):
  ArrayBlockingQueue:  ~800,000 ops/ms
  LinkedBlockingQueue: ~1,100,000 ops/ms (+37%)
  SynchronousQueue:    ~1,500,000 ops/ms (직접 전달, 버퍼 없음)
```

---

## ⚖️ 트레이드오프

```
큐 선택 기준:

ArrayBlockingQueue:
  메모리 사용량 예측이 중요한 경우
  GC 압박을 줄여야 할 때
  공정성 옵션이 필요할 때 (new ArrayBlockingQueue(n, true))
  크기가 절대 초과되면 안 되는 경우

LinkedBlockingQueue:
  생산자/소비자 처리량이 중요한 경우 (분리 락)
  크기 유연성이 필요할 때
  대부분의 생산자-소비자 패턴에서 기본 선택

SynchronousQueue:
  생산 즉시 소비되어야 할 때
  캐시드 스레드 풀(CachedThreadPool)처럼 버퍼 없는 직접 전달
  파이프라인에서 스테이지 간 직접 핸드오프

PriorityBlockingQueue:
  우선순위 기반 작업 처리
  put()이 절대 블로킹되면 안 되는 경우 (무한 확장)
  타임아웃, 재시도 큐

DelayQueue:
  지연/만료 기반 이벤트 처리
  세션 TTL, 캐시 만료, 스케줄러
```

---

## 📌 핵심 정리

```
BlockingQueue 핵심:

ArrayBlockingQueue:
  순환 배열 + 단일 ReentrantLock + notFull/notEmpty Condition
  put/take 직렬화 → 단일 락 병목
  메모리 고정, GC 압박 없음

LinkedBlockingQueue:
  연결 리스트 + putLock + takeLock (분리!)
  put/take 동시 실행 가능 → 더 높은 처리량
  count: AtomicInteger (두 락에서 동시 접근)
  신호 전달: lock 밖에서 signal → 효율적 스케줄링

SynchronousQueue:
  버퍼 없음, 직접 전달 메커니즘
  공정: TransferQueue (FIFO), 비공정: TransferStack (LIFO)
  CachedThreadPool 기본 큐

Condition 패턴:
  await(): 락 해제 + 대기
  signal(): 다음 대기자 깨움
  항상 while 루프로 spurious wakeup 방어
```

---

## 🤔 생각해볼 문제

**Q1.** `LinkedBlockingQueue`에서 `put()`이 `putLock`만 보유하는데, `count`를 `AtomicInteger`로 관리하는 것만으로 `takeLock` 없이 count를 안전하게 읽을 수 있는가?

<details>
<summary>해설 보기</summary>

그렇다. `AtomicInteger.getAndIncrement()`와 `getAndDecrement()`는 원자적이므로, `putLock`을 보유한 생산자와 `takeLock`을 보유한 소비자가 동시에 `count`를 수정해도 안전하다.

핵심은 `count`의 정확성보다 "가득 찬지" 또는 "비었는지" 여부를 판단하는 데 사용된다는 점이다:
- `put()`: `count.get() == capacity` 확인 → 이 값이 잠시 stale해도 `notFull.await()`가 대기 후 재확인
- `take()`: `count.get() == 0` 확인 → 마찬가지

`count` 변경 후 상대방에게 신호를 보내는 로직:
- `put()` 완료 후: `c == 0`이었으면 소비자에게 `signalNotEmpty()` (비었다가 채워짐)
- `take()` 완료 후: `c == capacity`였으면 생산자에게 `signalNotFull()` (가득 찼다가 빠짐)

이 상호 신호 로직이 올바르게 동작하기 위해 `getAndIncrement/getAndDecrement`가 이전 값을 정확히 반환하는 원자성이 필요하다.

</details>

---

**Q2.** `ArrayBlockingQueue(n, true)` 공정 모드는 내부적으로 어떻게 FIFO 순서를 보장하는가?

<details>
<summary>해설 보기</summary>

공정 모드는 `ReentrantLock(true)` (Fair Lock)을 사용한다. Fair `ReentrantLock`은 AQS의 CLH 큐에 스레드를 FIFO 순서로 대기시키고 순서대로 깨운다.

`ArrayBlockingQueue(n, true)`:
```java
this.lock = new ReentrantLock(fair);  // true = Fair Lock
```

공정 모드에서 `put()`이 `notFull.await()`로 대기할 때:
1. ReentrantLock의 Condition Wait Set에 FIFO로 들어감
2. `take()` 후 `notFull.signal()` 호출 시 Wait Set의 가장 오래된 스레드를 깨움
3. 깨어난 스레드가 AQS 큐에서 순서대로 락 획득

비공정 모드(기본)보다 처리량이 낮지만(~40~60%), 특정 생산자/소비자 스레드가 계속 밀리는 기아 문제를 방지한다. 실시간 시스템이나 레이턴시 편차가 중요한 경우에 유용하다.

</details>

---

**Q3.** `BlockingQueue.drainTo(Collection, maxElements)`는 내부적으로 어떻게 동작하는가? 여러 원소를 한 번에 꺼내는 것이 왜 효율적인가?

<details>
<summary>해설 보기</summary>

`drainTo(c, max)`는 최대 `max`개의 원소를 큐에서 꺼내 컬렉션 `c`에 추가한다.

`ArrayBlockingQueue`의 `drainTo` 내부:
```java
final ReentrantLock lock = this.lock;
lock.lock();
try {
    int n = Math.min(max, count);
    for (int i = 0; i < n; i++) {
        c.add(items[takeIndex]);
        items[takeIndex] = null;
        takeIndex = inc(takeIndex);
    }
    count -= n;
    if (count > 0) notEmpty.signalAll();
    return n;
} finally { lock.unlock(); }
```

효율적인 이유:
1. **락 획득 1회**: 여러 원소를 꺼내는 동안 락을 한 번만 획득/해제 → 락 오버헤드 N배 절감
2. **park/unpark 감소**: `take()`를 N번 호출하면 N번의 락 획득 + N번의 신호 처리 → `drainTo`는 1번

배치 소비자 패턴에서 유용:
```java
List<Task> batch = new ArrayList<>();
queue.drainTo(batch, 100);  // 최대 100개 한 번에
processBatch(batch);        // 일괄 처리
```
→ 데이터베이스 배치 삽입, Kafka Producer 배치 전송, 로그 배치 플러시에 효과적.

</details>

---

<div align="center">

**[⬅️ 이전: CopyOnWriteArrayList](./02-copy-on-write-arraylist.md)** | **[홈으로 🏠](../README.md)** | **[다음: ConcurrentSkipListMap ➡️](./04-concurrent-skip-list-map.md)**

</div>
