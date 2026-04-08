# 성능 병목 패턴 — Lock Contention과 JFR 진단

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Lock Contention(락 경합)이 처리량을 저하시키는 원리는?
- JFR(Java Flight Recorder)의 `jdk.MonitorEnter` 이벤트로 경합 지점을 찾는 방법은?
- `async-profiler`의 `-e lock` 모드로 락 대기 시간 Flame Graph를 생성하는 방법은?
- 과도한 `synchronized` 범위를 줄이는 리팩토링 패턴은?
- Lock Contention과 데드락을 어떻게 구별하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Lock Contention은 데드락처럼 완전히 멈추지 않아서 탐지하기 어렵다. 서비스가 느리고 CPU 사용률이 낮은데 응답 지연이 높다면 Lock Contention이 주범인 경우가 많다. JFR과 async-profiler로 정확한 경합 지점을 찾으면 코드 한 줄의 리팩토링으로 처리량이 수 배 향상될 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: synchronized 범위가 너무 넓음
  public synchronized String processRequest(Request req) {
      // 1. 검증 (빠름, 락 불필요)
      validate(req);
      // 2. 캐시 조회 (빠름, concurrent 맵이면 락 불필요)
      String cached = cache.get(req.key());
      if (cached != null) return cached;
      // 3. 외부 API 호출 (느림! 여기서 다른 스레드 전부 대기)
      String result = externalApi.fetch(req.key());
      // 4. 캐시 저장 (빠름, 락 불필요)
      cache.put(req.key(), result);
      return result;
  }
  // 전체 메서드가 하나의 락 → 외부 API 호출 동안 모든 요청 직렬화!

실수 2: CPU 낮음 = 아무 문제 없음이라는 오해
  CPU: 10%, 응답 시간: 2초 → "서버에 여유가 있는데 왜 느리지?"
  → 스레드들이 락 대기 중 (BLOCKED) → CPU 소비 없음
  → 실제로는 락 경합으로 직렬화됨

실수 3: JFR 없이 추측으로 최적화
  "synchronized를 전부 제거하면 빠르겠지" (→ 안전성 훼손)
  "스레드 풀을 늘리면 해결되겠지" (→ 경합만 증가)
  → JFR/async-profiler로 정확한 경합 지점 먼저 파악
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Lock Contention 해결 패턴

// 패턴 1: synchronized 범위 최소화
public String processRequest(Request req) {
    validate(req);                       // 락 없이 실행

    String cached = cache.get(req.key());  // ConcurrentHashMap → 락 없이 읽기
    if (cached != null) return cached;

    String result = externalApi.fetch(req.key());  // 락 없이 외부 호출

    cache.putIfAbsent(req.key(), result);  // 원자적 조건부 삽입
    return result;
}
// 락이 필요한 부분이 없어짐! ConcurrentHashMap이 내부적으로 처리

// 패턴 2: Lock Striping (전체 락 → 분산 락)
// Before: 하나의 Lock으로 전체 보호
Map<String, Data> sharedMap = new HashMap<>();
Object globalLock = new Object();
// synchronized(globalLock) { sharedMap.get/put }

// After: ConcurrentHashMap 또는 stripe 락
ConcurrentHashMap<String, Data> concurrentMap = new ConcurrentHashMap<>();
// 버킷별 독립 락 → 다른 키에 대한 동시 접근 허용

// 패턴 3: 읽기-쓰기 분리 (대부분 읽기인 경우)
ReadWriteLock rwLock = new ReentrantReadWriteLock();
// 읽기: 여러 스레드 동시 가능
rwLock.readLock().lock();
try { return sharedData.read(); } finally { rwLock.readLock().unlock(); }
// 쓰기: 단독 실행
rwLock.writeLock().lock();
try { sharedData.write(value); } finally { rwLock.writeLock().unlock(); }

// 패턴 4: Copy-on-Write (쓰기 드문 경우)
// 쓰기: 복사 후 수정 → volatile 참조 교체
// 읽기: 락 없이 volatile 참조 읽기
```

---

## 🔬 내부 동작 원리

### 1. Lock Contention이 처리량을 저하시키는 원리

```
단일 락에서의 직렬화 효과:

8스레드, 각 요청 100ms (락 내부 10ms + 락 외부 90ms):

락 없는 경우:
  모든 스레드 동시 실행 → 100ms 완료
  처리량: 8 req / 100ms = 80 req/s

