# LongAdder vs AtomicLong — 고경쟁 환경의 Cell 분산

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Striped64`의 `Cell[]` 배열로 CAS 충돌을 분산하는 원리는?
- 고경쟁 환경에서 `AtomicLong` CAS 실패율이 폭증하는 이유는?
- `LongAdder.sum()`이 모든 Cell을 합산할 때 일관성이 보장되지 않는 이유는?
- `LongAdder`와 `AtomicLong` 중 어떤 상황에서 어느 것을 선택해야 하는가?
- `LongAccumulator`와 `DoubleAdder`는 어떤 용도인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Micrometer, Prometheus 같은 메트릭 라이브러리, 서버의 요청 카운터, 통계 집계 코드가 내부적으로 `LongAdder`를 사용한다. 왜 `AtomicLong` 대신 `LongAdder`를 쓰는지 이해하면, 고경쟁 환경에서 올바른 카운터를 선택할 수 있고 직접 통계 수집 코드를 작성할 때도 올바른 선택을 할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 모든 카운터를 AtomicLong으로 구현
  AtomicLong requestCount = new AtomicLong(0);
  // 수천 개 스레드가 초당 수백만 번 increment
  → CAS 실패율 폭증 → CPU 낭비 → 처리량 감소

실수 2: LongAdder가 AtomicLong의 완전한 대체라고 오해
  LongAdder adder = new LongAdder();
  adder.increment();
  if (adder.sum() == 10) doSomething();  // 위험!
  → sum()은 이 시점의 정확한 값을 보장하지 않음
  → 읽는 동안 다른 스레드가 Cell을 업데이트할 수 있음
  → 단순 카운팅은 OK, "특정 값에 도달했는지" 확인은 AtomicLong

실수 3: LongAdder가 항상 AtomicLong보다 빠르다고 가정
  경쟁 없는 환경 (단일 스레드, 낮은 동시성):
    LongAdder: base CAS + 간헐적 Cell 탐색 오버헤드
    AtomicLong: 단순 CAS 1회
    → 낮은 경쟁에서 AtomicLong이 오히려 빠를 수 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 카운터 선택

// 패턴 1: 고경쟁 집계 카운터 → LongAdder
LongAdder requestCounter = new LongAdder();
// 여러 스레드에서 빈번하게
requestCounter.increment();
requestCounter.add(batchSize);

// 주기적 통계 리셋과 함께 읽기
long count = requestCounter.sumThenReset();  // 읽고 0으로 초기화
// 또는 단순 읽기 (순간 스냅샷, 정확하지 않을 수 있음)
long approxCount = requestCounter.sum();

// 패턴 2: 정확한 카운터 또는 상태 관리 → AtomicLong
AtomicLong sequenceNumber = new AtomicLong(0);
long nextId = sequenceNumber.incrementAndGet();  // 고유 ID 생성

AtomicLong version = new AtomicLong(0);
long currentVersion = version.get();
if (version.compareAndSet(currentVersion, currentVersion + 1)) {
    // 버전 확인 및 업데이트 (정확한 CAS 필요)
}

// 패턴 3: 최대/최소/합산 → LongAccumulator
LongAccumulator maxValue = new LongAccumulator(Long::max, Long.MIN_VALUE);
maxValue.accumulate(newValue);  // 스레드 안전한 max 업데이트
long currentMax = maxValue.get();

// 패턴 4: Micrometer Counter (내부적으로 LongAdder 사용)
// Counter counter = meterRegistry.counter("requests.total");
// counter.increment();
```

---

## 🔬 내부 동작 원리

### 1. AtomicLong 고경쟁 시 문제

