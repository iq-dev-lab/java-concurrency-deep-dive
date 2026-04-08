# 비동기 프로그래밍 비교 — CompletableFuture vs WebFlux vs Virtual Thread

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CompletableFuture`, Spring WebFlux(Reactor), Virtual Thread의 내부 동작 방식은 어떻게 다른가?
- I/O 바운드 100만 요청 시나리오에서 처리량과 메모리 사용량 차이는?
- 각 방식의 디버깅 난이도와 개발 생산성은 어떻게 비교되는가?
- 기존 코드베이스에서 각 방식으로 마이그레이션하는 비용은?
- "CompletableFuture로 이미 비동기 코드인데 Virtual Thread가 필요한가?"

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Non-Blocking이면 다 같은 것 아닌가?"라는 오해가 많다. CompletableFuture는 여전히 OS 스레드를 소비하고, WebFlux는 러닝 커브가 높으며, Virtual Thread는 Pinning 위험이 있다. 각 방식의 실제 동작 차이를 이해하면 시스템 요구사항에 맞는 정확한 선택을 할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: CompletableFuture = Non-Blocking이라는 오해
  "CompletableFuture로 이미 비동기 처리하고 있으니 최적화됐다"
  → CompletableFuture는 스레드 풀에서 실행 (OS 스레드 사용)
  → thenApply: 이전 단계 완료 스레드에서 또는 공통 풀에서 실행
  → I/O 대기 중에도 OS 스레드를 점유 (Non-Blocking 아님!)

실수 2: WebFlux = 항상 더 빠르다는 오해
  "WebFlux를 쓰면 Spring MVC보다 무조건 빠르겠지"
  → CPU 바운드 작업에서는 오히려 느릴 수 있음 (스케줄러 오버헤드)
  → 블로킹 I/O (JDBC, 구형 라이브러리) 사용 시 이벤트 루프 차단
  → 코드 복잡성 증가로 버그 발생 확률 상승

실수 3: 세 가지가 배타적이라는 오해
  "어떤 걸 선택하면 나머지는 쓸 수 없다"
  → CompletableFuture + Virtual Thread 조합 가능
  → Spring MVC + Virtual Thread 조합 (Spring Boot 3.2+)
  → WebFlux + 특정 작업에만 Virtual Thread 보조
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 세 가지 방식으로 동일한 작업 구현 비교

// === CompletableFuture 방식 ===
CompletableFuture<UserProfile> getProfile_CF(long userId) {
    return CompletableFuture
        .supplyAsync(() -> userRepo.findById(userId), dbPool)     // OS 스레드 사용
        .thenComposeAsync(user ->
            CompletableFuture.supplyAsync(
                () -> orderRepo.findByUser(user.id()), dbPool),   // OS 스레드 사용
            ioPool)
        .thenApply(orders -> new UserProfile(orders));
}
// 특징: 콜백 체인, 스레드 풀 명시 필요, 스택 트레이스 복잡

// === WebFlux (Reactor) 방식 ===
Mono<UserProfile> getProfile_WF(long userId) {
    return userRepository.findById(userId)       // R2DBC Non-Blocking
        .flatMap(user ->
            orderRepository.findByUserId(user.id())
                .collectList()
                .map(orders -> new UserProfile(orders)));
}
// 특징: Mono/Flux 체인, R2DBC 필요, 선언적, 디버깅 어려움

// === Virtual Thread 방식 ===
UserProfile getProfile_VT(long userId) throws Exception {
    // 기존 동기 코드 그대로! (VT가 알아서 Unmount)
    User user = userRepository.findById(userId).orElseThrow();
    List<Order> orders = orderRepository.findByUserId(user.id());
    return new UserProfile(orders);
}
// Executor에서 VT로 실행:
try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<UserProfile> future = exe.submit(() -> getProfile_VT(userId));
    return future.get();
}
// 특징: 기존 JDBC 그대로, 코드 가장 단순, Pinning 주의

// === 병렬 실행: 각 방식의 차이 ===
// CF: 명시적 thenCombine
// WF: Mono.zip() 또는 Flux.merge()
// VT + StructuredTaskScope:
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user   = scope.fork(() -> userRepo.findById(userId));
    var orders = scope.fork(() -> orderRepo.findByUserId(userId));
    scope.join().throwIfFailed();
    return new UserProfile(user.get(), orders.get());
}
```

---

