# 스레드 생성과 Thread Pool — ThreadPoolExecutor 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ThreadPoolExecutor`는 새 작업이 들어올 때 어떤 순서로 스레드와 큐를 사용하는가?
- corePoolSize와 maximumPoolSize는 언제 각각 의미를 갖는가?
- `CallerRunsPolicy`가 백프레셔를 구현하는 방식은 무엇인가?
- `ForkJoinPool`과 `ThreadPoolExecutor`의 설계 목표가 어떻게 다른가?
- Executors 팩토리 메서드가 왜 실무에서 위험한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring `@Async`, Tomcat 요청 처리, `CompletableFuture.supplyAsync()` — 이들은 모두 Thread Pool 위에서 동작한다. Thread Pool이 포화됐을 때 어떻게 동작하는지 모르면, 장애 시 `RejectedExecutionException`이 왜 터지는지, 요청이 왜 큐에서 무한정 대기하는지 원인을 찾을 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Executors 팩토리 메서드를 신뢰
  Executors.newFixedThreadPool(10)
  → LinkedBlockingQueue의 용량이 Integer.MAX_VALUE (무제한 큐)
  → 큐에 작업이 쌓여도 거절되지 않음
  → OOM 발생 전까지 아무 경고 없음

실수 2: corePoolSize와 maximumPoolSize 혼동
  corePoolSize=10, maximumPoolSize=100으로 설정했는데
  → 큐가 차지 않으면 maximumPoolSize까지 늘어나지 않음
  → "왜 스레드가 10개밖에 없지?" 라고 혼란

실수 3: CallerRunsPolicy를 이해하지 못하고 사용
  → 큐 포화 시 HTTP 요청 스레드가 직접 작업 실행
  → Tomcat 스레드가 묶여 다른 요청을 받지 못함
  → 전체 서버 응답 불가

실수 4: keepAliveTime을 너무 짧게 설정
  keepAliveTime=0 → 작업 완료 즉시 스레드 종료
  다음 요청 시 다시 스레드 생성 → 생성 비용 반복
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// ThreadPoolExecutor를 직접 설정하는 올바른 패턴
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                                 // corePoolSize
    50,                                 // maximumPoolSize
    60L, TimeUnit.SECONDS,              // keepAliveTime (idle 스레드 유지 시간)
    new ArrayBlockingQueue<>(200),      // 유한 큐 (용량 명시)
    new ThreadFactory() {               // 스레드 이름 지정 (디버깅 용이)
        private final AtomicInteger count = new AtomicInteger();
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "worker-" + count.incrementAndGet());
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()  // 거절 정책 명시
);

// 모니터링 (운영 필수)
System.out.println("활성 스레드: " + executor.getActiveCount());
System.out.println("큐 크기: " + executor.getQueue().size());
System.out.println("완료 작업: " + executor.getCompletedTaskCount());
System.out.println("풀 크기: " + executor.getPoolSize());
```

---

## 🔬 내부 동작 원리

### 1. 작업 제출 시 ThreadPoolExecutor의 의사결정 흐름

```
새 작업(Runnable)이 execute() 또는 submit()으로 제출될 때:

  ┌──────────────────────────────────────────────────────────┐
  │              ThreadPoolExecutor.execute(task)            │
  └───────────────────────────────┬──────────────────────────┘
                                  │
              현재 스레드 수 < corePoolSize?
               ┌──────YES──────┐  ┌──NO──────────────────────┐
               ▼               │  ▼                          │
        새 스레드 생성           │  큐(workQueue)에 넣기 시도   │
        작업 즉시 실행           │  ┌────성공────┐ ┌──실패──┐  │
        (Core Thread)          │  │            │ │        │  │
                               │  │ 큐에 저장   │ │현재 스레드│ │
                               │  │ 대기        │ │< maxPool?│ │
                               │  └────────────┘ └─┬────┬──┘  │
                               │                   │    │     │
                               │                  YES   NO    │
                               │                   │    │     │
                               │              새 스레드 │     │
                               │              생성하여  │  RejectedExecution
                               │              작업 실행 │  Handler 호출
                               │              (Extra   │
                               │               Thread) │  4가지 정책:
                               │                       │  AbortPolicy (기본)
                               │                       │  CallerRunsPolicy
                               │                       │  DiscardPolicy
                               │                       │  DiscardOldestPolicy
                               └───────────────────────┘

