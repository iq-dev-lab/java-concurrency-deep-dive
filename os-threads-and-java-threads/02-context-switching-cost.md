# 컨텍스트 스위칭 비용의 실체 — 레지스터·TLB·CPU 캐시

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 컨텍스트 스위칭이 정확히 무엇을 저장하고 복원하는가?
- TLB 플러시와 CPU 캐시 무효화가 왜 처리량에 영향을 미치는가?
- 자발적 vs 비자발적 컨텍스트 스위칭의 차이는 무엇인가?
- 스레드 수가 늘어날수록 처리량이 왜 역전되는가?
- JMH로 컨텍스트 스위칭 비용을 어떻게 측정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Thread Pool 크기를 코어 수의 10배로 설정했는데 처리량이 기대만큼 나오지 않는다면, 컨텍스트 스위칭 비용을 이해하지 못한 것이다. 스레드는 CPU를 점유하지 않을 때도 OS 스케줄러의 관리 비용을 유발한다. "스레드를 늘리면 빨라진다"는 직관이 무너지는 지점이 바로 컨텍스트 스위칭 오버헤드다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: 8코어 서버에서 처리량을 극대화하려고 스레드 수를 늘림

원리를 모를 때의 판단:
  CPU 사용률이 60%인데 처리량이 부족 → 스레드 더 늘리자
  ThreadPoolExecutor corePoolSize=8 → 100 → 500

결과:
  스레드 100개: 처리량 약간 증가
  스레드 500개: 처리량이 100개 때보다 오히려 감소
  CPU 사용률: 90%↑ (하지만 실제 작업이 아닌 스케줄링에 소비)
  응답 시간 P99: 3배 이상 증가

원인을 모른 채:
  "서버 사양이 부족하다" → 더 비싼 인스턴스로 이전
  → 마찬가지로 동일 패턴 반복
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```
컨텍스트 스위칭 비용을 알 때의 판단:

CPU 바운드 작업:
  최적 스레드 수 = CPU 코어 수 (±1~2)
  이유: 코어 수보다 많은 스레드는 서로 교대하며 컨텍스트 스위칭 발생
  → 실제 계산 시간 감소, 스위칭 오버헤드 증가

I/O 바운드 작업:
  최적 스레드 수 = 코어 수 × (1 + I/O대기시간/CPU처리시간)
  하지만 실제 측정 없이 공식만 믿지 말 것
  → JMH 벤치마크로 실제 처리량 측정 후 결정

측정 지표:
  vmstat 1 | grep cs           # 초당 컨텍스트 스위칭 횟수
  pidstat -w -p <pid> 1        # 프로세스별 컨텍스트 스위칭 상세
  perf stat -e context-switches java MyApp  # 정확한 카운터

판단 기준:
  초당 컨텍스트 스위칭 > 100,000회 → 스레드 수 과다 의심
  CPU 사용률은 높은데 처리량이 낮다 → 스위칭 오버헤드 확인
```

---

## 🔬 내부 동작 원리

### 1. 컨텍스트 스위칭이 저장/복원하는 것들

```
컨텍스트 스위칭 발생 시 OS가 하는 일:

[저장 단계 — 현재 스레드(Thread A)를 중단]

  1. CPU 레지스터 저장 → PCB(Process Control Block)
     범용 레지스터: rax, rbx, rcx, rdx, rsi, rdi, ...  (x86-64 기준 16개)
     스택 포인터: rsp (현재 스택의 최상단 위치)
     명령어 포인터: rip (다음 실행할 명령어 주소)
     플래그 레지스터: rflags (연산 결과 상태)
     세그먼트 레지스터: cs, ss, ds, es, fs, gs
     FPU/SSE 상태: xmm0~xmm15 (부동소수점/SIMD 레지스터)
       → 이것만 512바이트 (fxsave 명령어로 저장)

  비용:
    레지스터 저장: ~수십 ns
    PCB 업데이트: 메모리 쓰기

[전환 단계 — 다음 스레드(Thread B) 선택]

  2. OS 스케줄러가 실행 큐에서 다음 스레드 선택
     O(1) 스케줄러 또는 CFS(Completely Fair Scheduler)
     비용: ~수백 ns (스케줄러 알고리즘 실행)

[복원 단계 — Thread B 실행 재개]

  3. Thread B의 PCB에서 레지스터 복원
     → Thread B가 중단됐던 지점부터 실행 재개

전체 컨텍스트 스위칭 비용 (순수 스위칭):
  ~1 μs ~ 10 μs (하드웨어, OS, 스레드 상태에 따라 다름)
  하지만 이게 전부가 아니다 → TLB, CPU 캐시 영향이 더 크다
```