synchronized(전체 100ms 포함):
  한 번에 1스레드만 → 직렬화
  처리량: 1 req / 100ms = 10 req/s (8배 저하!)

synchronized(락 내부 10ms만):
  락 외부 90ms는 병렬 → 락 내부만 직렬화
  암달의 법칙: 1 / (0.9 + 0.1/8) ≈ 1/0.9125 ≈ 5.7배 처리량
  → 8스레드인데 5.7배만 빨라짐

암달의 법칙(Amdahl's Law):
  병렬화 가능 비율 = p, 순차 처리 비율 = (1-p)
  최대 가속 = 1 / ((1-p) + p/N)   N=프로세서 수

실무 교훈:
  락 내부 10%만 직렬화되어도 처리량의 이론 상한이 10배로 제한됨
  락 내부를 최대한 짧게 유지하는 것이 핵심

스레드 상태와 Contention:
  락 보유 중: RUNNABLE (실행 중)
  락 대기 중: BLOCKED (CPU 소비 없음)
  → CPU 낮음 + BLOCKED 스레드 많음 = Lock Contention 의심
```

### 2. JFR으로 Lock Contention 진단

```bash
# Step 1: JFR 시작 (운영 환경)
jcmd <pid> JFR.start \
  duration=60s \
  settings=profile \
  filename=/tmp/contention.jfr

# 또는 JVM 시작 시
java -XX:StartFlightRecording=duration=60s,settings=profile,filename=/tmp/contention.jfr \
     -jar myapp.jar

# Step 2: JFR 분석 (jfr CLI)
jfr print --events jdk.JavaMonitorEnter \
          --stack-depth 10 \
          /tmp/contention.jfr | head -200

# Step 3: 상위 경합 모니터 찾기
jfr print --json /tmp/contention.jfr | \
  python3 -c "
import json, sys, collections
data = json.load(sys.stdin)
events = [e for e in data.get('events', []) if e['type'] == 'jdk.JavaMonitorEnter']
# 클래스별 누적 대기 시간
wait_by_class = collections.defaultdict(float)
for e in events:
    cls = e.get('monitorClass', {}).get('name', 'unknown')
    duration = e.get('duration', 0)  # 나노초
    wait_by_class[cls] += duration
# 상위 10개 출력
for cls, total_ns in sorted(wait_by_class.items(), key=lambda x: -x[1])[:10]:
    print(f'{total_ns/1e6:.2f} ms  {cls}')
"

# 예상 출력:
# 15430.25 ms  com.example.UserService
# 8920.10 ms   java.util.HashMap
# 3210.50 ms   com.zaxxer.hikari.pool.HikariPool

# Step 4: JDK Mission Control (GUI)
# jmc -open /tmp/contention.jfr
# → Lock Instances 뷰 → 상위 경합 락 시각화
```

### 3. async-profiler로 Lock Contention Flame Graph

```bash
# async-profiler 다운로드 (https://github.com/async-profiler/async-profiler)

# Step 1: Lock 이벤트 프로파일링 (60초)
./asprof -d 60 -e lock -f /tmp/lock-flame.html <pid>

# 또는 JFR 형식으로 저장
./asprof -d 60 -e lock -o jfr -f /tmp/lock.jfr <pid>

# Step 2: HTML 열어 Flame Graph 해석
# X축: 누적 락 대기 시간 (넓을수록 경합 심함)
# Y축: 호출 스택 (아래가 호출자, 위가 락 보유 메서드)
# 넓은 박스 = 락 대기 핫스팟

# Step 3: 특정 메서드 필터링
./asprof -d 60 -e lock -f /tmp/lock-flame.html \
         --include 'com/example/*' \
         <pid>

# Step 4: Java 레벨 모니터 진입 시간 측정
# JVM 옵션 추가:
# -XX:+UnlockDiagnosticVMOptions -XX:+PrintContendedLocks
# (상세 로그, 운영 환경 주의)
```

### 4. 리팩토링 패턴 상세

```java
// 리팩토링 1: Double-Checked Locking (초기화 패턴)
// Before (전체 메서드 synchronized):
class ExpensiveResource {
    private static ExpensiveResource instance;
    public synchronized static ExpensiveResource getInstance() {
        if (instance == null) {
            instance = new ExpensiveResource();  // 초기화 이후에도 락!
        }
        return instance;
    }
}

// After (Holder 패턴, 락 없음):
class ExpensiveResource {
    private static class Holder {
        static final ExpensiveResource INSTANCE = new ExpensiveResource();
    }
    public static ExpensiveResource getInstance() {
        return Holder.INSTANCE;  // 락 없음, 스레드 안전
    }
}

// 리팩토링 2: 연산 밖으로 이동
// Before:
synchronized (this) {
    String result = expensiveComputation();  // 느린 연산
    list.add(result);
}

// After:
String result = expensiveComputation();  // 락 밖에서 수행
synchronized (this) {
    list.add(result);  // 빠른 상태 업데이트만
}

// 리팩토링 3: Lock Striping 직접 구현
class StripedCache<K, V> {
    private static final int STRIPES = 16;
    private final Map<K, V>[] maps = new HashMap[STRIPES];
    private final Object[] locks = new Object[STRIPES];

    StripedCache() {
        for (int i = 0; i < STRIPES; i++) {
            maps[i] = new HashMap<>();
            locks[i] = new Object();
        }
    }

    int stripe(K key) { return Math.abs(key.hashCode() % STRIPES); }

    V get(K key) {
        int s = stripe(key);
        synchronized (locks[s]) { return maps[s].get(key); }
    }

    void put(K key, V value) {
        int s = stripe(key);
        synchronized (locks[s]) { maps[s].put(key, value); }
    }
}
// → 서로 다른 stripe의 키는 동시 접근 가능
// 대안: ConcurrentHashMap이 이미 버킷 레벨 Lock Striping 제공
```

---

## 💻 실전 실험

### 실험 1: JFR으로 경합 지점 찾기

```java
public class ContentionDemo {
    private static final Map<String, Integer> SHARED_MAP = new HashMap<>();

    // 경합 유발 메서드 (synchronized 범위 과도)
    public synchronized static void processWithContention(String key) {
        try {
            Thread.sleep(1);  // 느린 처리 시뮬레이션
            SHARED_MAP.merge(key, 1, Integer::sum);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) throws Exception {
        // JVM 옵션: -XX:StartFlightRecording=duration=10s,filename=contention.jfr

        Thread[] threads = new Thread[16];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    processWithContention("key-" + (j % 10));
                }
            });
            threads[i].start();
        }

        long start = System.nanoTime();
        for (Thread t : threads) t.join();
        System.out.printf("완료: %.2f ms%n", (System.nanoTime()-start)/1e6);

        // JFR 분석 후 ContentionDemo.processWithContention이
        // 상위 경합 메서드로 나타남
    }
}
```

### 실험 2: synchronized vs ConcurrentHashMap 처리량 비교

```java
import org.openjdk.jmh.annotations.*;
import java.util.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(16)
public class MapContentionBenchmark {

