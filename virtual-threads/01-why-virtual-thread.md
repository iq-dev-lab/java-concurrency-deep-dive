# 왜 Virtual Thread인가 — OS 스레드의 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- OS 스레드 1:1 모델의 실제 비용은 얼마인가?
- I/O 바운드 작업에서 스레드가 대부분의 시간을 어떻게 낭비하는가?
- "Thread per Request" 모델의 처리량 상한선은 어디서 결정되는가?
- Virtual Thread는 이 문제를 어떻게 해결하는가?
- 기존 코드를 거의 바꾸지 않고 Virtual Thread를 도입할 수 있는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 21의 Virtual Thread(Project Loom)는 Java 동시성 모델의 패러다임 전환이다. "스레드 풀 크기를 얼마로 설정해야 하는가"라는 수십 년간의 질문에, "스레드를 수백만 개 만들면 된다"는 새로운 답을 제시한다. 하지만 이 전환이 어떻게 가능한지 이해하지 못하면, Pinning 같은 새로운 함정에 빠지거나 Virtual Thread의 이점을 활용하지 못한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 스레드 풀 크기를 크게 늘리면 해결된다는 오해
  # Tomcat 스레드 풀 설정
  server.tomcat.threads.max=500
  → 500개 스레드 생성: 500MB 메모리 (1MB/스레드 스택)
  → 500개 동시 요청 처리 가능
  → 501번째 요청: 대기
  → 문제: 500개 스레드가 I/O 대기 중 CPU를 쓰지 않고 있음
           실제 CPU 활용률은 5~10%에 불과

실수 2: Reactive(WebFlux)로 전환하면 완전히 해결된다는 가정
  "Reactive는 Non-Blocking이라 스레드가 적어도 된다"
  → 맞지만: 기존 코드 전면 재작성 (Mono/Flux API)
             디버깅 어려움 (스택 트레이스 이해 불가)
             Thread-per-Request 모델이 가진 직관성 포기
  → Virtual Thread: 기존 동기 코드 그대로, 스레드 낭비 제거

실수 3: Virtual Thread = 그냥 가벼운 OS 스레드라는 오해
  "Virtual Thread는 그냥 경량 스레드겠지"
  → Virtual Thread는 OS 스레드가 아님
  → JVM이 관리하는 Continuation 기반 스케줄링 단위
  → 블로킹 I/O 시 OS 스레드(캐리어)를 블로킹하지 않음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Virtual Thread 도입 전 (Thread-per-Request, 스레드 풀 방식)
ExecutorService pool = Executors.newFixedThreadPool(200);
pool.submit(() -> {
    // HTTP 요청 처리
    String data = httpClient.get("https://api.example.com");  // I/O 대기 중 스레드 차지
    return processData(data);
});
// 문제: 200개 스레드가 I/O 대기 중 → 200개 이상 동시 처리 불가

// Virtual Thread 도입 후
ExecutorService vExecutor = Executors.newVirtualThreadPerTaskExecutor();
vExecutor.submit(() -> {
    // 동일한 동기 코드 그대로!
    String data = httpClient.get("https://api.example.com");  // I/O 대기 시 캐리어 해제
    return processData(data);
});
// 장점: 100만 개 요청도 100만 개 Virtual Thread로 처리 가능
//       실제 OS 스레드(캐리어)는 CPU 코어 수만큼만 사용
//       코드 변경 없음!

// Java 21 Simple Factory
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("I am Virtual Thread: " + Thread.currentThread().isVirtual());
});
vt.join();
```

---

## 🔬 내부 동작 원리

### 1. OS 스레드 1:1 모델의 실제 비용

```
OS 스레드 구조 (Linux, JVM 기준):

각 Java 스레드 = OS 스레드 1개:
  커널 스택: ~8KB (고정, 시스템 콜 처리용)
  JVM 스택:  ~512KB~1MB (기본값, -Xss로 조정)
  Thread 객체: ~수 KB (Java 힙)
  OS 자료구조: task_struct 등 ~수 십 KB

총 스레드당 비용:
  ~1MB 메모리 (기본 설정)
  8,000개 스레드 → ~8GB RAM (불가)
  500개 스레드 → ~500MB RAM (일반적인 상한)

컨텍스트 스위칭 비용:
  레지스터 저장/복원: ~20~100ns
  TLB flush (필요 시): ~수 백 ns
  스케줄러 결정: ~수 μs
  캐시 콜드: 수 백 μs