### 2. TLB 플러시 — 보이지 않는 큰 비용

```
TLB(Translation Lookaside Buffer)란:
  가상 주소 → 물리 주소 변환 캐시
  CPU 내부 하드웨어, 매우 빠름 (~1 사이클)

  가상 주소 0x7fff1234
    ↓ TLB 히트!
  물리 주소 0x3abc5678  (즉시 변환)

  TLB 미스 시:
    Page Table을 메모리에서 직접 탐색
    → 4단계 페이지 테이블 워크 (x86-64)
    → ~수십 사이클 (메모리 접근 4회)

같은 프로세스 내 스레드 전환 (스레드끼리):
  동일한 가상 주소 공간 공유
  → TLB 항목 유지 가능 (부분적으로)
  → TLB 영향 최소

프로세스 전환 시:
  완전한 TLB 플러시 발생
  → 다음 프로세스가 메모리 접근할 때마다 TLB 미스
  → 초기 수백 ms 동안 성능 저하

Java 스레드 간 전환:
  같은 JVM 프로세스 내 스레드 → 주소 공간 공유
  → TLB 플러시 없음 (같은 프로세스)
  하지만 CPU 캐시 오염은 발생
```

### 3. CPU 캐시 오염 — 컨텍스트 스위칭의 진짜 비용

```
CPU 캐시 계층 (코어당):
  L1 캐시: ~32KB, ~1ns 접근, 4사이클
  L2 캐시: ~256KB, ~4ns 접근, 12사이클
  L3 캐시: ~수 MB, ~10ns 접근, 30사이클 (코어 공유)
  RAM:      ~수 GB, ~100ns 접근, 300사이클

Thread A가 실행 중일 때 캐시 상태:
  L1/L2: Thread A의 데이터로 가득 차 있음
  → Thread A의 모든 메모리 접근이 빠름 (캐시 히트)

  컨텍스트 스위치 → Thread B 실행:
  L1/L2: 여전히 Thread A의 데이터 (Thread B에게 불필요)
  Thread B가 자신의 데이터에 접근
    → 캐시 미스 → RAM에서 로드 (~100ns, 300사이클)
    → Thread B의 데이터로 캐시를 채워가는 과정
  
  "워밍업" 기간: Thread B가 캐시를 자신의 데이터로 채우는 시간
  → 수 ms 동안 캐시 미스율 폭증 → 처리량 저하

실제 측정:
  스레드 전환 직후 메모리 집약적 작업의 처리량:
  캐시가 따뜻할 때: 100%
  컨텍스트 스위치 직후: 40~60% (캐시 미스로 인한 대기)
  → 이것이 스레드 수 증가 시 처리량 역전의 핵심 원인

False Sharing (위공유):
  다른 스레드가 같은 캐시 라인(64바이트)의 다른 변수를 수정
  → 한 스레드의 쓰기가 다른 스레드의 캐시를 무효화
  → 실제로는 별개 변수지만 캐시 라인이 같아 성능 저하
  → @Contended 어노테이션으로 캐시 라인 분리 가능
```

### 4. 자발적 vs 비자발적 컨텍스트 스위칭

```
자발적 컨텍스트 스위칭 (Voluntary Context Switch):
  스레드 스스로 CPU를 양보
  발생 조건:
    Thread.sleep(n)     → TIMED_WAITING 상태
    Object.wait()       → WAITING 상태
    LockSupport.park()  → WAITING 상태
    I/O 블로킹          → WAITING 상태 (소켓 read 등)
    Thread.yield()      → RUNNABLE (재실행 가능 상태 유지)
  
  특징:
    스레드가 어차피 기다려야 할 때 발생
    CPU를 소모하지 않는 대기 → 효율적
    pidstat에서 cswch/s (voluntary)로 표시

비자발적 컨텍스트 스위칭 (Involuntary Context Switch):
  OS 스케줄러가 강제로 스레드를 중단
  발생 조건:
    타임 슬라이스 만료 (기본 ~10ms)
    더 높은 우선순위 스레드 등장
    멀티코어에서 로드 밸런싱
  
  특징:
    스레드가 계속 실행하고 싶지만 강제로 중단
    불필요한 캐시 오염 유발
    pidstat에서 nvcswch/s (non-voluntary)로 표시
    → 높은 nvcswch/s = 스레드가 CPU 경합 중 = 스레드 수 과다 의심

이상적인 상태:
  CPU 바운드 작업: 비자발적 스위칭 최소화 (스레드 = 코어 수)
  I/O 바운드 작업: 자발적 스위칭이 대부분 (I/O 대기 중 양보)
```