핵심 순서 (자주 오해하는 부분):
  1순위: corePoolSize 미만이면 → 즉시 새 스레드 생성 (큐 먼저 아님!)
  2순위: corePoolSize 이상이면 → 큐에 넣기 시도
  3순위: 큐 가득 차면 → maximumPoolSize까지 새 스레드 생성
  4순위: maximumPoolSize도 가득 차면 → RejectedExecutionHandler
```

### 2. Core/Max 스레드와 큐의 상호작용

```
시나리오: corePoolSize=5, maximumPoolSize=20, 큐=ArrayBlockingQueue(10)

상태 변화:

작업 1~5 제출:
  스레드 1~5 생성 (Core Thread)
  각 작업 즉시 처리 시작

작업 6~15 제출 (코어 스레드 5개 모두 바쁨):
  큐에 10개 적재 (큐 용량 10개 꽉 참)

작업 16~20 제출 (큐도 꽉 참):
  스레드 6~10 생성 (Extra Thread)
  작업 16~20 즉시 처리

작업 21 제출 (스레드 20개, 큐 10개 모두 꽉 참):
  → maximumPoolSize(20) 초과 불가
  → RejectedExecutionHandler 실행

  [큐에서 꺼낼 때]:
    어느 작업부터 처리? → 작업 6~15 (큐의 앞쪽부터)
    Extra Thread(스레드 6~10)는 큐에서 작업을 꺼내 처리
    keepAliveTime 후 idle이면 → Extra Thread 종료
    Core Thread는 keepAlive 설정에 따라 유지 또는 종료

직관과 다른 부분:
  maximumPoolSize는 "큐가 찰 때만" 의미 있음
  큐가 무한대(LinkedBlockingQueue 기본)이면:
    → 큐가 절대 안 참 → maximumPoolSize까지 절대 안 늘어남
    → corePoolSize 개수만 계속 실행
```

### 3. RejectedExecutionHandler 4가지 정책 비교

```
AbortPolicy (기본):
  throw new RejectedExecutionException()
  → 제출한 스레드에서 예외 발생
  → 작업 유실 가능 (예외 처리 안 하면)
  → 언제 사용: 작업 유실이 절대 안 되는 경우 (직접 처리)

CallerRunsPolicy:
  제출한 스레드가 직접 run() 실행
  → Pool이 포화되면 제출 속도 자체를 늦춤 (자연스러운 백프레셔)
  → 제출 스레드가 작업 완료될 때까지 블로킹
  → 언제 사용: 처리 속도 > 생산 속도로 자연 조절이 필요할 때
  주의: Tomcat 요청 스레드가 제출 스레드면 → 해당 스레드 묶임

  백프레셔 동작 예시:
    [Producer Thread] → execute(task) → Pool 포화 → 직접 실행
    Producer가 직접 실행 중 → 새 작업 생산 불가 → 생산 속도 감소
    Pool이 여유 생기면 → 다시 Pool에 제출 → 생산 속도 회복

DiscardPolicy:
  아무것도 하지 않음 (작업 조용히 유실)
  → 언제 사용: 오래된 데이터는 무의미한 경우 (실시간 센서 데이터 등)
  → 주의: 작업 유실을 모름 → 디버깅 어려움

DiscardOldestPolicy:
  큐의 가장 오래된 작업을 버리고 새 작업 삽입
  → 최신 작업 우선 처리
  → 언제 사용: 최신성이 중요한 경우 (최근 상태 업데이트 등)
```

### 4. ForkJoinPool — 다른 설계 철학

```
ThreadPoolExecutor:
  단일 공유 큐 (BlockingQueue)
  작업 제출 → 큐 → 스레드가 꺼내 처리
  적합: I/O 바운드, 독립적 작업, 작업 크기 균일한 경우

  ┌──────────────────────────────────────────────┐
  │              공유 작업 큐                     │
  │  [task1][task2][task3][task4][task5]...      │
  └──────┬───────────────────────────────────────┘
         │ 스레드들이 경쟁하며 꺼냄
  Thread1 Thread2 Thread3 Thread4 Thread5