현실:
  4코어 CPU, 200개 스레드
  동시 실행: 4개만 가능 (CPU 코어 수)
  나머지 196개: 대기 (ready queue 또는 I/O 대기)
  1초에 수천 번의 컨텍스트 스위칭 발생
```

### 2. I/O 바운드 작업에서의 비효율

```
전형적인 웹 서버 요청 처리 타임라인:

요청 수신
  │
  ├── DB 쿼리 (10ms) ←── 스레드 WAITING (10ms 낭비!)
  │      ↓
  ├── 외부 API 호출 (50ms) ←── 스레드 WAITING (50ms 낭비!)
  │      ↓
  ├── 파일 읽기 (5ms) ←── 스레드 WAITING (5ms 낭비!)
  │      ↓
  └── 응답 생성 (1ms) ←── 실제 CPU 사용
         ↓
      응답 반환

총 처리 시간: 66ms
실제 CPU 사용: 1ms (1.5%)
스레드 낭비: 65ms (98.5%)

OS 스레드 기반 계산:
  100ms/요청 처리, 그 중 95ms가 I/O 대기
  스레드 1개로 초당 10 req/s만 처리 가능
  1,000 req/s 달성하려면: 100개 스레드 필요
  10,000 req/s: 1,000개 스레드 → 1GB RAM (불가)

이것이 C10K 문제의 본질:
  10,000개 동시 연결 = 10,000개 스레드 = 불가능

기존 해결책의 한계:
  스레드 풀: 처리량 상한선 고정 (풀 크기에 비례)
  Reactive: 코드 복잡성 폭증, 학습 곡선 급격
  Async/Await (코루틴): 코드 전면 수정 필요
```

### 3. Virtual Thread의 해결 방식

```
Virtual Thread 모델 (M:N 매핑):

  수백만 개 Virtual Thread
  ↕ JVM 스케줄러 (ForkJoinPool)
  수십 개 캐리어 스레드 (= OS 스레드, CPU 코어 수)

Virtual Thread 메모리:
  초기 스택: ~수 KB (힙에 저장, 동적 확장)
  1,000,000개 VT: ~수 GB (힙에 균등 분산)
  vs OS 스레드 1,000개: ~1GB

블로킹 I/O 시 동작:
  Virtual Thread가 Socket.read() 호출
  → JDK가 내부적으로 NIO non-blocking으로 전환
  → 데이터 없음: Continuation 저장 → VT Unmount (캐리어 해제!)
  → 캐리어 스레드: 다른 VT 실행
  → 데이터 도착: Continuation 복원 → VT Mount (캐리어 재할당)
  → Socket.read() 반환, 코드 계속 실행

결과:
  캐리어 스레드(OS 스레드)가 I/O 대기 중 다른 VT를 실행
  CPU가 항상 바쁘게 유지됨
  I/O 대기 = 메모리 위의 Continuation 보관 (CPU 점유 없음)

처리량 계산 (Virtual Thread):
  100ms/요청, 95ms I/O 대기, 5ms CPU
  4코어 캐리어 = 4개 동시 CPU 실행
  CPU 부분만: 4 / 0.005s = 800 req/s (이론)
  실제: I/O 대기 VT가 많으면 캐리어가 항상 바쁨
  → 동시 처리 수 = 캐리어 CPU 효율로 결정 (스레드 수 아님)
```

### 4. Thread-per-Request 모델의 보존

```
Virtual Thread의 핵심 가치: 기존 프로그래밍 모델 유지

기존 동기 코드:
  String result = db.query("SELECT ...");  // 블로킹처럼 보임
  String api = httpClient.get(url);        // 블로킹처럼 보임
  logger.info("result: " + result);

Reactive 코드 (가독성 저하):
  Mono.fromCallable(() -> db.query("SELECT ..."))
      .flatMap(result ->
          WebClient.create().get().uri(url)
              .retrieve().bodyToMono(String.class)
              .map(api -> result + api))
      .doOnNext(combined -> logger.info(combined))
      .subscribe();

Virtual Thread (기존 코드 그대로):
  // 동기 코드와 완전히 동일하게 작성
  // 내부적으로 Non-Blocking으로 동작
  String result = db.query("SELECT ...");  // VT Unmount → Mount
  String api = httpClient.get(url);        // VT Unmount → Mount
  logger.info("result: " + result);