## 🔬 내부 동작 원리

### 1. CompletableFuture 내부 동작

```
CompletableFuture 스레드 모델:

supplyAsync(supplier):
  ForkJoinPool.commonPool()에서 supplier 실행 (기본)
  또는 명시적 Executor에서 실행
  → OS 스레드 사용 (I/O 대기 중에도 스레드 차지!)

thenApply(fn):
  이전 단계가 이미 완료됐으면: 현재 호출 스레드에서 실행
  아직 완료 안 됐으면: 이전 단계의 스레드에서 실행
  → 스레드 재사용 (스케줄링 없음), 하지만 어느 스레드인지 불명확

thenApplyAsync(fn):
  항상 ForkJoinPool.commonPool()에서 실행
  → 새 스케줄링 = 스케줄러 오버헤드 (수십 μs)

thenComposeAsync(fn):
  fn이 반환하는 CompletableFuture가 완료될 때까지 현재 스레드 점유
  → I/O를 감추는 것이지, Non-Blocking이 되는 것이 아님!

실제 문제:
  1000개 CompletableFuture 동시 실행
  각각 DB 쿼리(50ms) 포함
  ForkJoinPool.commonPool() 기본 크기 = CPU 코어 수
  → 4코어: 4개만 동시 실행, 나머지 대기
  → 총 처리 시간: 1000/4 × 50ms = 12,500ms

해결:
  별도 I/O 풀 사용: Executors.newFixedThreadPool(200)
  → 200개 동시 DB 쿼리 = 1000/200 × 50ms = 250ms
  → 하지만 200개 OS 스레드 상시 유지 (메모리 200MB)
```

### 2. WebFlux(Reactor) 내부 동작

```
WebFlux 이벤트 루프 모델:

Netty EventLoop:
  CPU 코어 수만큼의 스레드 (기본 2배)
  각 EventLoop: 무한 루프 → 이벤트 처리
  Non-Blocking I/O (NIO): 데이터 없으면 즉시 반환, Selector 등록

Mono/Flux 체인의 실제 실행:
  subscribe() 호출 → 구독 시작
  → Operator 체인이 데이터가 있을 때마다 실행 (콜백)
  → 모든 실행이 EventLoop 스레드에서 순차적으로

블로킹 호출이 있을 때:
  userRepo.findById(id)  ← JDBC (블로킹!)
  EventLoop 스레드가 DB 응답까지 차단됨
  → 해당 EventLoop의 다른 요청 전부 차단!
  → WebFlux의 치명적 안티패턴

올바른 WebFlux 패턴:
  R2DBC (Reactive 드라이버): DB 쿼리도 Mono/Flux 반환
  Schedulers.boundedElastic(): 블로킹 작업을 별도 풀에서 실행
    userRepo.findById(id) 블로킹이면:
    Mono.fromCallable(() -> userRepo.findById(id))
        .subscribeOn(Schedulers.boundedElastic())  // 블로킹 전용 풀

오류 처리와 역압력(Backpressure):
  Flux가 생산 속도 > 소비 속도 시 역압력 적용
  upstream에 slowdown 요청 → 자동 속도 조절
  → CompletableFuture나 VT에는 없는 기능
```

### 3. Virtual Thread의 실제 동작

```
Virtual Thread 실행 모델:

ForkJoinPool 캐리어 (CPU 코어 수):
  각 VT가 캐리어에 Mount
  블로킹 I/O → Unmount → 캐리어 해방 → 다른 VT 실행

1000개 VT 동시 DB 쿼리 (50ms):
  1000개 VT 생성 (~수 MB 힙)
  모두 동시에 DB 쿼리 시작
  → 거의 동시에 Unmount (DB 응답 대기)
  → 캐리어 4개는 준비된 VT 실행
  → DB 응답 도착하는 대로 Mount → 처리 → 완료
  → 총 처리: ~50ms + 각 VT 처리 시간 (병렬)

실제 병목:
  DB 커넥션 수 (HikariCP: maximum-pool-size)
  → 20개 커넥션이면 20개만 동시 쿼리
  → 나머지 980개 VT는 커넥션 대기 (Unmount 상태)
  → 커넥션이 반납될 때마다 순서대로 실행

메모리 비교 (1000개 동시 처리):
  OS 스레드 1000개:  ~1GB (1MB/스레드)
  VT 1000개:         ~5~50MB (힙)
```