```
AtomicLong.incrementAndGet() 내부:

  LOCK XADD [address], 1   ← x86 원자적 fetch-and-add
  (또는 CAS 루프로 fallback)

고경쟁 환경에서:
  ┌─────────────────────────────────────────────────────────────┐
  │  단일 캐시 라인 (64바이트, AtomicLong의 value 포함)           │
  │                                                             │
  │  Thread T1 ──→ LOCK XADD ──→ 성공                           │
  │  Thread T2 ──→ LOCK XADD ──→ 대기 (T1이 캐시 라인 독점)      │
  │  Thread T3 ──→ LOCK XADD ──→ 대기                           │
  │  ...                                                        │
  │  Thread T32 ─→ LOCK XADD ──→ 대기                           │
  └─────────────────────────────────────────────────────────────┘

결과:
  32개 스레드가 하나의 캐시 라인을 순서대로 독점
  → 처리량: 단일 스레드의 1/32 수준 (직렬화)
  → CPU는 각자 LOCK XADD를 실행하려다 대기 → 낭비

LOCK 접두사 비용:
  캐시 라인을 독점하는 동안 다른 코어는 해당 캐시 라인 접근 차단
  스레드가 많을수록 대기 시간 증가
  → 경쟁이 N배 증가하면 처리량은 1/N이 아닌 훨씬 나쁠 수 있음
```

### 2. Striped64와 Cell 분산 설계

```
Striped64 (LongAdder, LongAccumulator, DoubleAdder, DoubleAccumulator의 공통 기반):

핵심 필드:
  volatile long base;      // 경쟁 없을 때 사용하는 메인 카운터
  volatile Cell[] cells;   // 경쟁 발생 시 스레드별 Cell 배열
  volatile int cellsBusy;  // cells 배열 초기화/확장 중 플래그

@Contended static final class Cell {
    volatile long value;   // @Contended: 각 Cell이 독립 캐시 라인
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return VALUE.compareAndSet(this, cmp, val);
    }
}

increment() / add(x) 동작 흐름:

  ① cells == null (초기 상태)?
     → base에 CAS 시도
     → 성공: 완료 (경쟁 없음, 빠름)
     → 실패: ② 로 (경쟁 감지!)

  ② Thread에 Cell 할당 (해시로 인덱스 결정)
     스레드 ID 해시 → cells[index] 선택
     → cells[index] CAS 시도
     → 성공: 완료 (다른 Cell이므로 경쟁 없음!)
     → 실패: ③ 로

  ③ cells 배열 확장 또는 Cell 재해시
     cells 배열 크기를 2배로 확장 (최대 CPU 코어 수)
     현재 스레드를 다른 Cell로 재매핑
     → 다시 ② 시도

최종 아키텍처:
  ┌─────────────────────────────────────────────────────────────┐
  │  Thread T1 ──→ Cell[0] (CAS 경쟁 없음!)                     │
  │  Thread T2 ──→ Cell[1] (CAS 경쟁 없음!)                     │
  │  Thread T3 ──→ Cell[2] (CAS 경쟁 없음!)                     │
  │  Thread T4 ──→ Cell[3] (CAS 경쟁 없음!)                     │
  │  ...         각 Cell: 독립 캐시 라인 (@Contended 패딩)       │
  └─────────────────────────────────────────────────────────────┘
  sum() = base + Cell[0].value + Cell[1].value + ... + Cell[n].value
```

### 3. sum()의 비일관성

```
LongAdder.sum() 내부:
  public long sum() {
      Cell[] cs = cells;
      long sum = base;
      if (cs != null) {
          for (Cell c : cs) {
              if (c != null) sum += c.value;
              // 각 Cell.value는 volatile 읽기
          }
      }
      return sum;
  }

왜 일관성이 없는가:

  시간 T1: sum() 시작, base=5 읽음
  시간 T2: Thread A가 Cell[0]에 1 추가 → Cell[0].value=3
  시간 T3: sum()이 Cell[0].value=3 읽음
  시간 T4: Thread B가 Cell[1]에 1 추가 → Cell[1].value=2
  시간 T5: sum()이 Cell[1].value=2 읽음 (T4 이후)
  시간 T6: Thread C가 Cell[0]에 1 추가 → Cell[0].value=4 (sum이 이미 Cell[0] 읽은 후)

  결과: sum = 5 + 3 + 2 = 10
  실제: T1 시점: 5+2+1=8, T6 이후: 5+4+2=11
  → sum이 8도 11도 아닌 10을 반환할 수 있음

  이것은 버그가 아니라 설계 선택:
    sum()에 global lock이나 CAS를 걸면 LongAdder의 성능 이점이 사라짐
    "최종적으로 일관된 합산"을 제공하는 것이 설계 목표
    집계 통계(RPS 계산, 처리량 모니터링)에서는 허용 가능

sumThenReset() 한계:
  sum()과 reset() 사이에도 다른 스레드가 증가시킬 수 있음
  → 정확한 원자적 스냅샷 불가
  → "대략적 기간 내 카운트" 용도
```

