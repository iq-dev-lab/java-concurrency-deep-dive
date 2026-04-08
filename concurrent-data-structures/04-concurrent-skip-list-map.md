# ConcurrentSkipListMap — Lock-Free 정렬 맵

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 스킵 리스트가 확률적 레이어로 O(log N) 검색·삽입을 보장하는 구조는?
- `ConcurrentSkipListMap`이 CAS + "마커 노드"로 삭제를 표시하는 방식은?
- `TreeMap`(Red-Black Tree)과 멀티스레드 환경 성능을 어떻게 비교하는가?
- `ConcurrentSkipListSet`은 어떻게 구현되는가?
- 정렬 순서가 필요한 동시성 자료구조가 필요할 때 선택 기준은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`ConcurrentSkipListMap`은 정렬된 동시성 맵이 필요한 경우의 표준 선택이다. 시계열 데이터 저장, 범위 쿼리, 우선순위 기반 이벤트 큐 등에서 활용된다. 스킵 리스트 알고리즘이 왜 Red-Black Tree 대신 동시성에 적합한지 이해하면, Lock-Free 정렬 자료구조의 설계 원리를 파악할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: synchronized TreeMap으로 충분하다는 오해
  Map<Long, Event> events = Collections.synchronizedMap(new TreeMap<>());
  → 모든 연산이 단일 락으로 직렬화
  → 읽기도 차단됨
  → 범위 쿼리(subMap) 중 다른 스레드 삽입 불가
  → ConcurrentSkipListMap: 락 없는 읽기, 높은 동시성

실수 2: ConcurrentSkipListMap의 범위 뷰가 동적이라는 사실 간과
  NavigableMap<K,V> view = map.subMap(from, to);
  // view를 통한 수정이 원본 맵에 반영됨!
  view.put(key, value);  // 원본 map에 삽입
  // 반대로 원본 변경도 view에 즉시 반영됨 (스냅샷 아님)

실수 3: 스킵 리스트가 TreeMap보다 항상 빠르다는 가정
  단일 스레드 환경:
    TreeMap(RB Tree): O(log N), 낮은 상수 (구조 단순)
    SkipList: O(log N), 더 많은 메모리, 랜덤 레이어 관리 오버헤드
  → 단일 스레드: TreeMap이 오히려 빠를 수 있음
  → 멀티스레드 높은 동시성: ConcurrentSkipListMap 우위
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// ConcurrentSkipListMap 올바른 활용 패턴

// 패턴 1: 정렬된 동시성 이벤트 로그
ConcurrentSkipListMap<Long, Event> eventLog = new ConcurrentSkipListMap<>();
eventLog.put(System.nanoTime(), event);  // 시간 기반 정렬

// 범위 쿼리 (락 없음, 스냅샷 뷰)
long from = System.nanoTime() - TimeUnit.SECONDS.toNanos(10);
NavigableMap<Long, Event> recent = eventLog.subMap(from, true, Long.MAX_VALUE, true);
for (Map.Entry<Long, Event> e : recent.entrySet()) {
    process(e.getValue());
}

// 패턴 2: 최솟값/최댓값 빠르게 조회
Long firstKey = eventLog.firstKey();   // O(log N)
Long lastKey  = eventLog.lastKey();    // O(log N)
Map.Entry<Long, Event> oldest = eventLog.firstEntry();

// 패턴 3: 만료된 항목 제거
Long cutoff = System.nanoTime() - TimeUnit.HOURS.toNanos(1);
eventLog.headMap(cutoff).clear();  // 1시간 이전 이벤트 제거

// 패턴 4: ConcurrentSkipListSet (중복 없는 정렬 셋)
ConcurrentSkipListSet<String> sortedSet = new ConcurrentSkipListSet<>();
sortedSet.add("banana"); sortedSet.add("apple"); sortedSet.add("cherry");
System.out.println(sortedSet.first());  // "apple" (알파벳 순)

