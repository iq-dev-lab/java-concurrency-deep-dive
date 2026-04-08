# 라이브락과 기아 — 공정한 락의 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 라이브락(Livelock)이 데드락과 어떻게 다른가?
- Non-Fair Lock에서 특정 스레드가 영원히 락을 못 얻는 기아(Starvation) 조건은?
- Fair Lock으로 기아를 해결하지만 처리량이 저하되는 트레이드오프는?
- `ThreadMXBean`으로 스레드 CPU 사용량을 모니터링해 기아를 탐지하는 방법은?
- 기아 없이 처리량도 유지하는 실용적 설계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

기아는 데드락처럼 시스템이 완전히 멈추지 않아서 탐지가 어렵다. 특정 API만 응답이 느리거나, 특정 사용자 요청만 타임아웃이 발생하는 패턴이 기아의 징조다. `ThreadMXBean`과 `async-profiler`로 이를 탐지하는 방법과 공정한 설계를 아는 것이 SLA를 지키는 핵심이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 라이브락을 데드락으로 착각
  CPU 100% → "데드락은 CPU 0%이니 다른 문제겠지"
  → 라이브락은 CPU를 씀 (스레드들이 계속 시도하지만 진행 없음)
  → Thread Dump에 BLOCKED 없음, 모두 RUNNABLE
  → "진행하는 것처럼 보이지만 실제로 아무것도 완료 안 됨"

실수 2: 기아를 성능 문제로만 인식
  "특정 요청이 가끔 느리네, 서버 스펙 부족인가?"
  → Non-Fair Lock + 높은 경쟁 → 특정 스레드 계속 CAS 실패
  → 해당 스레드가 처리하는 요청만 계속 지연
  → Fair Lock 또는 락 설계 개선으로 해결

실수 3: Fair Lock이 항상 기아를 완전 방지한다는 오해
  "ReentrantLock(true) 쓰면 기아 없겠지"
  → Fair Lock은 락 획득 순서를 보장하지만
  → 우선순위 역전(Priority Inversion)은 별도 문제
  → OS 스케줄러가 특정 스레드에 CPU 타임을 적게 주면 여전히 지연 가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 기아 방지 패턴

// 패턴 1: Fair Lock (FIFO 순서 보장)
ReentrantLock fairLock = new ReentrantLock(true);  // true = Fair
// FIFO 순서로 락 획득 → 기아 없음, 단 처리량 감소

// 패턴 2: 타임아웃 + 재시도 (기아 자가 회복)
boolean acquired = false;
while (!acquired) {
    acquired = lock.tryLock(100, TimeUnit.MILLISECONDS);
    if (!acquired) {
        log.warn("락 획득 실패, 재시도...");
        // 백오프 또는 대안 처리
    }
}

// 패턴 3: 락 없이 CAS (기아 가능하지만 빠름, 짧은 대기 용)
// LongAdder/AtomicInteger: 기아 가능하지만 실용적 문제 드묾

// 패턴 4: 작업 큐 + 단일 소비자 (기아 구조적 방지)
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();
// 생산자: queue.put(task)
// 소비자 1개: while (true) { process(queue.take()); }
// → FIFO 보장, 기아 없음, 스레드 안전
```

---

## 🔬 내부 동작 원리

### 1. 라이브락 발생 메커니즘

```
라이브락 시나리오:

두 스레드가 서로 상대방에게 양보:
  Thread A: lock1 시도 → 성공
             lock2 tryLock → 실패
             → lock1 해제 (양보)
             → 짧은 대기

  Thread B: lock2 시도 → 성공
             lock1 tryLock → 실패
             → lock2 해제 (양보)
             → 짧은 대기

  문제: A와 B가 동시에 대기 후 동시에 재시도
       계속 같은 패턴 반복 → 아무것도 진행 안 됨
       CPU는 계속 사용됨 (재시도 루프)

데드락 vs 라이브락 비교:
  데드락: 스레드 BLOCKED, CPU 0%, 아무것도 실행 안 됨
  라이브락: 스레드 RUNNABLE, CPU 높음, 많은 실행이지만 진행 없음

jstack 시그니처:
  데드락: "Found N Java-level deadlock"
  라이브락: 스레드 모두 RUNNABLE, 스택에 "tryLock", 
             "backoff", "retry" 같은 패턴, 그러나 진행 지표(카운터) 멈춤

라이브락 해결:
  랜덤 백오프: 두 스레드가 다른 시간에 재시도
  우선순위 부여: 한 스레드가 항상 먼저 시도
  중앙 조율자: 큐/채널을 통한 단방향 통신
```

### 2. 기아 발생 메커니즘과 Non-Fair Lock

```
Non-Fair Lock에서 기아:

ReentrantLock(false) 기본 동작:
  락 해제 시:
    큐에서 대기 중인 스레드 unpark
    AND 새로 들어온 스레드도 즉시 CAS 시도 가능 (Barging)

  결과:
    새로 도착한 스레드 T_new:
      unpark 대기 시간 없이 즉시 CAS → 높은 성공 확률
    큐에서 대기 중인 스레드 T_waiting:
      park에서 unpark → 스케줄러 대기 (~수 μs) → CAS 시도
      → T_new에게 계속 밀림

  극단적 경우:
    T_waiting이 수십 ms 또는 그 이상 대기
    고트래픽 환경에서 특정 스레드 처리가 수십 초 지연 가능
    SLA 위반

기아 탐지:
  ThreadMXBean으로 스레드별 CPU 사용 시간 모니터링
  특정 스레드의 CPU 사용이 0에 가까우면 기아 의심
  
Fair Lock으로 해결:
  ReentrantLock(true)
  큐 FIFO 순서 엄격 적용
  Barging 불가 → 모든 스레드 큐 통과 → 순서 보장
  단점: 처리량 ~40~60% 감소
```

### 3. ThreadMXBean으로 기아 탐지

```java
import java.lang.management.*;

public class StarvationDetector {
    private final ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
    private final Map<Long, Long> lastCpuTime = new ConcurrentHashMap<>();

    public void detectStarvation() {
        // CPU 시간 측정 활성화
        if (!mxBean.isThreadCpuTimeEnabled()) {
            mxBean.setThreadCpuTimeEnabled(true);
        }

        ThreadInfo[] threads = mxBean.dumpAllThreads(false, false);
        for (ThreadInfo info : threads) {
            long threadId = info.getThreadId();
            long cpuNs = mxBean.getThreadCpuTime(threadId);

            Long prev = lastCpuTime.put(threadId, cpuNs);
            if (prev != null) {
                long delta = cpuNs - prev;  // 지난 측정 이후 CPU 사용 (ns)

                // 10초간 CPU 시간이 1ms 미만이고 RUNNABLE이면 기아 의심
                if (delta < 1_000_000 && info.getThreadState() == Thread.State.BLOCKED) {
                    log.warn("기아 의심: {} (BLOCKED, CPU={}ns in 10s)",
                        info.getThreadName(), delta);
                }
            }
        }
    }

    // 주기적으로 실행
    public void startMonitoring() {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
        scheduler.scheduleAtFixedRate(this::detectStarvation, 10, 10, TimeUnit.SECONDS);
    }
}
```

---

## 💻 실전 실험

### 실험 1: Non-Fair vs Fair Lock 기아 비교

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.*;

public class StarvationDemo {
    static final int THREADS = 8;
    static final int ITERATIONS = 1_000_000;

    public static void main(String[] args) throws InterruptedException {
        // Non-Fair Lock 실험
        runExperiment(new ReentrantLock(false), "Non-Fair");
        // Fair Lock 실험
        runExperiment(new ReentrantLock(true), "Fair");
    }

    static void runExperiment(ReentrantLock lock, String mode) throws InterruptedException {
        AtomicInteger[] counts = new AtomicInteger[THREADS];
        for (int i = 0; i < THREADS; i++) counts[i] = new AtomicInteger(0);

        Thread[] threads = new Thread[THREADS];
        for (int t = 0; t < THREADS; t++) {
            final int tid = t;
            threads[t] = new Thread(() -> {
                for (int i = 0; i < ITERATIONS; i++) {
                    lock.lock();
                    try { counts[tid].incrementAndGet(); }
                    finally { lock.unlock(); }
                }
            }, "Worker-" + t);
        }

        long start = System.nanoTime();
        for (Thread th : threads) th.start();
        for (Thread th : threads) th.join();

        System.out.printf("[%s] %.2f ms%n", mode, (System.nanoTime()-start)/1e6);
        System.out.print("획득 횟수: ");
        int min = Integer.MAX_VALUE, max = 0;
        for (AtomicInteger c : counts) {
            System.out.print(c.get() + " ");
            min = Math.min(min, c.get());
            max = Math.max(max, c.get());
        }
        System.out.printf("%n최소=%d, 최대=%d, 편차=%.2f%%%n%n",
            min, max, (max-min)*100.0/ITERATIONS);
        // Non-Fair: 편차 클 수 있음 (기아)
        // Fair: 편차 거의 없음 (공정)
    }
}
```

---

## 📊 성능/비용 비교

```
Non-Fair vs Fair Lock 비교 (8스레드 JMH):

지표                | Non-Fair    | Fair       | 비고
───────────────────┼────────────┼────────────┼────────────────────
처리량 (ops/ms)     | 3,000,000  | 600,000    | Non-Fair가 ~5배 빠름
지연시간 분포        | 편차 큼     | 균일        | Fair가 p99 낮음
기아 가능성          | 있음       | 없음       | Fair가 안전
컨텍스트 스위칭      | 적음       | 많음       | park/unpark 더 빈번
```

---

## ⚖️ 트레이드오프