### 4. Cell 해시와 스레드 매핑

```
Cell 인덱스 결정 방법:

  Thread마다 probe (해시 시드) 값 보유
    ThreadLocalRandom.getProbe() → 스레드 로컬 해시 값
    처음 할당: 랜덤 초기화
    CAS 충돌 시: advanceProbe() → 새 probe 값 계산 (재해시)

  cells[probe & (cells.length - 1)] 로 Cell 선택
  (배열 크기는 항상 2의 거듭제곱 → & 연산으로 모듈로)

  충돌 발생 시:
    advanceProbe()로 새 인덱스 → 다른 Cell로 이동
    → 각 스레드가 자신만의 Cell을 찾을 때까지 재해시

  배열 크기 확장:
    현재 크기 < CPU 코어 수 → 2배 확장 가능
    최대 크기 = Runtime.getRuntime().availableProcessors()의 2의 거듭제곱
    → CPU 코어 수 이상의 Cell은 불필요 (코어당 하나면 충분)

  이상적 상태:
    32코어 서버, 100스레드: cells 크기 = 64
    100개 스레드가 64개 Cell에 고루 분산
    → 평균 1.5개 스레드/Cell → CAS 충돌 최소화
```

---

## 💻 실전 실험

### 실험 1: 경쟁 수준별 AtomicLong vs LongAdder 처리량

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class AdderVsAtomicBenchmark {

    AtomicLong atomic = new AtomicLong(0);
    LongAdder adder = new LongAdder();

    @Benchmark @Threads(1)
    public void atomic_1t() { atomic.incrementAndGet(); }

    @Benchmark @Threads(4)
    public void atomic_4t() { atomic.incrementAndGet(); }

    @Benchmark @Threads(16)
    public void atomic_16t() { atomic.incrementAndGet(); }

    @Benchmark @Threads(32)
    public void atomic_32t() { atomic.incrementAndGet(); }

    @Benchmark @Threads(1)
    public void adder_1t() { adder.increment(); }

    @Benchmark @Threads(4)
    public void adder_4t() { adder.increment(); }

    @Benchmark @Threads(16)
    public void adder_16t() { adder.increment(); }

    @Benchmark @Threads(32)
    public void adder_32t() { adder.increment(); }
}
// 결과: 1~4 스레드: atomic ≈ adder
//       16~32 스레드: adder가 5~20배 이상 빠름
```

### 실험 2: sum()의 비일관성 확인

```java
import java.util.concurrent.atomic.*;

public class LongAdderConsistencyTest {
    static final LongAdder adder = new LongAdder();
    static final int THREADS = 8, ITERS = 1_000_000;

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERS; j++) adder.increment();
            });
        }
        for (Thread t : threads) t.start();

        // 실행 중 sum()을 계속 읽어 비일관성 관찰
        long prevSum = 0;
        int decreaseCount = 0;
        for (int i = 0; i < 100; i++) {
            Thread.sleep(10);
            long cur = adder.sum();
            if (cur < prevSum) {
                decreaseCount++;  // sum이 줄어든 경우: 비일관성!
                System.out.println("sum 감소: " + prevSum + " → " + cur);
            }
            prevSum = cur;
        }

        for (Thread t : threads) t.join();
        System.out.println("최종 sum: " + adder.sum() +
            " (기대: " + (long)THREADS * ITERS + ")");
        System.out.println("중간 감소 횟수: " + decreaseCount +
            " (비일관성 관찰)");
    }
}
```

### 실험 3: LongAccumulator로 최대값 추적

```java
import java.util.concurrent.atomic.*;