디버깅 편의성:
  Virtual Thread의 스택 트레이스가 동기 코드처럼 보임
  jstack으로 읽기 쉬운 스택 덤프
  예외 스택 트레이스가 직관적
  
  Reactive의 스택 트레이스:
    onNext -> map -> flatMap -> subscribe (읽기 어려움)
```

---

## 💻 실전 실험

### 실험 1: OS 스레드 vs Virtual Thread 생성 비용 비교

```java
public class ThreadCreationBenchmark {
    public static void main(String[] args) throws Exception {
        int count = 100_000;

        // OS 스레드 생성 비용
        long start = System.nanoTime();
        Thread[] threads = new Thread[count];
        for (int i = 0; i < count; i++) {
            threads[i] = new Thread(() -> {
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            });
        }
        for (Thread t : threads) t.start();
        System.out.printf("OS 스레드 %d개 생성: %.2f ms%n",
            count, (System.nanoTime() - start) / 1_000_000.0);
        // 주의: 100,000개 OS 스레드 생성 시 OOM 또는 시스템 한계 초과 가능
        // 실제로는 수천 개가 한계

        // Virtual Thread 생성 비용
        start = System.nanoTime();
        Thread[] vts = new Thread[count];
        for (int i = 0; i < count; i++) {
            vts[i] = Thread.ofVirtual().unstarted(() -> {
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            });
        }
        for (Thread t : vts) t.start();
        System.out.printf("Virtual Thread %d개 생성: %.2f ms%n",
            count, (System.nanoTime() - start) / 1_000_000.0);

        for (Thread t : vts) t.join();
        // Virtual Thread: 수 십 ms, OS 스레드: 수 초 또는 실패
    }
}
```

### 실험 2: I/O 바운드 처리량 비교

```java
import java.util.concurrent.*;
import java.net.http.*;
import java.net.URI;

public class IoThroughputComparison {
    static final int REQUESTS = 1000;
    static final HttpClient client = HttpClient.newHttpClient();

    static void simulateIo() throws Exception {
        Thread.sleep(50);  // 50ms I/O 시뮬레이션
    }

    public static void main(String[] args) throws Exception {
        // 방법 1: 고정 스레드 풀 (200개)
        ExecutorService fixedPool = Executors.newFixedThreadPool(200);
        long start = System.nanoTime();
        CountDownLatch latch1 = new CountDownLatch(REQUESTS);
        for (int i = 0; i < REQUESTS; i++) {
            fixedPool.submit(() -> {
                try { simulateIo(); } catch (Exception e) {}
                finally { latch1.countDown(); }
            });
        }
        latch1.await();
        System.out.printf("고정 풀(200): %.2f ms%n",
            (System.nanoTime() - start) / 1_000_000.0);
        fixedPool.shutdown();

        // 방법 2: Virtual Thread per Task
        ExecutorService vtPool = Executors.newVirtualThreadPerTaskExecutor();
        start = System.nanoTime();
        CountDownLatch latch2 = new CountDownLatch(REQUESTS);
        for (int i = 0; i < REQUESTS; i++) {
            vtPool.submit(() -> {
                try { simulateIo(); } catch (Exception e) {}
                finally { latch2.countDown(); }
            });
        }
        latch2.await();
        System.out.printf("Virtual Thread:  %.2f ms%n",
            (System.nanoTime() - start) / 1_000_000.0);
        vtPool.shutdown();
        // Virtual Thread 예상 결과: ~50ms (모두 동시 실행)
        // 고정 풀 예상 결과: ~250ms (1000/200 × 50ms)
    }
}
```

### 실험 3: Virtual Thread 기본 API 탐색

```java
public class VirtualThreadApiExplore {
    public static void main(String[] args) throws Exception {
        // Virtual Thread 생성 방법들
        Thread vt1 = Thread.ofVirtual().name("my-vt").start(() -> {
            System.out.println("VT1: " + Thread.currentThread().getName());
            System.out.println("VT1 isVirtual: " + Thread.currentThread().isVirtual());
        });

        // Thread.Builder 재사용
        Thread.Builder.OfVirtual builder = Thread.ofVirtual().name("worker-", 0);
        Thread vt2 = builder.start(() -> System.out.println("VT2"));
        Thread vt3 = builder.start(() -> System.out.println("VT3"));

        // ExecutorService 방식 (권장)
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 10; i++) {
                executor.submit(() -> {
                    System.out.println("Task on: " + Thread.currentThread());
                });
            }
        } // AutoCloseable: submit 완료 후 종료

        vt1.join(); vt2.join(); vt3.join();

        // Virtual Thread 정보
        Thread current = Thread.currentThread();
        System.out.println("현재 스레드: " + current);
        System.out.println("Virtual 여부: " + current.isVirtual());
        System.out.println("데몬 여부: " + current.isDaemon());  // VT는 항상 데몬

        // 스레드 상태
        Thread sleepingVt = Thread.ofVirtual().start(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
        });
        Thread.sleep(100);
        System.out.println("VT 상태: " + sleepingVt.getState());  // TIMED_WAITING
        sleepingVt.interrupt();
        sleepingVt.join();
    }
}
```

---

## 📊 성능/비용 비교

```
OS 스레드 vs Virtual Thread 비교:

특성                   | OS 스레드           | Virtual Thread
──────────────────────┼────────────────────┼──────────────────────────
스택 메모리            | ~1MB (고정, 조정 가능)| ~수 KB (힙, 동적 확장)
생성 비용              | ~100μs (OS 시스템콜) | ~수 μs (힙 할당)
컨텍스트 스위칭        | ~1~10μs (OS 스케줄러)| ~수 ns~μs (JVM 수준)
최대 동시 스레드 수     | ~수천 개 (메모리 제한)| ~수백만 개 (힙 크기 제한)
I/O 블로킹 시         | 캐리어(OS 스레드) 차지 | Unmount → 캐리어 해제
스레드 덤프            | jstack 가능          | jstack 가능 (VT 포함)
ThreadLocal            | 정상 사용 가능        | 주의 (100만 VT × 복사)
디버깅                 | 스택 트레이스 직관적   | 동일 (동기 스타일)
기존 코드 호환         | -                   | ✅ (대부분 그대로)

I/O 바운드 처리량 비교 (50ms I/O 요청):
  스레드 풀 200개: ~4,000 req/s (200 / 0.05s)
  스레드 풀 1000개: ~20,000 req/s (1000 / 0.05s, 메모리 ~1GB)
  Virtual Thread: CPU 코어 수 × (CPU 활용률) 기준
                  4코어, 1ms CPU/요청 → ~4,000 req/s CPU bound 도달 전
                  I/O 대기: 무제한 동시 VT → 사실상 처리량 = I/O 완료 속도
```

---

## ⚖️ 트레이드오프

```
Virtual Thread 도입의 트레이드오프:

이점:
  I/O 바운드 작업 처리량 대폭 향상
  기존 동기 코드 그대로 (Reactive 전환 불필요)
  스레드 풀 크기 튜닝 불필요
  직관적인 디버깅/프로파일링

제약사항:
  Pinning 문제: synchronized 블록 내 블로킹 → 캐리어 고착
  ThreadLocal 남용: 100만 VT에 각각 복사 → 메모리 폭증
  스레드 풀링 안티패턴: VT는 풀링하면 안 됨 (매 작업 새로 생성)
  CPU 바운드 작업: VT 이점 없음 (OS 스레드와 동일)

적합한 사용 사례:
  HTTP 서버 (각 요청마다 VT)
  데이터베이스 쿼리 집약 서비스
  마이크로서비스 간 다수의 동기 HTTP 호출
  파일 I/O 집약 처리

부적합한 사용 사례:
  CPU 집약적 연산 (GPU 계산, 이미지 처리)
  Reactive 스트림이 이미 최적화된 시스템
  synchronized 블록이 많고 리팩토링 불가한 레거시
```

---

## 📌 핵심 정리

```
Virtual Thread 핵심:

OS 스레드 한계:
  ~1MB/스레드 → 수천 개가 실용적 상한
  I/O 대기 중 OS 스레드 차지 → CPU 낭비
  Thread-per-Request → 처리량 상한 = 스레드 수 / 응답 시간

Virtual Thread 해결:
  M:N 모델: 수백만 VT ↔ CPU 코어 수의 캐리어 스레드
  블로킹 I/O → 내부적으로 NIO non-blocking
  I/O 대기 시 Continuation 저장 → Unmount → 캐리어 재사용
  처리량 상한 = CPU 바운드 부분 (I/O 대기 무관)

코드 변경 없음:
  기존 동기 코드 그대로
  Thread.ofVirtual() 또는 Executors.newVirtualThreadPerTaskExecutor()
  Spring: spring.threads.virtual.enabled=true