### 4. 세 방식 스택 트레이스 비교

```
동일 작업의 스택 트레이스 비교:

=== CompletableFuture ===
java.lang.Exception
    at UserService.lambda$getProfile$2(UserService.java:45)  // 람다 (읽기 어려움)
    at java.util.concurrent.CompletableFuture$AsyncSupply.run(...)
    at java.util.concurrent.ForkJoinTask.doExec(...)
    at java.util.concurrent.ForkJoinPool.runWorker(...)
→ 실제 비즈니스 로직이 어디에 있는지 파악 어려움

=== WebFlux ===
java.lang.Exception
    at UserService.lambda$getProfile$1(UserService.java:30)
    at reactor.core.publisher.MonoFlatMap$FlatMapMain.onNext(...)
    at reactor.core.publisher.FluxMap$MapSubscriber.onNext(...)
    at reactor.core.publisher.MonoSubscribeOn$SubscribeOnSubscriber.onNext(...)
    at reactor.netty.channel.FluxReceive.drainReceiver(...)
    ... (20~30 프레임의 Reactor 내부 스택)
→ 매우 읽기 어려움, 프레임 수가 많음

=== Virtual Thread ===
java.lang.Exception
    at UserService.getProfile(UserService.java:20)
    at HttpHandler.handle(HttpHandler.java:15)
    at Thread.run(Thread.java:840) [virtual]
→ 동기 코드와 동일하게 직관적
→ [virtual] 태그로 VT임을 구별
```

---

## 💻 실전 실험

### 실험 1: 세 방식의 처리량 비교

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Fork(1)
@Measurement(iterations = 5, time = 10)
public class AsyncComparisonBenchmark {

    static final int IO_DELAY_MS = 50;
    ExecutorService ioPool = Executors.newFixedThreadPool(200);
    ExecutorService vtPool = Executors.newVirtualThreadPerTaskExecutor();

    // CompletableFuture + OS 스레드 풀
    @Benchmark @Threads(1)
    public String completableFuture() throws Exception {
        return CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(IO_DELAY_MS); } catch (InterruptedException e) {}
            return "result";
        }, ioPool).get();
    }

    // Virtual Thread
    @Benchmark @Threads(1)
    public String virtualThread() throws Exception {
        return vtPool.submit(() -> {
            Thread.sleep(IO_DELAY_MS);
            return "result";
        }).get();
    }

    // WebFlux Mono (Reactor) - 별도 벤치마크 환경 필요
    // Mono.delay(Duration.ofMillis(IO_DELAY_MS)).block()
}
```

### 실험 2: 메모리 사용량 비교

```java
public class MemoryComparison {
    static void measure(String name, Runnable setup) throws Exception {
        System.gc();
        long before = Runtime.getRuntime().totalMemory() -
                      Runtime.getRuntime().freeMemory();
        setup.run();
        Thread.sleep(100);
        long after = Runtime.getRuntime().totalMemory() -
                     Runtime.getRuntime().freeMemory();
        System.out.printf("[%s] 메모리 증가: %.2f MB%n",
            name, (after - before) / 1_048_576.0);
    }

    public static void main(String[] args) throws Exception {
        int count = 10_000;

        // OS 스레드 (메모리 무거움 → count 줄여야 할 수 있음)
        measure("OS Thread " + count, () -> {
            CountDownLatch latch = new CountDownLatch(count);
            for (int i = 0; i < count; i++) {
                Thread t = new Thread(() -> {
                    try { Thread.sleep(2000); }
                    catch (InterruptedException e) {}
                    latch.countDown();
                });
                t.start();
            }
        });

        // Virtual Thread
        measure("Virtual Thread " + count, () -> {
            CountDownLatch latch = new CountDownLatch(count);
            for (int i = 0; i < count; i++) {
                Thread.ofVirtual().start(() -> {
                    try { Thread.sleep(2000); }
                    catch (InterruptedException e) {}
                    latch.countDown();
                });
            }
        });
        // VT가 OS 스레드 대비 ~10~50배 적은 메모리 사용
    }
}
```

---

## 📊 성능/비용 비교

```
세 방식 정량 비교 (I/O 바운드, 50ms I/O):