public class MaxTracker {
    // 스레드 안전한 최대값 추적
    static LongAccumulator maxLatency = new LongAccumulator(Long::max, 0L);
    static LongAccumulator minLatency = new LongAccumulator(Long::min, Long.MAX_VALUE);
    static LongAdder totalLatency = new LongAdder();
    static LongAdder requestCount = new LongAdder();

    static void recordLatency(long latencyMs) {
        maxLatency.accumulate(latencyMs);
        minLatency.accumulate(latencyMs);
        totalLatency.add(latencyMs);
        requestCount.increment();
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[16];
        for (int i = 0; i < 16; i++) {
            final int tid = i;
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    long latency = (long)(Math.random() * 1000 + tid * 10);
                    recordLatency(latency);
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();

        long count = requestCount.sum();
        System.out.printf("요청 수:   %,d%n", count);
        System.out.printf("최대 지연: %d ms%n", maxLatency.get());
        System.out.printf("최소 지연: %d ms%n", minLatency.get());
        System.out.printf("평균 지연: %.2f ms%n", (double)totalLatency.sum() / count);
    }
}
```

---

## 📊 성능/비용 비교

```
스레드 수별 처리량 비교 (JMH, increment/ms):

스레드 수    | AtomicLong   | LongAdder   | 비율 (Adder/Atomic)
───────────┼─────────────┼─────────────┼─────────────────────
1           | 500,000     | 450,000     | 0.9x (약간 느림)
2           | 350,000     | 490,000     | 1.4x
4           | 200,000     | 520,000     | 2.6x
8           | 100,000     | 530,000     | 5.3x
16          | 50,000      | 540,000     | 10.8x
32          | 25,000      | 545,000     | 21.8x

→ 경쟁이 없으면 AtomicLong이 약간 빠름 (Cell 배열 탐색 오버헤드)
→ 경쟁이 심해질수록 LongAdder 이점 폭발적으로 증가
→ CPU 코어 수 이상의 스레드에서 LongAdder는 거의 선형 확장

메모리 사용량:
  AtomicLong: ~24바이트 (헤더 + long)
  LongAdder (무경쟁): ~56바이트 (Striped64 기본)
  LongAdder (경쟁 후, 8코어): + 8개 Cell × 128바이트(@Contended) = ~1KB
  → LongAdder는 경쟁 발생 시 메모리 증가 (Cell 배열 확장)
```

---

## ⚖️ 트레이드오프

```
LongAdder vs AtomicLong 선택 기준:

LongAdder 선택:
  고경쟁 집계 카운터 (요청 수, 이벤트 수, 바이트 수)
  "정확한 현재 값"보다 "빠른 증가"가 중요
  Micrometer, Prometheus 메트릭 수집
  최종 합산 시 약간의 오차 허용 가능한 통계

AtomicLong 선택:
  정확한 시퀀스 번호 생성 (ID, 버전)
  CAS로 상태 전환 (compare-and-set 패턴)
  특정 임계값 도달 감지 (if count == 10)
  낮은 경쟁 환경 (단일 스레드, 소수 스레드)

LongAccumulator 선택:
  최대/최소/누적 등 임의의 이항 연산
  LongAdder보다 더 일반적인 연산 필요 시

주의 사항:
  LongAdder.sum()은 스냅샷 보장 없음
  LongAdder.reset()은 원자적이지 않음 (sum과 별도)
  sumThenReset()은 sum+reset을 단일 메서드로 제공하지만 원자적이지 않음
  → 정확한 원자적 스냅샷 필요: AtomicLong 사용
```

---

## 📌 핵심 정리

```
LongAdder / Striped64 핵심:

문제:
  AtomicLong 고경쟁 시 하나의 캐시 라인을 놓고 경합
  → CAS 실패 폭증 → CPU 낭비 → 처리량 하락

해결 (Striped64):
  base: 경쟁 없을 때 메인 카운터
  cells[]: 경쟁 발생 시 스레드별 Cell 할당
  @Contended: 각 Cell이 독립 캐시 라인 (False Sharing 방지)
  스레드 해시로 Cell 선택 → CAS 충돌 최소화

sum():
  base + Σ cells[i].value
  일관성 보장 없음 (읽는 동안 다른 스레드가 수정 가능)
  → 집계 통계에 적합, 정확한 상태 관리에는 부적합

선택 기준:
  고경쟁 카운터       → LongAdder
  정확한 값/CAS 필요 → AtomicLong
  최대/최소/임의 연산 → LongAccumulator
  낮은 경쟁         → AtomicLong (오버헤드 없음)
```

---

## 🤔 생각해볼 문제

**Q1.** `LongAdder`의 Cell 배열 크기가 CPU 코어 수를 넘지 않는 이유는?

<details>
<summary>해설 보기</summary>

Cell이 CPU 코어보다 많아도 동시에 서로 다른 Cell에서 CAS를 실행할 수 있는 스레드는 코어 수만큼이다. 추가 Cell은 불필요한 메모리만 차지한다.

코어 수보다 많은 스레드가 있어도(M:1 매핑 제외), 동시에 CPU를 점유하는 스레드는 코어 수를 초과할 수 없다. 따라서 코어 수 = Cell 수가 이론적 최적이다. Striped64는 `Runtime.getRuntime().availableProcessors()`의 2의 거듭제곱 이상으로는 확장하지 않는다.

Virtual Thread(M:N)에서는 다수의 Virtual Thread가 소수의 캐리어 스레드(OS 스레드)에 매핑되므로, 이 제한이 여전히 합리적이다.

</details>

---

**Q2.** `LongAdder.sumThenReset()`을 두 스레드가 동시에 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`sumThenReset()`은 원자적이지 않다. 두 스레드가 동시에 호출하면:
1. 두 스레드 모두 비슷한 합계를 반환할 수 있다 (중복 집계)
2. 일부 값이 두 번 집계되거나 누락될 수 있다

내부적으로 base를 0으로 리셋하고 각 Cell을 0으로 리셋하는 동안, 다른 스레드가 일부 Cell에 새로운 값을 추가할 수 있다. 이 값은 이미 리셋된 Cell에 쓰여서 다음 집계 때 포함된다.

실무에서 `sumThenReset()`은 단일 스레드가 주기적으로 호출하는 경우에만 올바르다. 여러 스레드가 동시에 읽어야 한다면 별도 동기화가 필요하거나, `AtomicLong`으로 설계를 변경해야 한다.

</details>

---

**Q3.** `LongAdder`의 Cell 배열이 한번 확장되면 축소되지 않는다. 이것이 문제가 될 수 있는가?

<details>
<summary>해설 보기</summary>

잠재적으로 문제가 될 수 있다. 트래픽 스파이크 동안 Cell 배열이 최대 크기까지 확장된 후, 낮은 트래픽 기간에도 메모리를 계속 차지한다.

@Contended가 각 Cell에 128바이트 패딩을 추가하므로, 8코어 서버에서 Cell 8개 = 8 × 128바이트 = 1KB. 수백만 개의 `LongAdder` 인스턴스가 있다면(모든 API 엔드포인트별 카운터 등) 이 메모리가 문제가 될 수 있다.

해결책:
1. 이미 확장된 `LongAdder`는 재사용하거나 `new LongAdder()`로 교체
2. 사용 빈도가 낮은 카운터는 `AtomicLong` 사용
3. Micrometer의 `Counter`처럼 레지스트리가 관리하면 내부적으로 적절히 처리

이것이 Micrometer가 메트릭 수집에 `LongAdder`를 직접 노출하지 않고 `Counter` 추상화 뒤에 감추는 이유 중 하나다.

</details>

---

<div align="center">

**[⬅️ 이전: AtomicReference와 ABA 해결](./03-atomic-reference-stamped.md)** | **[홈으로 🏠](../README.md)** | **[다음: Lock-Free 자료구조 ➡️](./05-lock-free-data-structures.md)**

</div>