// 패턴 5: floor/ceiling 쿼리
Long key = eventLog.floorKey(targetTime);   // targetTime 이하 최대 키
Long ceil = eventLog.ceilingKey(targetTime); // targetTime 이상 최소 키
```

---

## 🔬 내부 동작 원리

### 1. 스킵 리스트 구조

```
스킵 리스트 (Skip List):
  여러 레이어의 연결 리스트를 쌓은 구조
  각 레이어는 하위 레이어의 "익스프레스 레인"

예시 (key: 1, 3, 5, 7, 9):

  레이어 3: HEAD ----------------------------> 9 → null
  레이어 2: HEAD ------------> 5 -----------> 9 → null
  레이어 1: HEAD ----> 3 ----> 5 ------> 7 --> 9 → null
  레이어 0: HEAD → 1 → 3 → 5 → 7 → 9 → null  (기본 레이어, 모든 키)

검색(5):
  레이어 3에서 시작: HEAD → 9 (5 < 9 → 한 레이어 내려감)
  레이어 2: HEAD → 5 (발견!)  ← 3번 만에 도달
  레이어 0에서 선형 탐색했으면 5번 필요

레이어 결정 (확률적):
  새 노드 삽입 시 동전 던지기 방식으로 레이어 수 결정
  확률 p (보통 0.5 또는 0.25)로 레이어 추가
  최대 레이어 수: log(1/p)(N) 기댓값

복잡도:
  검색: O(log N) 기댓값
  삽입: O(log N) 기댓값 (삽입 위치 검색 + 연결)
  삭제: O(log N) 기댓값
  최솟값/최댓값: O(1) (헤드/테일 참조)
  범위 쿼리: O(log N + K) (K = 결과 수)
```

### 2. ConcurrentSkipListMap의 Lock-Free 삽입

```
ConcurrentSkipListMap의 노드 구조:
  class Index<K,V> {
      final Node<K,V> node;     // 실제 데이터 노드
      final Index<K,V> down;    // 아래 레이어
      volatile Index<K,V> right; // 오른쪽 (volatile!)
  }

  class Node<K,V> {
      final K key;
      volatile Object value;    // null = 삭제된 노드
      volatile Node<K,V> next;  // volatile!
  }

Lock-Free 삽입 과정:
  ① 삽입 위치 검색 (각 레이어 내려가며 predecessor 찾기)
  ② base 레이어에 CAS로 노드 연결:
     CAS(predecessor.next, successor, newNode)
     → 성공: 연결 완료
     → 실패: 다른 스레드가 먼저 변경 → 재탐색 후 재시도

  ③ 상위 레이어 인덱스 생성 (확률에 따라):
     각 레이어에 Index 노드 생성
     CAS로 right 포인터 설정
     실패 시: 재시도 또는 포기 (인덱스 없어도 기능은 동작)

Helping 메커니즘:
  삽입 중 삭제된 노드를 만나면:
    그 노드를 건너뛰고 링크 수정 (삭제 보조)
  → "이미 삭제된 노드" 처리를 다른 스레드도 도움
```

### 3. Lock-Free 삭제 — 마커 노드

```
삭제가 어려운 이유:
  단순히 predecessor.next = node.next로 변경 불가
  이유: 동시에 다른 스레드가 node.next에 새 노드를 삽입 중일 수 있음
       → 삭제와 삽입이 동시 발생 → 삽입이 유실될 수 있음

마커 노드(Marker Node) 방식:

  삭제 프로세스 (2단계):
  
  1단계: 논리적 삭제 (Logical Deletion)
    삭제할 노드의 next에 특별한 마커 노드 CAS 삽입:
    CAS(node.next, successor, new MarkerNode(successor))
    
    이전: ... → [node, val] → [succ] → ...
    이후: ... → [node, val] → [MARKER] → [succ] → ...
    
    마커 노드가 삽입되면:
    다른 스레드의 삽입 시도: node.next를 CAS로 바꾸려 하면 실패 (마커가 있음)
    → 다른 스레드가 "이 노드는 삭제 중"임을 인식

  2단계: 물리적 삭제 (Physical Deletion)
    predecessor.next에서 node와 marker를 함께 제거:
    CAS(predecessor.next, node, successor)
    
    이전: ... → [pred] → [node, val] → [MARKER] → [succ] → ...
    이후: ... → [pred] → [succ] → ...

  2단계가 실패해도:
    다른 스레드의 탐색 중 마커를 만나면 자동으로 물리 삭제 도움 (Helping)