지표                    | CompletableFuture   | WebFlux(Reactor)      | Virtual Thread
───────────────────────┼─────────────────────┼───────────────────────┼────────────────────
동시 처리 단위            | OS 스레드 수          | EventLoop 스레드 수      | VT 수 (무제한)
I/O 대기 중 스레드 소비    | OS 스레드 차지         | EventLoop 해방 (NIO)    | 캐리어 해방 (Unmount)
메모리 (1000 동시)       | ~200MB (200스레드)    | ~수 MB (2~8 EventLoop)  | ~10~50MB (힙 VT)
처리량 (50ms I/O)       | 제한적 (풀 크기)       | 높음 (이벤트 기반)          | 높음 (무제한 VT)
코드 가독성              | 중간 (콜백 체인)       | 낮음 (Mono/Flux 체인)     | 높음 (동기 스타일)
디버깅 난이도            | 중간                  | 높음 (리액티브 스택)       | 낮음 (직관적 스택)
러닝 커브               | 중간                  | 높음                    | 낮음 (기존 지식 활용)
기존 코드 마이그레이션     | 부분 수정               | 전면 재작성              | 최소 변경

CPU 바운드 작업 처리량:
  세 방식 모두 유사 (CPU 코어 수에 의존)
  VT의 이점 없음

역압력(Backpressure):
  CompletableFuture: ❌ (없음)
  WebFlux:           ✅ (Reactor Project의 핵심)
  Virtual Thread:    ❌ (별도 구현 필요)
```

---

## ⚖️ 트레이드오프

```
사용 사례별 선택 가이드:

CompletableFuture 선택:
  병렬 작업 체인이 필요한 경우 (thenCompose, thenCombine)
  기존 스레드 풀 기반 코드에 비동기 추가
  Java 8 이상 환경
  단: I/O 바운드라면 VT가 더 적합

WebFlux(Reactor) 선택:
  극한 처리량 (수만 동시 연결)
  역압력(Backpressure) 필수 (스트리밍, 이벤트 소싱)
  R2DBC 등 Reactive 드라이버 사용 가능
  팀이 Reactive 프로그래밍에 익숙
  MSA에서 서비스 간 Non-Blocking 체인

Virtual Thread 선택:
  I/O 바운드 서비스 (HTTP API, DB 쿼리)
  기존 Spring MVC + JDBC 코드를 최소 변경으로 개선
  팀의 Reactive 학습 비용이 부담
  디버깅/유지보수 가독성 중요
  Spring Boot 3.2+ 환경

혼합 전략 (실무):
  Spring MVC + VT: 요청 처리 레이어
  CompletableFuture: 특정 병렬 작업
  WebFlux: 실시간 스트리밍 엔드포인트만
```

---

## 📌 핵심 정리

```
세 방식 핵심 차이:

CompletableFuture:
  OS 스레드 기반 비동기 콜백
  I/O 대기 중 OS 스레드 점유 (Non-Blocking 아님)
  풀 크기가 동시성 상한
  병렬 작업 조합에 강함

WebFlux(Reactor):
  이벤트 루프 + Non-Blocking I/O
  R2DBC 필수 (JDBC = 이벤트 루프 차단)
  메모리 효율 최고 (소수 EventLoop 스레드)
  역압력, 스트리밍에 강함
  코드 복잡성, 러닝 커브 높음

Virtual Thread:
  기존 동기 코드 + JVM 레벨 Non-Blocking
  I/O 대기 시 캐리어 해방 (Unmount)
  JDBC 그대로 사용 가능 (Pinning 주의)
  코드 가독성 최고
  역압력 없음, CPU 바운드 이점 없음

선택 원칙:
  새 프로젝트 + 팀 경험 없음 → VT + Spring MVC
  처리량 극한 + 스트리밍 → WebFlux
  기존 CF 코드 있음 → CF 유지 또는 VT 점진 전환
  CPU 바운드 → 세 방식 모두 OS 스레드 수가 결정