### 5. 스레드 수와 처리량의 관계

```
8코어 서버, CPU 바운드 작업 예시:

스레드 수    실행 가능한 스레드    컨텍스트 스위칭/초    처리량
─────────────────────────────────────────────────────────
1            1                  0                    12.5%
4            4                  ~0                   50%
8            8                  ~0 (이상적)            100%
16           8 (코어 한계)        ~8,000               90%  ← 스위칭 오버헤드 시작
32           8 (코어 한계)        ~24,000              75%
100          8 (코어 한계)        ~80,000              50%  ← 역전
500          8 (코어 한계)        ~400,000             20%  ← 스케줄러가 병목

스레드 수 > 코어 수인 경우:
  코어 N개, 스레드 M개 (M >> N)
  각 스레드의 타임슬라이스 = 총 CPU 시간 / M
  타임슬라이스가 짧아질수록:
    → 실제 작업 시간 감소
    → 컨텍스트 스위칭 비율 증가
    → 캐시 워밍업 효율 감소

I/O 바운드 작업은 다르다:
  스레드가 I/O 대기 중이면 CPU를 사용하지 않음
  → 다른 스레드가 CPU 활용 가능
  → 스레드 수 > 코어 수가 도움이 됨
  하지만 모든 스레드가 동시에 CPU를 원할 때 역전 발생
```

---

## 💻 실전 실험

### 실험 1: vmstat으로 컨텍스트 스위칭 측정

```bash
# 컨텍스트 스위칭 많은 프로그램 실행
cat > ContextSwitchTest.java << 'EOF'
public class ContextSwitchTest {
    static volatile long counter = 0;
    
    public static void main(String[] args) throws InterruptedException {
        int threadCount = Integer.parseInt(args[0]);
        System.out.println("스레드 수: " + threadCount);
        
        Thread[] threads = new Thread[threadCount];
        for (int i = 0; i < threadCount; i++) {
            threads[i] = new Thread(() -> {
                long start = System.currentTimeMillis();
                while (System.currentTimeMillis() - start < 10000) {
                    counter++;  // CPU 경합 유발
                }
            });
            threads[i].start();
        }
        
        // 1초마다 counter 출력
        for (int i = 0; i < 10; i++) {
            long prev = counter;
            Thread.sleep(1000);
            System.out.printf("초당 처리량: %,d%n", counter - prev);
        }
    }
}
EOF

javac ContextSwitchTest.java

# 터미널 1: vmstat으로 컨텍스트 스위칭 모니터링
vmstat 1 30 | awk 'NR==1{print} NR>2{print $12, $13, "cs/s"}'
# cs 컬럼이 초당 컨텍스트 스위칭 수

# 터미널 2: 각각 실행하며 cs 변화 확인
java ContextSwitchTest 1    # 최소 스위칭
java ContextSwitchTest 8    # 코어 수와 동일
java ContextSwitchTest 32   # 코어 수의 4배
java ContextSwitchTest 100  # 과도한 스레드
```

### 실험 2: pidstat으로 스레드별 컨텍스트 스위칭 상세 분석

```bash
# Java 프로그램 실행 후 PID 확인
java -Xss512k ContextSwitchTest 50 &
PID=$!

# pidstat으로 프로세스별 컨텍스트 스위칭 측정
# -w: 컨텍스트 스위칭 통계
# -t: 스레드 레벨
pidstat -w -t -p $PID 1 10

# 출력:
# TID   cswch/s  nvcswch/s  Command
# 자발적(cswch) vs 비자발적(nvcswch) 비율로 상태 진단
```

### 실험 3: JMH로 스레드 수별 처리량 벤치마크