ForkJoinPool:
  스레드마다 개별 Deque (Work-Stealing Queue)
  큰 작업을 작은 작업으로 분할(Fork) 후 병합(Join)
  자신의 큐가 비면 다른 스레드의 큐에서 훔쳐옴 (Work Stealing)
  적합: CPU 바운드, 분할 정복 알고리즘, 재귀적 병렬 처리

  Thread1: [subtask1][subtask2][subtask3]  ← 자신 큐
  Thread2: []  ← 빈 큐 → Thread1 큐에서 훔침
  Thread3: [subtask4][subtask5]
  Thread4: []  ← Thread3 큐에서 훔침

Java 21 Virtual Thread의 ForkJoinPool:
  commonPool 기반이 아닌 전용 ForkJoinPool
  CPU 코어 수만큼의 캐리어 스레드
  Work Stealing으로 캐리어 스레드를 항상 바쁘게 유지

언제 어느 것을 쓰는가:
  CPU 바운드 분할 정복 → ForkJoinPool, RecursiveTask
  I/O 바운드, 독립 작업 → ThreadPoolExecutor
  Virtual Thread → 별도 고려 (Chapter 6)
  CompletableFuture.supplyAsync() → ForkJoinPool.commonPool() (기본)
    → 커스텀 Executor 지정 권장
```

### 5. ThreadPoolExecutor 상태 전이

```
ThreadPoolExecutor의 내부 상태:

  RUNNING    → 정상 동작, 새 작업 수락
     │
     │ shutdown()
     ▼
  SHUTDOWN   → 새 작업 거절, 큐/실행 중인 작업은 완료 대기
     │
     │ 큐 비고 + 모든 스레드 종료
     ▼
  TIDYING    → 모든 작업 완료, terminated() 훅 실행 예정
     │
     │ terminated() 완료
     ▼
  TERMINATED → 완전 종료

  RUNNING → shutdownNow() → STOP
  STOP: 실행 중인 작업 인터럽트, 큐 비움

Graceful Shutdown 패턴:
  executor.shutdown();                  // 새 작업 거절, 기존 완료 대기
  if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
      executor.shutdownNow();           // 30초 후 강제 종료
      if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
          System.err.println("Pool이 종료되지 않음");
      }
  }
```

---

## 💻 실전 실험

### 실험 1: ThreadPoolExecutor 동작 단계별 관찰

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadPoolInspector {
    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor pool = new ThreadPoolExecutor(
            3,                              // corePoolSize
            6,                              // maximumPoolSize
            10L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(4),    // 큐 용량 4
            r -> new Thread(r, "worker-" + new AtomicInteger().incrementAndGet()),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );

        // 작업 제출하며 풀 상태 출력
        for (int i = 1; i <= 15; i++) {
            final int taskId = i;
            try {
                pool.execute(() -> {
                    System.out.printf("[Task %d] 실행 스레드: %s%n",
                        taskId, Thread.currentThread().getName());
                    try { Thread.sleep(2000); } catch (InterruptedException e) {}
                });
            } catch (RejectedExecutionException e) {
                System.out.printf("[Task %d] 거절됨!%n", taskId);
            }

            System.out.printf("제출 후: 풀크기=%d, 활성=%d, 큐=%d%n",
                pool.getPoolSize(),
                pool.getActiveCount(),
                pool.getQueue().size());
            Thread.sleep(100);
        }

        pool.shutdown();
        pool.awaitTermination(30, TimeUnit.SECONDS);
    }
}
```

### 실험 2: CallerRunsPolicy 백프레셔 효과 측정

```java
public class CallerRunsBackpressureTest {
    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor pool = new ThreadPoolExecutor(
            2, 2, 0L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(2),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.CallerRunsPolicy()  // 백프레셔
        );

        long startTime = System.currentTimeMillis();
        int submitted = 0;

        // 빠르게 작업 제출 시도
        for (int i = 0; i < 20; i++) {
            long submitStart = System.nanoTime();
            pool.execute(() -> {
                try { Thread.sleep(500); } catch (InterruptedException e) {}
            });
            long submitEnd = System.nanoTime();
            submitted++;

            System.out.printf("작업 %d 제출 완료: %.2f ms (실행 스레드: %s)%n",
                submitted,
                (submitEnd - submitStart) / 1_000_000.0,
                Thread.currentThread().getName());
        }
        // 칼러런스 정책 시 pool 포화 구간에서 제출 스레드(main)가 직접 실행
        // → 제출 시간이 ~500ms로 늘어나는 것을 관찰

        pool.shutdown();
    }
}
```