```

---

## 🤔 생각해볼 문제

**Q1.** `CompletableFuture.thenApply(fn)`에서 `fn`은 어느 스레드에서 실행되는가? 이것이 성능에 미치는 영향은?

<details>
<summary>해설 보기</summary>

`thenApply(fn)`은 두 가지 경우가 있다:

1. **이전 단계가 이미 완료됐을 때**: `fn`을 호출하는 현재 스레드(주로 메인 스레드 또는 이전 콜백을 호출한 스레드)에서 동기적으로 실행된다.

2. **이전 단계가 아직 실행 중일 때**: 이전 단계를 실행 중인 스레드가 완료 후 `fn`을 실행한다.

성능 영향:
- 스레드 전환이 없어서 빠르다 (스케줄러 오버헤드 없음)
- 하지만 어느 스레드에서 실행될지 예측 불가 → 스레드 로컬 데이터 의존 시 문제
- 긴 `fn`이 예상치 못한 스레드를 차지할 수 있음

항상 특정 스레드에서 실행하려면 `thenApplyAsync(fn, executor)`를 사용한다. 하지만 이 경우 스케줄링 오버헤드가 추가된다 (수십 μs).

Virtual Thread에서는 이 문제가 없다. 각 VT가 자신의 스택을 가지고 독립적으로 실행되므로 "어느 캐리어에서 실행되는가"를 신경 쓸 필요가 없다.

</details>

---

**Q2.** WebFlux의 `Schedulers.boundedElastic()`은 Virtual Thread와 무엇이 다른가?

<details>
<summary>해설 보기</summary>

둘 다 블로킹 작업을 이벤트 루프 스레드 밖에서 실행한다는 점에서 유사하지만 근본적으로 다르다.

`Schedulers.boundedElastic()`:
- OS 스레드 풀 (기본 최대 10 × CPU 코어 수)
- 블로킹 작업을 이 풀에서 실행
- 결과가 준비되면 이벤트 루프로 다시 전환 (context switch)
- Mono/Flux 체인 유지 (Reactive 프로그래밍 모델)

Virtual Thread:
- JVM 힙의 Continuation 기반
- 블로킹 I/O 시 Unmount (캐리어 해방)
- 재개 시 캐리어에 Mount (다른 캐리어일 수 있음)
- 동기 코드 스타일 유지

실용적 차이:
- `boundedElastic()`은 OS 스레드 수 제한이 있음 (최대 `10 × CPU`)
- VT는 사실상 무제한 (힙 크기 제한)
- `boundedElastic()` + Reactor = 코드가 여전히 Mono/Flux 체인
- VT = 동기 코드 그대로

WebFlux와 VT를 혼용하는 경우: Reactor 체인 내에서 `Mono.fromCallable(() -> blockingCall()).subscribeOn(Schedulers.fromExecutor(vtExecutor))`로 VT를 사용할 수 있다. 하지만 복잡성만 증가하므로 Spring MVC + VT를 선택하는 것이 일반적으로 더 단순하다.

</details>

---

**Q3.** "CompletableFuture로 이미 비동기 코드인데 Virtual Thread로 마이그레이션이 필요한가?" 판단 기준은?

<details>
<summary>해설 보기</summary>

마이그레이션이 **필요한** 경우:
1. **OS 스레드 수가 병목**: 동시 처리 수를 늘리려고 스레드 풀을 계속 키워야 하는 상황
2. **메모리가 문제**: 수백 개 스레드가 수백 MB를 차지하는 상황
3. **코드 복잡성**: CF 콜백 체인이 유지보수하기 어려운 상황

마이그레이션이 **불필요한** 경우:
1. **CPU 바운드 작업**: VT의 이점이 없음 (OS 스레드와 동일)
2. **이미 처리량 충분**: 현재 부하에서 스레드 부족 없음
3. **CF의 병렬 조합이 핵심**: `thenCombine`, `allOf` 패턴이 핵심 로직 → CF가 더 적합
4. **WebFlux 이미 사용**: 이미 Reactive로 최적화됨

혼합 전략:
```java
// CF의 병렬 조합 + VT 실행자 결합
ExecutorService vtExec = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<User>    userFuture   = CompletableFuture.supplyAsync(
    () -> userRepo.findById(id), vtExec);   // VT에서 실행
CompletableFuture<List<Order>> orderFuture = CompletableFuture.supplyAsync(
    () -> orderRepo.findByUserId(id), vtExec);  // VT에서 실행

// 두 결과 합치기 (CF의 강점)
CompletableFuture.allOf(userFuture, orderFuture)
    .thenApply(v -> new UserProfile(userFuture.join(), orderFuture.join()));
```

→ CF의 병렬 조합 + VT의 I/O 효율 결합이 가능하다.

</details>

---

<div align="center">

**[⬅️ 이전: Lock Contention 진단](./03-lock-contention-profiling.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring 동시성 이슈 ➡️](./05-spring-concurrency-issues.md)**

</div>