```java
// pom.xml에 JMH 의존성 추가 후 실행
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 3, time = 2)
@Measurement(iterations = 5, time = 2)
public class ThreadCountBenchmark {

    private AtomicLong counter = new AtomicLong();

    @Benchmark
    @Threads(1)
    public long singleThread() {
        return counter.incrementAndGet();
    }

    @Benchmark
    @Threads(4)
    public long fourThreads() {
        return counter.incrementAndGet();
    }

    @Benchmark
    @Threads(8)
    public long eightThreads() {
        return counter.incrementAndGet();
    }

    @Benchmark
    @Threads(32)
    public long thirtyTwoThreads() {
        return counter.incrementAndGet();
    }

    @Benchmark
    @Threads(100)
    public long hundredThreads() {
        return counter.incrementAndGet();
    }
}

// 실행:
// java -jar benchmarks.jar .*ThreadCountBenchmark.* -f 2 -wi 3 -i 5
```

### 실험 4: perf로 CPU 캐시 미스율 측정

```bash
# perf stat으로 캐시 관련 카운터 측정
perf stat -e cache-references,cache-misses,context-switches \
    java ContextSwitchTest 8 2>&1

# 비교: 스레드 수에 따른 캐시 미스율 변화
perf stat -e cache-misses java ContextSwitchTest 8
perf stat -e cache-misses java ContextSwitchTest 100

# 출력 예시:
#  Performance counter stats for 'java ContextSwitchTest 100':
#    2,345,123      cache-references
#    1,876,543      cache-misses         # 80% 미스율! (정상은 5~15%)
#       45,234      context-switches
```

---

## 📊 성능/비용 비교

```
컨텍스트 스위칭 구성 요소별 비용:

구성 요소                    | 비용          | 비고
───────────────────────────┼──────────────┼─────────────────────
레지스터 저장/복원             | ~100 ns       | x86-64 범용 레지스터 16개
FPU/SSE 상태 저장/복원        | ~50 ns        | xmm 레지스터 (fxsave)
OS 스케줄러 실행              | ~500 ns       | CFS 알고리즘
TLB 영향 (같은 프로세스)       | 최소           | 주소 공간 공유
CPU L1/L2 캐시 오염          | ~수 ms        | 워밍업 기간 동안 처리량 저하
─────────────────────────────────────────────────────────────
순수 스위칭 오버헤드           | ~1~10 μs      |
실효 처리량 영향              | 최대 수십 %     | 캐시 미스 효과 포함

스레드 수에 따른 초당 컨텍스트 스위칭 (CPU 바운드, 8코어 기준):

스레드 수    | 초당 CS 횟수  | CPU 스위칭 오버헤드| 처리량 (상대)
───────────┼─────────────┼───────────────┼──────────────
8          | ~100        | <1%           | 100%
16         | ~8,000      | ~1%           | 97%
32         | ~24,000     | ~5%           | 88%
100        | ~80,000     | ~15%          | 70%
500        | ~400,000    | ~40%          | 45%
```

---

## ⚖️ 트레이드오프

```
스레드 수 늘리기의 장단점:

장점:
  I/O 대기 시 다른 스레드가 CPU 활용 가능
  동시 요청 처리 수 증가
  응답 시간 분산 (한 요청이 늦어도 다른 요청은 처리)

단점:
  컨텍스트 스위칭 오버헤드 증가
  CPU 캐시 오염 → 캐시 미스율 증가
  메모리 소비 (스레드 스택)
  OS 스케줄러 부하 증가

최적점 찾기:
  정답은 없다. 워크로드 특성에 따라 다름
  → 반드시 JMH 또는 실제 부하 테스트로 측정

Virtual Thread와의 비교:
  OS 스레드의 컨텍스트 스위칭: OS 스케줄러, CPU 캐시 오염
  Virtual Thread의 스위칭: JVM 스케줄러, 같은 캐리어 스레드에서
    Continuation 교체 → CPU 캐시 오염 최소화
  → I/O 바운드에서 Virtual Thread가 훨씬 효율적인 이유
```

---

## 📌 핵심 정리

```
컨텍스트 스위칭 비용의 실체:

스위칭 시 발생하는 일:
  레지스터 저장/복원: ~100ns (빠름)
  OS 스케줄러 실행: ~500ns
  CPU 캐시 오염: 워밍업 기간 수 ms (이게 진짜 비용)

자발적 vs 비자발적:
  자발적: I/O 대기, sleep → 효율적 (어차피 기다려야 함)
  비자발적: 타임슬라이스 만료 → 비효율 (작업 중 강제 중단)

스레드 수 최적화:
  CPU 바운드: 코어 수 = 최적
  I/O 바운드: 코어 수 × (1 + 대기/처리) → 측정 필수

진단:
  vmstat 1 | cs 컬럼     # 초당 컨텍스트 스위칭
  pidstat -w -p <pid>    # 프로세스별 자발/비자발 구분
  perf stat -e cache-misses  # 캐시 미스율 확인

경고 신호:
  초당 CS > 100,000회 → 스레드 수 과다
  캐시 미스율 > 30% → 스레드 간 캐시 경합
  nvcswch > cswch → CPU 경합 심각
```