    final HashMap<Integer, Integer> syncMap = new HashMap<>();
    final ConcurrentHashMap<Integer, Integer> concMap = new ConcurrentHashMap<>();

    @Setup public void setup() {
        for (int i = 0; i < 1000; i++) { syncMap.put(i, i); concMap.put(i, i); }
    }

    @Benchmark
    public Integer synchronizedGet() {
        synchronized (syncMap) {
            return syncMap.get((int)(Math.random() * 1000));
        }
    }

    @Benchmark
    public Integer concurrentGet() {
        return concMap.get((int)(Math.random() * 1000));
    }

    @Benchmark
    public void synchronizedPut() {
        synchronized (syncMap) {
            syncMap.put((int)(Math.random() * 1000), 1);
        }
    }

    @Benchmark
    public void concurrentPut() {
        concMap.put((int)(Math.random() * 1000), 1);
    }
}
// 결과: 16스레드에서 concurrent*가 synchronized* 대비 5~10배 높은 처리량
```

### 실험 3: Lock Striping vs 단일 락 비교

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(16)
public class StripingBenchmark {

    // 단일 락 + HashMap
    final Object singleLock = new Object();
    final java.util.HashMap<Integer, Integer> singleMap = new java.util.HashMap<>();

    // 스트라이프 락 (16개)
    static final int STRIPES = 16;
    final Object[] stripeLocks = new Object[STRIPES];
    final java.util.HashMap<Integer, Integer>[] stripeMaps = new java.util.HashMap[STRIPES];

    @Setup public void setup() {
        for (int i = 0; i < STRIPES; i++) {
            stripeLocks[i] = new Object();
            stripeMaps[i] = new java.util.HashMap<>();
        }
    }

    @Benchmark
    public Integer singleLockGet() {
        int k = (int)(Math.random() * 10000);
        synchronized (singleLock) { return singleMap.get(k); }
    }

    @Benchmark
    public Integer stripedLockGet() {
        int k = (int)(Math.random() * 10000);
        int s = Math.abs(k % STRIPES);
        synchronized (stripeLocks[s]) { return stripeMaps[s].get(k); }
    }
}
// 결과: 16스트라이프 vs 단일 락 → ~12~15배 처리량 향상
```