```
Non-Fair vs Fair 선택:

Non-Fair (기본):
  처리량 최대화 (SLA = 전체 처리량 기준)
  특정 요청 지연 허용 가능
  고빈도 짧은 임계 구역

Fair:
  모든 요청 동등 처리 (SLA = p99/p999)
  실시간성 요구 서비스
  지연 편차 최소화 중요

실용적 절충:
  tryLock(timeout): Non-Fair이지만 최대 대기 시간 보장
  작업 큐 (BlockingQueue): 구조적으로 기아 없음
  읽기-쓰기 분리 (ReadWriteLock): 읽기 병렬화로 기아 감소
```

---

## 📌 핵심 정리

```
라이브락 / 기아 핵심:

라이브락:
  CPU 사용 + 진행 없음 (모두 RUNNABLE)
  원인: 상호 양보, 동기 재시도
  해결: 랜덤 백오프, 우선순위 부여

기아:
  특정 스레드가 락을 계속 못 얻음
  Non-Fair Lock + 고경쟁 = 기아 가능
  해결: Fair Lock, 타임아웃 tryLock, 큐 기반 설계

탐지:
  ThreadMXBean.getThreadCpuTime() 모니터링
  Thread Dump에서 BLOCKED 스레드와 보유자 분석
  async-profiler flame graph

트레이드오프:
  Fair Lock: 기아 없음 + 처리량 ~5배 감소
  Non-Fair: 높은 처리량 + 기아 가능
  → SLA 요구사항에 따라 선택
```

---

## 🤔 생각해볼 문제

**Q1.** `PriorityBlockingQueue`를 사용하면 낮은 우선순위 작업이 영원히 처리되지 않는 기아가 발생할 수 있는가?

<details>
<summary>해설 보기</summary>

이론적으로 가능하다. 높은 우선순위 작업이 계속 추가되면 낮은 우선순위 작업은 영원히 `take()`를 기다린다. 이를 **우선순위 기아(Priority Starvation)**라 한다.

해결책:
1. **에이징(Aging)**: 대기 시간에 비례해 우선순위를 높임. JDK 표준 지원은 없어 직접 구현 필요.
2. **다중 큐**: 우선순위 레벨별 별도 큐 + 각 큐에 비율 할당 (예: HIGH 60%, MED 30%, LOW 10%)
3. **타임아웃**: 일정 시간 내 처리 안 되면 우선순위 상향
4. **Phased Queue**: 일정 주기로 낮은 우선순위 작업을 강제 처리

실무에서는 SLA가 있는 요청에 별도 큐를 두고, 백그라운드 작업에만 낮은 우선순위를 부여하는 것이 일반적이다.

</details>

---

**Q2.** CPU 바운드 작업에서 라이브락이 발생했을 때 CPU 사용률이 100%라면, I/O 바운드 작업에서 라이브락은 CPU 사용률이 어떻게 되는가?

<details>
<summary>해설 보기</summary>

I/O 바운드 작업에서 라이브락은 CPU 사용률이 낮을 수 있다. 스레드들이 락을 시도→실패→I/O 대기→재시도를 반복하면, I/O 대기 동안 CPU를 사용하지 않기 때문이다.

증상:
- CPU: 낮음 (20~40%)
- I/O: 높음 (많은 재시도로 I/O 폭발)
- 처리량: 거의 0 (라이브락)
- 스레드 상태: WAITING 또는 TIMED_WAITING (I/O 대기 중)

이 경우 Thread Dump만으로 진단하기 어렵다. 처리량 메트릭(초당 완료된 작업 수)이 0에 가까운데 CPU와 I/O가 활발하면 라이브락을 의심한다.

JFR의 이벤트 타임라인을 보면 같은 스레드들이 반복적으로 락 획득 시도→실패 패턴을 보인다.

</details>

---

**Q3.** Fair `ReentrantLock`이 FIFO 순서를 보장한다면, `CountDownLatch`와 같이 모든 스레드가 동시에 깨어나는 경우에도 FIFO가 유지되는가?

<details>
<summary>해설 보기</summary>

아니다. `CountDownLatch.countDown()`이 `await()` 중인 모든 스레드를 동시에 깨울 때, 깨어난 순서는 보장되지 않는다. AQS의 `propagate` 모드로 공유 락이 한꺼번에 해제되어 모든 대기 스레드가 동시에 RUNNABLE이 된다.

이후 Fair `ReentrantLock`을 획득하려 하면 그때부터 FIFO가 적용된다. 즉:
- `countDown()` 후 모든 스레드 깨어남 (순서 없음)
- 깨어난 스레드들이 Fair Lock 획득 시도 → 도착 순서(큐 등록 순서)로 처리

Fair Lock의 FIFO는 **락을 기다리기 시작한 시점의 순서**를 보장한다. 여러 스레드가 동시에 깨어나 동시에 락을 시도하면, 그 중 AQS 큐에 먼저 등록된 순서로 처리된다.

</details>

---

<div align="center">

**[⬅️ 이전: 데드락 분석](./01-deadlock-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: Lock Contention 진단 ➡️](./03-lock-contention-profiling.md)**

</div>
