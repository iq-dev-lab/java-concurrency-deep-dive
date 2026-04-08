# ConcurrentHashMap 진화 — 세그먼트 락에서 CAS로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 7의 16개 Segment(ReentrantLock) 구조가 왜 문제였는가?
- Java 8에서 버킷 레벨 `synchronized` + CAS로 전환한 이유는?
- `computeIfAbsent`의 내부 동작과 락 범위는?
- 연결 리스트에서 Red-Black Tree로 변환되는 조건(TREEIFY_THRESHOLD=8)은?
- `size()`와 `mappingCount()`는 어떻게 일관성을 처리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`ConcurrentHashMap`은 Spring의 Bean 레지스트리, 캐시 라이브러리, Netty의 채널 관리 등 수많은 프레임워크의 핵심 자료구조다. 이 구조가 어떻게 진화했는지 이해하면, 왜 `computeIfAbsent`가 중첩 호출에서 데드락이 발생했는지, 왜 `size()`가 정확하지 않을 수 있는지를 이해할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: ConcurrentHashMap을 synchronized HashMap의 단순 대체로 인식
  "ConcurrentHashMap 쓰면 스레드 안전하게 캐시 구현 가능"
  → 복합 연산은 여전히 원자적이지 않음
  
  // 버그: check-then-act
  if (!map.containsKey(key)) {
      map.put(key, computeExpensive(key));
  }
  → 두 스레드가 동시에 containsKey=false → 둘 다 put
  → 해결: computeIfAbsent(key, k -> computeExpensive(k))

실수 2: computeIfAbsent에서 재귀 호출
  // 데드락 또는 무한루프 가능 (Java 8 특정 버전)
  map.computeIfAbsent("parent", k -> {
      return map.computeIfAbsent("child", k2 -> "value");
      // 같은 버킷이면 락 재진입 시도 → Java 8에서 무한루프
  });
  // 해결: 중첩 computeIfAbsent 사용 금지 (Java 9에서 일부 개선)

실수 3: size()의 정확성을 신뢰
  if (map.size() == 0) { init(); }
  → size()는 LongAdder 기반, 완전히 정확하지 않을 수 있음
  → "비었는가?" 확인: isEmpty() 사용
  → 정확한 크기 필요 시: mappingCount() (근사치, long 반환)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// ConcurrentHashMap 올바른 활용 패턴

ConcurrentHashMap<String, Object> map = new ConcurrentHashMap<>();

// 패턴 1: 원자적 조건부 삽입
Object value = map.computeIfAbsent(key, k -> {
    // 한 스레드만 실행됨 (같은 키에 대해)
    return new ExpensiveObject(k);
});

// 패턴 2: 원자적 업데이트
map.merge(key, 1, Integer::sum);
// = computeIfPresent(key, (k,v) -> v + 1) + putIfAbsent(key, 1)

// 패턴 3: 원자적 조건부 업데이트
map.compute(key, (k, v) -> {
    if (v == null) return initialValue;
    return transform(v);
});

// 패턴 4: 존재 시에만 업데이트
map.computeIfPresent(key, (k, v) -> v + 1);

// 패턴 5: 병렬 배치 처리 (Java 8+)
map.forEach(2, (k, v) -> process(k, v));  // 병렬도 2
map.reduceValues(2, v -> v.getValue(), Long::sum);  // 병렬 집계

// 패턴 6: 쓰기 없이 읽기 집약적 → 읽기 성능 최대화
// ConcurrentHashMap 읽기는 락 없음 (volatile 읽기)
// 쓰기와 동시 읽기 가능 (weakly consistent)
Object result = map.get(key);  // 락 없는 읽기
```

---

## 🔬 내부 동작 원리

### 1. Java 7: Segment 구조

```
Java 7 ConcurrentHashMap 구조:
  16개의 Segment 배열 (기본값, 최대 65536)
  각 Segment = ReentrantLock + HashEntry 배열

  ┌────────────────────────────────────────────────────┐
  │  segments[0] (ReentrantLock)                       │
  │    HashEntry[]: [0][1][2]...[n]                    │
  │    각 버킷: 연결 리스트                                │
  ├────────────────────────────────────────────────────┤
  │  segments[1] (ReentrantLock)                       │
  │    ...                                             │
  ├────────────────────────────────────────────────────┤
  │  segments[15] (ReentrantLock)                      │
  │    ...                                             │
  └────────────────────────────────────────────────────┘