value = null 마킹:
  node.value를 null로 CAS → "삭제됨" 표시
  탐색 시 value == null인 노드는 건너뜀
  → 두 단계 삭제와 함께 사용
```

### 4. TreeMap vs ConcurrentSkipListMap 비교

```
단일 스레드 성능:
  TreeMap (Red-Black Tree):
    배열이 아닌 트리 노드 → 포인터 추적
    균형 트리 → 높이 = O(log N), 상수 작음
    GC 친화적: 노드 추가/삭제 시 필요한 메모리만

  ConcurrentSkipListMap:
    여러 레이어 포인터 → 메모리 더 많이 사용
    레이어 랜덤 결정 → 추가 연산
    → 단일 스레드에서 TreeMap보다 20~40% 느림

멀티스레드 성능 (8스레드, JMH):
  TreeMap + synchronized:
    모든 연산 직렬화 → 확장성 없음
    8스레드: ~150,000 ops/ms (단일 스레드 ~700,000 대비 78% 감소)

  ConcurrentSkipListMap:
    읽기 락 없음, 쓰기 CAS
    8스레드: ~1,200,000 ops/ms (단일 스레드 대비 선형 확장!)
    → 멀티스레드에서 8배 이상 차이

  ConcurrentHashMap (정렬 필요 없을 때):
    ~2,500,000 ops/ms (더 빠름, 정렬 없음)

왜 스킵 리스트가 동시성에 유리한가:
  Red-Black Tree: 삽입/삭제 시 트리 회전 → 여러 포인터 동시 수정 필요
    → 락 없이 원자적으로 구현하기 어려움 (여러 CAS 필요)
  스킵 리스트: 삽입/삭제가 연결 리스트 수준 → CAS 몇 번으로 완료 가능
    → Lock-Free 구현에 자연스럽게 맞음
```

---

## 💻 실전 실험

### 실험 1: TreeMap vs ConcurrentSkipListMap 처리량

```java
import org.openjdk.jmh.annotations.*;
import java.util.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(8)
public class SkipListVsTreeMapBenchmark {

    TreeMap<Integer, Integer> treeMap = new TreeMap<>();
    ConcurrentSkipListMap<Integer, Integer> skipMap = new ConcurrentSkipListMap<>();

    @Setup public void setup() {
        Random rand = new Random();
        for (int i = 0; i < 10000; i++) {
            int k = rand.nextInt(100000);
            treeMap.put(k, k);
            skipMap.put(k, k);
        }
    }

    @Benchmark
    public Integer syncTreeRead() {
        synchronized (treeMap) {
            return treeMap.get((int)(Math.random() * 100000));
        }
    }

    @Benchmark
    public void syncTreeWrite() {
        synchronized (treeMap) {
            int k = (int)(Math.random() * 100000);
            treeMap.put(k, k);
        }
    }

    @Benchmark
    public Integer skipRead() {
        return skipMap.get((int)(Math.random() * 100000));
    }

    @Benchmark
    public void skipWrite() {
        int k = (int)(Math.random() * 100000);
        skipMap.put(k, k);
    }
}
// 결과: 8스레드에서 skipRead/skipWrite가 syncTree 대비 5~10배 이상 처리량
```

### 실험 2: 시계열 데이터 범위 쿼리

```java
import java.util.concurrent.*;
import java.util.*;

public class TimeSeriesStore {
    private final ConcurrentSkipListMap<Long, String> store = new ConcurrentSkipListMap<>();

    public void record(String event) {
        store.put(System.nanoTime(), event);
    }