주의:
  Pinning (synchronized 내 블로킹) → Chapter 04
  ThreadLocal 남용 → Chapter 05
  CPU 바운드 작업은 VT 이점 없음
```

---

## 🤔 생각해볼 문제

**Q1.** Virtual Thread가 I/O 바운드 작업에서 이점이 있다면, CPU 바운드 작업(예: 암호화, 이미지 리사이징)에서는 왜 이점이 없는가?

<details>
<summary>해설 보기</summary>

Virtual Thread의 이점은 "I/O 대기 중 캐리어를 해제하여 다른 작업을 실행"하는 데서 온다. CPU 바운드 작업은 I/O 대기가 없어 캐리어를 계속 점유하며 실행된다.

4코어 시스템에서 CPU 바운드 작업 8개를 동시에 실행하면:
- OS 스레드 8개: 4개씩 번갈아 실행 (컨텍스트 스위칭)
- Virtual Thread 8개: 캐리어 4개에 각 2개씩 할당 → 동일하게 4개씩 실행

CPU 바운드에서 VT와 OS 스레드의 처리량 차이는 거의 없다. 오히려 VT 스케줄링 오버헤드로 약간 느릴 수 있다.

CPU 바운드 병렬화는 Virtual Thread가 아닌 `ForkJoinPool`이나 `ParallelStream`, `CompletableFuture`와 CPU 코어 수에 맞는 스레드 풀이 더 적합하다.

</details>

---

**Q2.** Virtual Thread는 항상 데몬 스레드인가? `main()` 메서드 종료 후에도 VT가 실행되는가?

<details>
<summary>해설 보기</summary>

Virtual Thread는 항상 데몬 스레드다 (`isDaemon() == true`). 데몬 스레드는 모든 비데몬 스레드가 종료되면 JVM이 강제 종료할 때 함께 종료된다.

`main()` 스레드(비데몬)가 종료될 때:
- 다른 비데몬 스레드가 없으면 JVM 종료
- 모든 VT(데몬)도 강제 종료

따라서 `main()`에서 VT를 시작하고 `join()`하지 않으면 VT가 완료되기 전에 종료될 수 있다.

올바른 방법:
```java
// 방법 1: join()으로 대기
Thread vt = Thread.ofVirtual().start(task);
vt.join();

// 방법 2: ExecutorService AutoCloseable
try (ExecutorService e = Executors.newVirtualThreadPerTaskExecutor()) {
    e.submit(task);
}  // close() = shutdown() + awaitTermination() → 모든 작업 완료 보장
```

VT가 데몬인 이유: VT를 수백만 개 생성하는 경우 JVM 종료 시 모든 VT를 강제 종료해야 하므로 데몬으로 설계됐다.

</details>

---

**Q3.** 기존 `ThreadPoolExecutor` 기반 코드를 Virtual Thread로 전환할 때 "스레드 풀링 안티패턴"이란 무엇인가?

<details>
<summary>해설 보기</summary>

OS 스레드는 생성 비용(~100μs)이 높아 풀링이 필수다. 그러나 Virtual Thread는 생성 비용이 매우 낮아(~수 μs) 풀링이 오히려 역효과를 낸다.

잘못된 패턴:
```java
// Virtual Thread를 풀에서 재사용 (안티패턴!)
ExecutorService vtPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors(),
    Thread.ofVirtual().factory()  // VT factory로 고정 풀 생성
);
// 문제: VT를 재사용하면 VT가 가져야 할 ThreadLocal 격리가 무너질 수 있음
//       VT 풀링은 메모리 누수 (이전 작업의 ThreadLocal이 남음)
//       그리고 VT의 핵심 이점(무한 확장)을 스스로 제한

올바른 패턴:
ExecutorService e = Executors.newVirtualThreadPerTaskExecutor();
// 각 작업마다 새 VT 생성 → 낮은 생성 비용으로 충분
// ThreadLocal 완전 격리
// 처리량 상한 없음
```

풀링 안티패턴의 핵심: VT를 풀링하면 기존 OS 스레드 풀과 동일한 처리량 상한을 갖게 된다. VT를 쓰는 의미가 없어진다.

</details>

---

<div align="center">

**[⬅️ 이전: Ch5 Exchanger와 Phaser](../concurrent-data-structures/05-exchanger-phaser.md)** | **[홈으로 🏠](../README.md)** | **[다음: Continuation과 스케줄링 ➡️](./02-continuation-scheduling.md)**

</div>