---

## 📊 성능/비용 비교

```
Lock Contention 개선 전후 처리량 비교 (16스레드):

방법                          | 처리량       | 비고
─────────────────────────────┼────────────┼─────────────────────────────
synchronized(전체 메서드)       | 10,000/s   | 완전 직렬화
synchronized(최소 범위)        | 50,000/s   | 5배 향상
Lock Striping(16개)          | 140,000/s  | 14배 향상
ConcurrentHashMap            | 200,000/s  | 20배 향상 (버킷 CAS)
ReadWriteLock(읽기 90%)       | 180,000/s  | 18배 향상

JFR 오버헤드:
  기본 설정 (default): ~1% CPU
  profile 설정:        ~2~3% CPU
  lock 이벤트 추가:    ~3~5% CPU
  → 운영 환경 상시 활성화 가능

async-profiler -e lock 오버헤드:
  ~2~5% CPU (프로파일링 대상 메서드에 비례)
```

---

## ⚖️ 트레이드오프

```
Lock Contention 해결 전략 트레이드오프:

synchronized 범위 축소:
  장점: 단순, 코드 최소 변경
  단점: 범위 밖 연산의 원자성 보장 어려움

Lock Striping:
  장점: 동시성 선형 향상 (스트라이프 수에 비례)
  단점: 구현 복잡성, 균등 분산 필요 (해시 함수)

ConcurrentHashMap:
  장점: Lock Striping 자동 적용, 표준 API
  단점: 복합 연산의 원자성 직접 보장 어려움

ReadWriteLock:
  장점: 읽기 병렬화 (읽기 >> 쓰기일 때 효과적)
  단점: 쓰기 기아 가능, 관리 복잡

Lock-Free (CAS):
  장점: 락 없음 → 스케줄러 영향 없음
  단점: CAS 실패 시 재시도, 고경쟁에서 효율 저하

진단 우선:
  측정 없이 최적화하지 말 것
  JFR으로 실제 경합 지점을 먼저 파악
  가장 큰 경합이 있는 곳 1~2개만 집중 개선
```

---

## 📌 핵심 정리

```
Lock Contention 핵심:

증상:
  CPU 낮음 + BLOCKED 스레드 많음 + 처리량 저하
  응답 시간 높음 + 스레드 덤프에 같은 락 대기자 다수

진단:
  JFR: jdk.JavaMonitorEnter 이벤트 → 경합 지점/누적 시간
  async-profiler: -e lock → Flame Graph
  Thread Dump: BLOCKED 스레드 + 락 보유자 분석

해결:
  synchronized 범위 최소화 (핵심!)
  Lock Striping (여러 독립 락으로 분산)
  ConcurrentHashMap (버킷 레벨 자동 스트라이핑)
  ReadWriteLock (읽기 비율 높을 때)
  Lock-Free (CAS, 단일 변수)

암달의 법칙:
  직렬화 비율이 처리량 상한을 결정
  10%만 직렬화되어도 최대 10배 가속
  → 락 내부를 최대한 짧고 빠르게

워크플로우:
  JFR 수집 → 경합 클래스 파악 → 코드 분석 → 범위 축소
  → 재측정 → 개선 확인
```

---

## 🤔 생각해볼 문제

**Q1.** JFR의 `jdk.JavaMonitorEnter` 이벤트는 언제부터 기록되는가? 모든 `synchronized` 진입이 기록되는가?

<details>
<summary>해설 보기</summary>

`jdk.JavaMonitorEnter` 이벤트는 **경합이 있는 경우에만** 기록된다. 즉, 락이 이미 해제되어 있어 즉시 획득한 경우는 이벤트가 발생하지 않는다.