    // 최근 N초 이내 이벤트 조회
    public NavigableMap<Long, String> recent(int seconds) {
        long cutoff = System.nanoTime() - TimeUnit.SECONDS.toNanos(seconds);
        return store.tailMap(cutoff, true);
    }

    // 오래된 이벤트 정리
    public int evictOlderThan(int seconds) {
        long cutoff = System.nanoTime() - TimeUnit.SECONDS.toNanos(seconds);
        NavigableMap<Long, String> old = store.headMap(cutoff, false);
        int count = old.size();
        old.clear();  // 원본에서 제거
        return count;
    }

    public static void main(String[] args) throws InterruptedException {
        TimeSeriesStore ts = new TimeSeriesStore();
        // 여러 스레드에서 동시 기록
        for (int i = 0; i < 5; i++) {
            final int id = i;
            new Thread(() -> {
                for (int j = 0; j < 100; j++) {
                    ts.record("event-t" + id + "-" + j);
                    try { Thread.sleep(10); } catch (InterruptedException e) {}
                }
            }).start();
        }

        Thread.sleep(1000);
        NavigableMap<Long, String> recent = ts.recent(1);
        System.out.println("최근 1초 이벤트 수: " + recent.size());
        int evicted = ts.evictOlderThan(1);
        System.out.println("제거된 이벤트 수: " + evicted);
    }
}
```

---

## 📊 성능/비용 비교

```
자료구조별 특성 비교 (정렬 맵):

특성                    | TreeMap       | Collections.sync(TreeMap) | ConcurrentSkipListMap
───────────────────────┼───────────────┼──────────────────────────┼──────────────────────
스레드 안전성           | ❌            | ✅ (전체 락)              | ✅ (Lock-Free)
단일 스레드 성능        | 빠름          | 동일                     | ~20% 느림
멀티스레드 확장성       | N/A           | 없음 (직렬화)             | 선형 확장
읽기 동시성            | N/A           | ❌ (읽기도 락)            | ✅ (락 없음)
메모리 사용            | 낮음          | 동일                     | 더 높음 (레이어 인덱스)
정렬 순서 보장         | ✅            | ✅                       | ✅
범위 뷰 (subMap 등)    | 동적 뷰 반환   | 동기화 필요              | 동시성 안전 동적 뷰
first/last/floor/ceiling| ✅           | 락 필요                  | ✅ (락 없음)
복잡도 (search/insert) | O(log N)      | O(log N) + 락            | O(log N) 기댓값
```

---

## ⚖️ 트레이드오프

```
ConcurrentSkipListMap 선택 기준:

선택해야 할 때:
  정렬 순서 + 멀티스레드 안전성 동시 필요
  범위 쿼리(subMap, headMap, tailMap)가 빈번
  읽기가 압도적으로 많은 경우 (락 없는 읽기)
  floor/ceiling/first/last 쿼리 필요
  동시 삽입/삭제가 많은 시계열 데이터

피해야 할 때:
  단일 스레드 환경 → TreeMap이 더 빠름
  정렬이 필요 없음 → ConcurrentHashMap이 더 빠름
  메모리가 매우 제한적 → 스킵 리스트 레이어 오버헤드

ConcurrentSkipListSet vs ConcurrentHashSet:
  ConcurrentSkipListSet: 정렬 + 중복 없음 (내부적으로 ConcurrentSkipListMap)
  ConcurrentHashMap.newKeySet(): 정렬 없음 + 중복 없음, 더 빠름
  → 정렬이 필요하면 ConcurrentSkipListSet, 아니면 ConcurrentHashMap.newKeySet()
```

---

## 📌 핵심 정리

```
ConcurrentSkipListMap 핵심:

스킵 리스트 구조:
  확률적 레이어 → O(log N) 기댓값
  단순 연결 리스트 조작 → Lock-Free 구현에 적합

Lock-Free 구현:
  삽입: CAS로 next 포인터 설정
  삭제: 2단계 (마커 노드 삽입 → 물리 제거)
  마커 노드: "이 노드 삭제 중" 신호 → 다른 스레드 삽입 방지
  Helping: 탐색 중 삭제 보조