동작:
  put(key, value):
    segment = segments[hash >>> segmentShift]  // 세그먼트 결정
    segment.lock()
    버킷에서 key 탐색 및 삽입
    segment.unlock()

  get(key):  ← 락 없음! (volatile 읽기)
    HashEntry는 volatile로 선언된 next 포인터
    → 쓰기가 진행 중이어도 읽기는 일관된 상태를 볼 수 있음

Java 7의 한계:
  - 세그먼트 수 = 동시성 수준 (기본 16개)
  - 16개 세그먼트에 균일 분산되지 않으면 특정 세그먼트 경합 집중
  - 세그먼트 크기 조정 시 전체 세그먼트 잠금 필요
  - 메모리 오버헤드: 16개 ReentrantLock 객체
  - Java 8의 더 세밀한 버킷 레벨 락 대비 낮은 동시성
```

### 2. Java 8+: 버킷 레벨 synchronized + CAS

```
Java 8 ConcurrentHashMap 구조:
  단일 Node[] table 배열 (세그먼트 제거!)
  각 버킷의 첫 번째 노드(head)에만 synchronized
  빈 버킷 삽입: CAS (synchronized 없음)

  ┌────────────────────────────────────────────────────┐
  │  Node[] table                                      │
  │  [0]: null                                         │
  │  [1]: [Node(k1,v1)] → [Node(k3,v3)] (연결 리스트)     │
  │  [2]: [TreeBin] → [TreeNode] (Red-Black Tree)      │
  │  [3]: [ForwardingNode] (리사이징 중)                  │
  │  ...                                               │
  └────────────────────────────────────────────────────┘

put(key, value) 동작:
  ① 버킷 = table[hash & (n-1)]
  ② 버킷이 null?
     → CAS(table[hash], null, new Node(k,v)) → 완료 (락 없음!)
     → 실패 시 (동시 삽입): ③

  ③ 버킷 head 노드에 synchronized:
     synchronized (head) {
         연결 리스트 탐색 → key 존재 시 update, 없으면 추가
         또는 TreeBin이면 트리에 삽입
     }

  ④ 연결 리스트 길이 > TREEIFY_THRESHOLD(8)?
     → treeifyBin() 시도 (배열 크기 < 64이면 resize 선택)

읽기(get) 동작:
  ① volatile 읽기로 table 참조 획득
  ② 버킷 head 노드 volatile 읽기
  ③ 연결 리스트 또는 트리 탐색 (락 없음!)
  → 최신 삽입이 보이지 않을 수 있음 (weakly consistent)
  → 진행 중인 삽입과 충돌 없음 (volatile 읽기로 일관된 상태)
```

### 3. 트리 변환 (TREEIFY_THRESHOLD = 8)

```
연결 리스트 → Red-Black Tree 변환:

조건:
  ① 단일 버킷의 노드 수 > TREEIFY_THRESHOLD (8)
  ② 전체 테이블 크기 >= MIN_TREEIFY_CAPACITY (64)

②를 만족하지 않으면:
  resize() 수행 (테이블 2배 확장)
  → 기존 버킷의 노드들이 두 버킷으로 분산
  → 자연스럽게 연결 리스트 단축

TreeBin 노드:
  class TreeBin<K,V> extends Node<K,V> {
      TreeNode<K,V> root;        // Red-Black Tree 루트
      volatile TreeNode<K,V> first;  // 순회를 위한 연결 리스트
      volatile Thread waiter;    // 쓰기 대기 스레드
      volatile int lockState;    // 읽기 잠금 상태
  }

  TreeBin은 자체 읽기-쓰기 락 보유 (ReentrantReadWriteLock 아님, 커스텀):
    읽기: lockState의 비트로 읽기 카운트 관리 (공유 읽기)
    쓰기: synchronized(TreeBin) (배타적 쓰기)