또한 임계값(threshold)이 있다. JFR의 기본 설정에서는 약 20ms 이상 대기한 경우만 기록한다. `profile` 설정에서는 더 짧은 대기도 기록한다.

커스텀 임계값 설정:
```bash
# 1ms 이상 대기만 기록
jcmd <pid> JFR.start \
  settings=profile \
  filename=/tmp/contention.jfr
  
# JFR 설정 파일로 임계값 조정
# jvm-settings.jfc 파일에서 jdk.JavaMonitorEnter의 threshold 값 변경
```

따라서 짧은 경합(< 20ms)이 빈번하게 발생하는 경우를 파악하려면 `profile` 설정이나 임계값을 낮춰야 한다. async-profiler `-e lock`은 모든 락 대기를 샘플링 방식으로 수집하므로 짧은 경합도 탐지할 수 있다.

</details>

---

**Q2.** `Collections.synchronizedMap(new HashMap<>())`과 `ConcurrentHashMap`은 성능이 다른가? 어떤 상황에서 각각을 선택하는가?

<details>
<summary>해설 보기</summary>

성능 차이가 크다:

`Collections.synchronizedMap(HashMap)`:
- 모든 메서드가 단일 `synchronized (mutex)` 블록
- 읽기와 쓰기가 모두 직렬화됨
- 복합 연산(check-then-act)을 `synchronized` 블록으로 감싸서 원자적으로 처리 가능

`ConcurrentHashMap`:
- 버킷 레벨 `synchronized` + CAS (Java 8+)
- 읽기는 락 없음 (volatile 읽기)
- 서로 다른 버킷의 쓰기는 동시 가능
- 복합 연산: `computeIfAbsent`, `merge`, `compute` 등 원자적 API 제공

선택 기준:
- **다수 스레드 동시 접근**: `ConcurrentHashMap`
- **iterator 사용 중 수정 방지**: `Collections.synchronizedMap` (외부에서 `synchronized` 블록으로 iterator 보호)
- **복합 원자 연산**: `ConcurrentHashMap`의 `compute*` 메서드
- **레거시 코드 호환**: 경우에 따라 `synchronizedMap`이 필요할 수도

16스레드 JMH 기준: `ConcurrentHashMap`의 `get()`이 `synchronizedMap` 대비 **10~20배** 높은 처리량.

</details>

---

**Q3.** Lock Contention 없이도 응답 시간이 높을 수 있다. Lock Contention이 원인인지 다른 문제(GC, CPU 바운드, 네트워크)인지 어떻게 구별하는가?

<details>
<summary>해설 보기</summary>

각 원인별 징표:

**Lock Contention**:
- CPU: 낮음 (10~30%)
- Thread Dump: `BLOCKED` 상태 스레드 다수, 같은 락 보유자/대기자
- JFR: `jdk.JavaMonitorEnter` 누적 시간 높음
- 부하 증가 시 처리량이 선형으로 증가하지 않음

**GC 일시 정지**:
- CPU: 간헐적으로 급등
- JVM 로그: GC 로그에서 STW(Stop-The-World) 시간
- JFR: `jdk.GarbageCollection` 이벤트의 pause duration
- 응답 시간이 주기적으로 급증(GC 주기와 일치)

**CPU 바운드**:
- CPU: 100% 근접
- Thread Dump: 스레드가 `RUNNABLE` 상태로 CPU 연산 중
- async-profiler CPU 프로파일링: 특정 메서드가 CPU 점유

**네트워크/I/O**:
- CPU: 낮음
- Thread Dump: `WAITING on condition` + 소켓/파일 I/O 스택
- 네트워크 모니터링: 높은 RTT 또는 낮은 대역폭

진단 순서:
1. Thread Dump 수집 → BLOCKED 비율 확인
2. JFR 수집 → GC pause, MonitorEnter 상위 5개 확인
3. async-profiler CPU/lock 프로파일링

하나의 문제가 다른 문제를 유발하기도 한다. 예: Lock Contention → 스레드 증가 → 메모리 증가 → GC 압박 → 응답 지연 악화.

</details>

---

<div align="center">

**[⬅️ 이전: 라이브락과 기아](./02-livelock-starvation.md)** | **[홈으로 🏠](../README.md)** | **[다음: 비동기 프로그래밍 비교 ➡️](./04-async-comparison.md)**

</div>