### 실험 3: Executors 팩토리 메서드의 위험성 확인

```java
public class ExecutorsRisk {
    public static void main(String[] args) throws InterruptedException {
        // 위험: 무한 큐 (OOM 유발 가능)
        ExecutorService fixed = Executors.newFixedThreadPool(2);
        // 내부: new ThreadPoolExecutor(2, 2, 0L, ..., new LinkedBlockingQueue<>())
        // LinkedBlockingQueue 기본 용량 = Integer.MAX_VALUE = 21억

        // 빠르게 작업 제출
        for (int i = 0; i < 1_000_000; i++) {
            fixed.submit(() -> {
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            });
        }
        // 2개 스레드가 처리하는 동안 큐에 999,998개가 쌓임
        // 각 Runnable 객체 ~수십 바이트 → 수십 MB 메모리 소비
        // 작업량이 많으면 OOM

        ThreadPoolExecutor tpe = (ThreadPoolExecutor) fixed;
        System.out.println("큐 크기: " + tpe.getQueue().size()); // 거의 1,000,000
        fixed.shutdownNow();
    }
}
```

---

## 📊 성능/비용 비교

```
Executors 팩토리 메서드별 내부 구성:

팩토리 메서드                    | core | max           | 큐                    | 위험
───────────────────────────────┼──────┼───────────────┼────────────────────────┼──────────
newFixedThreadPool(n)           | n    | n             | LinkedBlockingQueue(∞) | OOM (큐 무한)
newSingleThreadExecutor()       | 1    | 1             | LinkedBlockingQueue(∞) | OOM (큐 무한)
newCachedThreadPool()           | 0    | Integer.MAX   | SynchronousQueue(0)   | 스레드 폭발
newScheduledThreadPool(n)       | n    | Integer.MAX   | DelayedWorkQueue      | 스레드 폭발

RejectedExecutionHandler 정책별 특성:

정책               | 예외 발생 | 작업 유실 | 생산 속도 억제 | 사용 권장
──────────────────┼──────────┼──────────┼──────────────┼──────────────────
AbortPolicy        | ✅        | 가능      | ❌            | 예외 처리 필수
CallerRunsPolicy   | ❌        | ❌        | ✅            | 백프레셔 필요 시
DiscardPolicy      | ❌        | ✅        | ❌            | 최신성이 중요할 때
DiscardOldestPolicy| ❌        | ✅(오래된)| ❌            | 최근 작업 우선 시
```

---

## ⚖️ 트레이드오프

```
Thread Pool 크기 설계 트레이드오프:

corePoolSize 크게:
  장점: 요청 급증 시 즉시 대응 (큐에 가지 않음)
  단점: idle 스레드가 많아 메모리 낭비

maximumPoolSize 크게:
  장점: 트래픽 스파이크 대응 가능
  단점: 많은 스레드 생성으로 컨텍스트 스위칭 증가

큐 용량 크게:
  장점: 일시적 트래픽 스파이크 흡수
  단점: 응답 지연 누적 (큐에서 오래 대기), OOM 위험
  → 큐가 크면 maximumPoolSize가 의미 없음 (큐가 안 참으므로)

큐 용량 작게 (또는 SynchronousQueue):
  장점: maximumPoolSize까지 스레드 빠르게 생성, 지연 최소화
  단점: 트래픽 스파이크에 취약 (거절 발생)

실무 권장:
  LinkedBlockingQueue 대신 ArrayBlockingQueue (유한 용량 명시)
  거절 정책 명시적 설정 (기본 AbortPolicy 주의)
  스레드 이름 지정 (Thread Dump 시 파악 용이)
  모니터링 지표 노출 (Micrometer + Actuator)
```

---

## 📌 핵심 정리