---

## 🤔 생각해볼 문제

**Q1.** `Thread.yield()`를 호출하면 컨텍스트 스위칭이 발생하는가? `Thread.sleep(0)`과 어떻게 다른가?

<details>
<summary>해설 보기</summary>

`Thread.yield()`는 현재 스레드를 `RUNNABLE` 상태로 유지하면서 OS 스케줄러에게 "다른 스레드를 먼저 실행해도 된다"고 힌트를 준다. 실제로 스위칭이 일어날지는 OS 스케줄러의 판단이며, 힌트가 무시될 수도 있다. 스레드는 여전히 실행 가능 상태이므로 곧바로 다시 스케줄링될 수 있다.

`Thread.sleep(0)`은 JVM에 따라 다르게 동작하지만, 일반적으로 `yield()`와 유사하게 동작한다. 일부 구현에서는 OS 스케줄러에 실제로 시간을 양보한다.

두 방법 모두 스핀 락이나 바쁜 대기(busy-wait) 상황에서 CPU를 과다 소비하는 것을 줄이는 데 사용되지만, `LockSupport.park()`처럼 스레드를 완전히 블로킹하지는 않으므로 CPU 점유는 계속된다.

</details>

---

**Q2.** 8코어 서버에서 순수 CPU 바운드 작업을 처리하는 Thread Pool이 있다. 스레드 수를 9개로 하면 8개보다 처리량이 낮아지는가?

<details>
<summary>해설 보기</summary>

이론적으로는 낮아질 수 있지만, 실제로는 **거의 차이 없거나 약간 낮아진다**. 이유는 다음과 같다.

- 8코어에 9스레드: 항상 하나의 스레드가 대기하게 되어 컨텍스트 스위칭이 소량 발생
- 하지만 JVM GC, Compiler 스레드 등도 CPU를 소비하므로 실제로는 "코어 수 + 1" 정도가 나을 수도 있음
- 타임슬라이스(~10ms)와 컨텍스트 스위칭 비용(~수 μs)의 비율을 생각하면, 스레드 9개의 오버헤드는 미미함

실제로 JMH로 측정해보면 8 vs 9 스레드의 차이는 1~2% 수준이거나 측정 오차 범위 내인 경우가 많다. 스레드가 30개, 100개로 늘어날 때 의미 있는 성능 저하가 나타난다.

</details>

---

**Q3.** False Sharing이 컨텍스트 스위칭과 어떤 관계가 있는가? `@Contended` 어노테이션은 어떻게 이를 해결하는가?

<details>
<summary>해설 보기</summary>

False Sharing은 컨텍스트 스위칭과 직접 관련은 없지만, 멀티코어 환경에서 **캐시 라인 무효화**라는 유사한 메커니즘으로 성능을 저하시킨다.

CPU 캐시는 64바이트 캐시 라인 단위로 동작한다. 서로 다른 스레드가 논리적으로 별개의 변수를 수정하더라도, 두 변수가 같은 64바이트 캐시 라인에 위치하면 한 스레드의 쓰기가 다른 코어의 캐시를 무효화한다(MESI 프로토콜의 Invalidate).

```java
// False Sharing 발생 예시
class Counter {
    long a;  // 두 변수가 같은 캐시 라인에 위치
    long b;
}
// Thread 1이 a를 수정 → Thread 2의 b 캐시도 무효화

// @Contended로 해결
class Counter {
    @Contended long a;  // 캐시 라인 패딩 추가 (64바이트 정렬)
    @Contended long b;  // 별개의 캐시 라인에 배치
}
// -XX:-RestrictContended JVM 플래그 필요
```

`@Contended`는 해당 필드 앞뒤에 패딩 바이트를 추가하여 다른 변수와 다른 캐시 라인에 배치한다. `LongAdder`의 `Cell` 배열이 내부적으로 `@Contended`를 사용하는 이유다.

</details>

---

<div align="center">

**[⬅️ 이전: OS 스레드 모델](./01-os-thread-model.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스레드 생성과 Thread Pool ➡️](./03-thread-pool-executor.md)**

</div>