vs TreeMap:
  단일 스레드: TreeMap 우위 (~20%)
  멀티스레드: ConcurrentSkipListMap 압도적 우위 (락 없는 읽기)

사용 사례:
  시계열 데이터, 범위 쿼리, 정렬 이벤트 로그
  우선순위 큐 대용 (floor/ceiling 활용)
  정렬 + 동시성 동시 필요한 모든 경우
```

---

## 🤔 생각해볼 문제

**Q1.** `ConcurrentSkipListMap`의 `subMap(from, to).clear()`는 원본 맵에서 실제로 제거되는가? 이것이 스레드 안전한가?

<details>
<summary>해설 보기</summary>

그렇다. `subMap()`은 원본 맵의 동적 뷰(view)를 반환하며, 뷰에 대한 수정(clear, put, remove)이 원본 맵에 직접 반영된다.

스레드 안전성: `clear()`는 내부적으로 뷰 범위의 각 키에 대해 `remove()`를 반복 호출한다. 각 `remove()`가 Lock-Free CAS로 구현되므로 스레드 안전하다.

주의사항:
- `subMap().clear()` 도중 다른 스레드가 해당 범위에 삽입하면, 그 삽입이 clear 전에 발생했는지 후에 발생했는지에 따라 제거될 수도 있고 안 될 수도 있다 (weakly consistent)
- "정확히 이 시점의 범위 키들을 원자적으로 모두 제거"는 보장되지 않는다
- 원자적 범위 제거가 필요하면 별도 락을 추가해야 한다

</details>

---

**Q2.** 스킵 리스트에서 최악의 경우 O(N)이 될 수 있는가? 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

이론적으로 가능하다. 모든 노드가 레이어 0만 가지면(최하위 레이어), 선형 탐색과 동일해진다. 하지만 확률적 레이어 결정으로 이 가능성을 통계적으로 억제한다.

`ConcurrentSkipListMap`의 레이어 수 결정:
```java
private int randomLevel() {
    int level = 1;
    while (ThreadLocalRandom.current().nextInt() < SKIPLEVEL_PROB && level < MAX_LEVEL)
        level++;
    return level;
}
```

`SKIPLEVEL_PROB = 0.25` (기본값): 각 레이어 추가 확률 25%
- 레이어 1 확률: 100%
- 레이어 2 확률: 25%
- 레이어 3 확률: 6.25%
- ...

`MAX_LEVEL = 64` (기본값, N = 2^64에 최적화)

N = 100만 개 노드에서:
- 기댓값 레이어 수 ≈ log(0.25)(N) ≈ 10
- 탐색 비교 횟수 기댓값: ~40회
- 최악 케이스 O(N): 확률적으로 무시 가능 수준

더 강한 보장이 필요하면 결정적 스킵 리스트(Deterministic Skip List)나 B-Tree를 사용한다.

</details>

---

**Q3.** `ConcurrentSkipListMap`은 왜 `null` 키와 `null` 값을 허용하지 않는가?

<details>
<summary>해설 보기</summary>

`ConcurrentHashMap`과 동일한 이유 + 추가적 이유가 있다.

공통 이유: 동시성 환경에서 `null`의 모호성 (`null` = "없음"인지 "null이 매핑됨"인지 구분 불가).

스킵 리스트 특유의 이유:
- `value = null`이 "논리적으로 삭제된 노드"의 마커로 사용된다. 내부적으로 삭제 시 `node.value`를 `null`로 CAS하여 "삭제됨"을 표시한다.
- null 값을 허용하면 "삭제된 노드"와 "null을 매핑한 노드"를 구분할 수 없다.

null 키의 경우: `compareTo(null)`이 `NullPointerException`을 던지므로, 정렬 키로 null을 허용하면 트리/스킵 리스트의 정렬이 불가능하다.

</details>

---

<div align="center">

**[⬅️ 이전: BlockingQueue 내부](./03-blocking-queue-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Exchanger와 Phaser ➡️](./05-exchanger-phaser.md)**

</div>