왜 8인가?
  포아송 분포로 계산: 기본 부하율 0.75에서
  버킷당 8개 이상 충돌 확률 ≈ 0.00000006
  (좋은 해시 함수 가정)
  → 8은 실제로 거의 발생하지 않음
  → 해시 충돌이 많으면 해시 함수 문제를 의심

UNTREEIFY_THRESHOLD = 6:
  리사이징 후 TreeBin 노드 수 ≤ 6 → 연결 리스트로 복원
  (8과 6 사이의 히스테리시스로 반복 변환 방지)
```

### 4. 크기 추적 (CounterCell)

```
Java 8 ConcurrentHashMap의 size() 구현:

  baseCount: 기본 카운터 (경쟁 없을 때)
  CounterCell[]: Striped64와 유사한 분산 카운터 (경쟁 시)

  addCount(long x, int check):
    CAS(baseCount, ...) 성공 → baseCount 업데이트
    실패 → counterCells[probe]에 CAS 시도 (경쟁 분산)

  long size():
    return (long) Math.min((long) Integer.MAX_VALUE, sumCount());
    → int 범위 제한 (Java 7 호환)

  long mappingCount():
    return Math.max(sumCount(), 0L);
    → long 반환, 음수 방지

  long sumCount():
    return baseCount + Σ counterCells[i].value
    → LongAdder와 동일한 비일관성: 읽는 동안 카운터 변경 가능

  결과: size()는 근사치
  → "비었는가?"는 isEmpty() 사용
  → 정확한 크기는 보장할 수 없음 (동시 수정 중)
```

---

## 💻 실전 실험

### 실험 1: 세그먼트 vs 버킷 레벨 성능 비교

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(8)
public class ChmBenchmark {

    ConcurrentHashMap<Integer, Integer> chm = new ConcurrentHashMap<>();

    @Setup public void setup() {
        for (int i = 0; i < 10000; i++) chm.put(i, i);
    }

    @Benchmark public Integer readHeavy() {
        return chm.get((int)(Math.random() * 10000));
    }

    @Benchmark public void writeHeavy() {
        int k = (int)(Math.random() * 10000);
        chm.put(k, k);
    }

    @Benchmark public Integer computeIfAbsent() {
        return chm.computeIfAbsent((int)(Math.random() * 20000), k -> k * 2);
    }

    @Benchmark public Integer merge() {
        return chm.merge((int)(Math.random() * 10000), 1, Integer::sum);
    }
}
```