```
ThreadPoolExecutor 핵심:

작업 처리 순서:
  1순위: corePoolSize 미만 → 즉시 새 스레드
  2순위: 큐에 적재
  3순위: 큐 포화 → maximumPoolSize까지 스레드 추가
  4순위: max도 포화 → RejectedExecutionHandler

흔한 함정:
  Executors.newFixedThreadPool() → 무한 큐 (OOM 위험)
  Executors.newCachedThreadPool() → 무한 스레드 (OOM 위험)
  큐가 크면 maximumPoolSize까지 절대 안 늘어남

올바른 설정:
  ArrayBlockingQueue + 명시적 용량
  거절 정책 명시
  스레드 이름 지정
  keepAliveTime 적절히 설정

모니터링:
  getActiveCount()     # 실행 중인 스레드 수
  getQueue().size()    # 대기 중인 작업 수
  getCompletedTaskCount()  # 완료된 작업 총계
  → 큐 크기가 계속 증가 중이면 처리 속도 < 생산 속도 신호
```

---

## 🤔 생각해볼 문제

**Q1.** `corePoolSize=10, maximumPoolSize=10`으로 설정하고 `allowCoreThreadTimeOut(true)`를 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`allowCoreThreadTimeOut(true)`를 설정하면 Core Thread도 `keepAliveTime` 동안 idle 상태이면 종료된다. 기본적으로 Core Thread는 idle이어도 Pool에 유지되지만(Pool이 축소되지 않음), 이 설정을 켜면 Core Thread도 수명이 생긴다.

`corePoolSize=maximumPoolSize=10`이면 Fixed Thread Pool과 같은 구성이다. 여기에 `allowCoreThreadTimeOut(true)`를 추가하면 작업이 없을 때 스레드가 0개까지 줄어든다. 다음 작업이 들어오면 다시 스레드가 생성된다.

리소스를 절약하고 싶지만 평소에는 스레드를 유지하고 싶을 때 `keepAliveTime`을 길게 설정하는 것이 균형점이다. 서버리스 환경이나 배치 작업에서 유용하다.

</details>

---

**Q2.** `execute()`와 `submit()`의 차이는 무엇인가? 예외 처리 관점에서 어느 쪽이 더 안전한가?

<details>
<summary>해설 보기</summary>

`execute(Runnable)`는 반환값이 없고 작업 내부에서 발생한 `RuntimeException`이 `UncaughtExceptionHandler`로 전파된다. 예외가 처리되지 않으면 해당 스레드가 종료되고 Pool이 새 스레드를 생성한다.

`submit(Callable/Runnable)`은 `Future<T>`를 반환한다. 작업 내부 예외는 `Future.get()` 호출 시 `ExecutionException`으로 래핑되어 나온다. `get()`을 호출하지 않으면 예외가 **조용히 삼켜진다**.

```java
// 위험: 예외가 사라짐
executor.submit(() -> {
    throw new RuntimeException("이 예외는 어디로?");
});
// Future를 get()하지 않으면 예외 출력조차 안 됨

// 올바른 방법
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("예외");
});
try {
    future.get();
} catch (ExecutionException e) {
    System.err.println("작업 실패: " + e.getCause());
}
```

실무에서는 `submit()` 후 반환된 `Future`를 반드시 처리하거나, `CompletableFuture.exceptionally()`를 활용해 예외를 명시적으로 처리해야 한다.

</details>

---

**Q3.** Spring `@Async`의 기본 설정은 어떤 Thread Pool을 사용하는가? 왜 기본 설정을 그대로 운영에 사용하면 위험한가?

<details>
<summary>해설 보기</summary>

Spring `@Async`는 `AsyncAnnotationBeanPostProcessor`가 처리하며, 별도 `TaskExecutor` 빈이 없으면 `SimpleAsyncTaskExecutor`를 사용한다. 이는 **Thread Pool이 아니며**, 요청마다 새 Thread를 생성한다. Thread를 재사용하지 않으므로 대량 요청 시 스레드 폭발로 OOM이 발생한다.

올바른 설정:
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

`ThreadPoolTaskExecutor`는 `ThreadPoolExecutor`의 Spring 래퍼로, 위에서 배운 모든 설정을 동일하게 적용할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: 컨텍스트 스위칭 비용](./02-context-switching-cost.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스레드 상태 머신 ➡️](./04-thread-state-machine.md)**

</div>