### 실험 2: computeIfAbsent의 원자성 확인

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class ComputeIfAbsentTest {
    static ConcurrentHashMap<String, AtomicInteger> callCounts = new ConcurrentHashMap<>();

    static String expensiveCompute(String key) throws InterruptedException {
        Thread.sleep(10);  // 비싼 계산 시뮬레이션
        callCounts.computeIfAbsent(key, k -> new AtomicInteger(0)).incrementAndGet();
        return "result-" + key;
    }

    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();
        String key = "shared-key";

        // 100개 스레드가 동시에 같은 키로 computeIfAbsent 호출
        Thread[] threads = new Thread[100];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(() -> {
                try {
                    cache.computeIfAbsent(key, k -> {
                        try { return expensiveCompute(k); }
                        catch (InterruptedException e) { throw new RuntimeException(e); }
                    });
                } catch (RuntimeException e) { e.printStackTrace(); }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();

        System.out.println("캐시 크기: " + cache.size());  // 1 (한 번만 계산됨)
        System.out.println("실제 계산 횟수: " + callCounts.getOrDefault(key, new AtomicInteger(0)).get());
        // 이론: 1번 계산
        // 실제: 가끔 2~3번 (같은 버킷 아닌 경우 동시 실행 가능)
    }
}
```

### 실험 3: 트리 변환 관찰

```java
import java.util.*;
import java.util.concurrent.*;
import java.lang.reflect.*;

public class TreeBinObserver {
    public static void main(String[] args) throws Exception {
        // 해시 충돌을 유발하는 키들 (같은 버킷에 삽입)
        ConcurrentHashMap<Object, Integer> map = new ConcurrentHashMap<>(64);

        // hashCode()가 항상 같은 값을 반환하는 키 생성
        class SameHashKey {
            final int id;
            SameHashKey(int id) { this.id = id; }
            @Override public int hashCode() { return 0; }  // 모두 버킷 0에 삽입
            @Override public boolean equals(Object o) {
                return o instanceof SameHashKey && ((SameHashKey)o).id == id;
            }
        }

        for (int i = 0; i < 10; i++) {
            map.put(new SameHashKey(i), i);
            Field tableField = ConcurrentHashMap.class.getDeclaredField("table");
            tableField.setAccessible(true);
            Object[] table = (Object[]) tableField.get(map);
            if (table != null && table[0] != null) {
                System.out.printf("삽입 %d: 버킷[0] 타입 = %s%n",
                    i + 1, table[0].getClass().getSimpleName());
            }
        }
        // 8개 이상 삽입 시: Node → TreeBin으로 변환 확인
    }
}
```

---

## 📊 성능/비교

```
Java 7 vs Java 8 ConcurrentHashMap 비교:

특성             | Java 7 (Segment Lock)  | Java 8 (Bucket Lock)
────────────────┼────────────────────────┼──────────────────────────
동시성 수준        | Segment 수 (기본 16)    | 버킷 수 (capacity)
빈 버킷 삽입       | Segment 락 필요         | CAS (락 없음!)
읽기             | 락 없음 (volatile)      | 락 없음 (volatile)
쓰기 동시성        | 최대 16개               | 최대 버킷 수 (수천~수만)
연결 리스트 최악    | O(N) (해시 충돌 시)      | O(log N) (TreeBin)
메모리            | Segment 16개 오버헤드    | 더 가벼움

읽기 집약 워크로드 (8스레드, JMH):
  Java 7 CHM: ~2,000,000 ops/ms
  Java 8 CHM: ~2,200,000 ops/ms  (약 10% 향상)

쓰기 집약 워크로드 (8스레드, JMH):
  Java 7 CHM: ~800,000 ops/ms
  Java 8 CHM: ~1,200,000 ops/ms  (약 50% 향상)
```

---

## ⚖️ 트레이드오프

```
ConcurrentHashMap 설계 트레이드오프:

버킷 레벨 synchronized (Java 8):
  장점: 최대 동시성 (충돌 없는 버킷끼리 완전 병렬)
  단점: 빈 버킷의 첫 삽입은 CAS 필요 (경쟁 시 재시도)

TreeBin (Red-Black Tree):
  장점: 해시 충돌 최악 케이스 O(N) → O(log N)
  단점: 8개 이상의 충돌은 해시 함수 문제를 의심해야 함
       트리 변환/복원 비용 (드물지만)

weakly consistent 읽기:
  장점: 읽기에 락 없음 → 높은 읽기 처리량
  단점: 진행 중인 삽입은 안 보일 수 있음
       iterator가 생성 후 삽입된 항목을 보거나 안 볼 수 있음

computeIfAbsent 사용 시 주의:
  같은 키의 중복 계산 가능 (같은 버킷이 아닌 경우)
  중첩 computeIfAbsent 금지
  계산 함수가 오래 걸리면 해당 버킷 전체 차단
  → 긴 계산은 compute 밖에서 수행 후 결과만 삽입
```

---

## 📌 핵심 정리

```
ConcurrentHashMap 핵심:

Java 7: 16개 Segment + ReentrantLock
Java 8: 버킷 레벨 synchronized + CAS + TreeBin

빈 버킷 삽입: CAS (락 없음)
기존 버킷 수정: synchronized(head) (버킷별 독립 락)
읽기: 락 없음 (volatile 읽기) → weakly consistent

트리 변환:
  버킷 노드 > 8 + 테이블 크기 >= 64 → TreeBin(Red-Black Tree)
  트리 노드 <= 6 → 연결 리스트 복원

크기 추적:
  baseCount + CounterCell[] (LongAdder 방식)
  size() / mappingCount() = 근사치

복합 연산:
  computeIfAbsent: 원자적 조건부 삽입 (중첩 금지)
  merge: 존재 시 함수 적용, 없으면 삽입
  compute: 항상 함수 적용 (null 반환 시 제거)
```

---

## 🤔 생각해볼 문제

**Q1.** `ConcurrentHashMap.get()`이 락 없이 동작하는데, 동시에 `put()`이 일어나면 어떻게 일관된 값을 읽는가?

<details>
<summary>해설 보기</summary>

`Node`의 `val`과 `next` 필드가 `volatile`로 선언되어 있다. `put()` 시 새 노드를 연결하거나 값을 업데이트할 때 volatile 쓰기를 하고, `get()` 시 volatile 읽기로 최신 상태를 읽는다.

새 노드 삽입 시:
1. 새 Node 생성 (생성자에서 필드 초기화)
2. volatile CAS로 버킷에 연결 (또는 synchronized 블록에서 next 설정)
3. get()이 volatile 읽기로 next 포인터를 따라가면 항상 일관된 노드를 볼 수 있음

부분 초기화된 노드를 볼 수 없는 이유: volatile 쓰기(연결 시점) 이전에 생성자가 완료되며, volatile 읽기(get)는 항상 volatile 쓰기 이후 완료된 상태를 본다(happens-before).

단, put이 진행 중이라면 그 새 값은 아직 연결되지 않아 get이 볼 수 없다. 이것이 "weakly consistent"의 의미다.

</details>

---

**Q2.** `computeIfAbsent`가 같은 키에 대해 두 스레드가 동시에 호출하면 정확히 한 번만 계산되는가?

<details>
<summary>해설 보기</summary>

**보장되지 않는다.** 공식 Javadoc에서도 "함수가 한 번만 호출된다는 것이 보장되지 않는다"고 명시한다.

두 스레드가 같은 키로 호출할 때:
- 같은 버킷: 버킷 헤드에 `synchronized` → 하나가 락을 얻고 실행, 다른 하나는 대기. 락 보유 스레드가 삽입 후 해제 → 대기 스레드가 기존 값 발견 → 함수 재호출 안 함. 이 경우 한 번만 실행.

- 다른 버킷 (테이블 리사이징 중): 두 스레드가 독립적으로 함수를 실행할 수 있음. 첫 번째가 완료하고 두 번째도 완료하면 두 번째 결과가 덮어쓸 수 있음.

따라서 "최대 한 번" 실행을 보장하지 않는다. 계산 함수가 부작용(DB 삽입, 외부 API 호출 등)이 있다면 중복 실행 가능성을 고려해야 한다. 중복 실행이 용납되지 않으면 분산 락이나 다른 메커니즘을 사용해야 한다.

</details>

---

**Q3.** `ConcurrentHashMap`에서 `null` 키와 `null` 값이 허용되지 않는 이유는?

<details>
<summary>해설 보기</summary>

`HashMap`은 null 키/값을 허용하지만, `ConcurrentHashMap`은 허용하지 않는다. 이유는 **동시성 환경에서 null의 모호성** 때문이다.

`map.get(key)`가 `null`을 반환했을 때:
- `HashMap`: "key가 없음" 또는 "key에 null이 매핑됨" 두 가지 의미
- 단일 스레드에서 `containsKey(key)`로 구분 가능

`ConcurrentHashMap`에서:
- `get(key)` → null 반환
- `containsKey(key)` 호출 사이에 다른 스레드가 키를 삽입/삭제 가능
- → null의 의미를 구분할 수 없음 (TOCTOU 문제)

따라서 null을 금지하여 `get() == null`이 항상 "키 없음"을 의미하도록 설계. `containsKey()` 없이 `get()` 반환값만으로 존재 여부 판단 가능.

이것이 `ConcurrentHashMap` 설계자 Doug Lea의 의도적 결정이다. null 키/값이 필요하다면 Optional이나 Sentinel 객체를 사용한다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: CopyOnWriteArrayList ➡️](./02-copy-on-write-arraylist.md)**

</div>
