# CopyOnWriteArrayList — 쓰기 복사와 읽기 안전성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 쓰기마다 배열 전체를 복사하는 이유는 무엇인가?
- 읽기 스레드가 락 없이 안전한 이유는?
- `iterator()`가 생성 시점의 스냅샷을 유지하는 원리는?
- 쓰기 비용(O(N) 복사)이 감당 가능한 사용 사례와 부적합한 사례는?
- `CopyOnWriteArraySet`은 내부적으로 어떻게 구현되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring의 이벤트 리스너 목록, Netty의 핸들러 파이프라인 같이 "읽기가 압도적으로 많고 쓰기가 드문" 컬렉션에 `CopyOnWriteArrayList`가 널리 쓰인다. 이 자료구조의 트레이드오프를 이해하지 못하면, 쓰기가 빈번한 곳에 잘못 사용해 성능 문제를 만들거나, 적합한 상황에서 사용을 꺼리게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 쓰기가 빈번한 컬렉션에 사용
  // 카운터나 자주 변경되는 상태에 사용
  CopyOnWriteArrayList<Event> events = new CopyOnWriteArrayList<>();
  // 초당 10,000번 add() 호출
  // → 매번 전체 배열 복사 → O(N × 쓰기 수) = 심각한 성능 문제

실수 2: CopyOnWriteArrayList의 iterator로 요소 제거 시도
  for (Listener l : listeners) {
      l.onEvent(event);
      if (l.isDone()) {
          listeners.remove(l);  // ConcurrentModificationException 아님
                                // 하지만 iterator의 스냅샷에는 반영 안 됨
      }
  }
  // iterator()는 스냅샷 → remove()가 원본 배열에 작용
  // 현재 순회 중인 iterator에는 삭제가 반영되지 않음 → 의도대로 동작 안 함

실수 3: set()이나 remove()가 다른 스레드의 읽기를 차단한다는 오해
  "COWAL은 쓰기 시 전체를 잠그니까 읽기가 차단된다"
  → 쓰기는 새 배열을 만들고 참조를 교체
  → 읽기 스레드는 이전 배열 스냅샷을 계속 사용 → 차단 없음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// CopyOnWriteArrayList 올바른 사용 패턴

// 적합한 패턴 1: 이벤트 리스너 관리
CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();

// 리스너 등록/해제 (드물게 발생)
listeners.add(newListener);
listeners.remove(oldListener);

// 이벤트 발행 (빈번하게 발생, 락 없음)
for (EventListener l : listeners) {
    l.onEvent(event);
}
// 순회 중 다른 스레드가 add/remove해도 ConcurrentModificationException 없음
// 단, 순회 중 추가된 리스너는 이 순회에서 호출 안 됨 (스냅샷)

// 적합한 패턴 2: 설정 목록 (초기화 후 변경 드묾)
CopyOnWriteArrayList<String> allowedHosts = new CopyOnWriteArrayList<>(
    List.of("host1", "host2", "host3")
);
// 설정 변경: 드물게 새 목록으로 교체
allowedHosts.set(0, "newHost1");  // 내부적으로 전체 복사

// 읽기: 빈번하게, 락 없음
boolean allowed = allowedHosts.contains(requestHost);

// 부적합한 패턴: 빈번한 쓰기
// CopyOnWriteArrayList<Metric> metrics = ...;
// metrics.add(newMetric);  // 초당 수천 번 → O(N) 복사 반복 → 성능 재앙
// → 대안: ConcurrentLinkedQueue, LinkedBlockingQueue
```

---

## 🔬 내부 동작 원리

### 1. 쓰기 시 배열 복사 메커니즘

```java
// CopyOnWriteArrayList 소스 (단순화)
public class CopyOnWriteArrayList<E> {
    final transient ReentrantLock lock = new ReentrantLock();
    private transient volatile Object[] array;  // volatile 참조!

    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();  // 쓰기는 단일 락으로 직렬화
        try {
            Object[] elements = getArray();  // 현재 배열
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);  // 전체 복사!
            newElements[len] = e;
            setArray(newElements);  // volatile 쓰기 → 원자적 참조 교체
            return true;
        } finally {
            lock.unlock();
        }
    }

    public E get(int index) {
        return (E)(getArray()[index]);  // 락 없음! volatile 배열 참조 읽기
    }

    final Object[] getArray() { return array; }  // volatile 읽기
    final void setArray(Object[] a) { array = a; }  // volatile 쓰기

    public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);  // 현재 배열 스냅샷
        // getArray() = 이 시점의 volatile 읽기 → 이후 변경 영향 없음
    }
}

쓰기 흐름:
  [현재 배열: A, B, C] ← array (volatile 참조)
  
  add(D) 호출:
    lock 획득
    현재 배열 복사 → 새 배열 [A, B, C, D]
    array = 새 배열 (volatile 쓰기)
    lock 해제
  
  읽기 스레드 (add 진행 중):
    array 읽기 → 이전 배열 [A, B, C] 참조 유지
    → D가 보이지 않음 (이전 스냅샷)
    
  읽기 스레드 (add 완료 후):
    array 읽기 → 새 배열 [A, B, C, D]
    → D가 보임
```

### 2. Iterator의 스냅샷 보장

```java
static final class COWIterator<E> implements ListIterator<E> {
    private final Object[] snapshot;  // 생성 시점의 배열 스냅샷
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;  // 현재 배열 참조 저장 (복사 아님!)
        // volatile로 읽힌 배열 참조를 저장
        // 이후 add/remove가 새 배열을 만들어도 이 snapshot은 변하지 않음
    }

    public E next() {
        return (E) snapshot[cursor++];  // 스냅샷에서 읽기
    }

    // remove()는 UnsupportedOperationException
    // → iterator를 통한 삭제 불가 (스냅샷이기 때문)
}

스냅샷이 보장되는 이유:
  Java 배열은 참조 타입 → snapshot은 배열 객체를 참조
  add()는 새 배열을 만들고 array를 교체 → 기존 배열 객체는 그대로
  snapshot이 참조하는 이전 배열 객체는 변경되지 않음
  → snapshot.length, snapshot[i]가 변하지 않음 (불변 스냅샷)

메모리 누수 주의:
  오래된 iterator가 이전 배열을 참조 → GC 불가
  큰 배열을 가진 COWAL에서 iterator를 오래 보유 시 메모리 누수
```

### 3. 성능 특성과 메모리

```
쓰기 비용:
  add(E): O(N) 배열 복사 + O(1) 원소 추가 = O(N)
  remove(int): O(N) 복사 = O(N)
  set(int, E): O(N) 복사 = O(N)
  addAll(Collection): O(N + M) (N=현재 크기, M=추가 크기)

읽기 비용:
  get(int): O(1) (배열 인덱스 접근)
  contains(Object): O(N) (순차 탐색)
  iterator(): O(1) (스냅샷 참조 저장)
  for-each: O(N) 순회 (락 없음)

메모리:
  쓰기마다 새 배열 생성 → 이전 배열은 GC 대상
  동시에 여러 iterator 활성화 시 여러 이전 버전의 배열이 힙에 공존
  N × (쓰기 중 활성 iterator 수) 배열이 동시에 존재 가능

CopyOnWriteArraySet:
  내부: CopyOnWriteArrayList
  add(): COWAL.addIfAbsent() 사용 (이미 있으면 추가 안 함)
  → O(N) 복사 + O(N) 중복 검사 = O(N) 쓰기
  → Set이지만 쓰기가 O(N): 대용량 Set에는 부적합
```

---

## 💻 실전 실험

### 실험 1: 스냅샷 동작 확인

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class CowalSnapshotTest {
    public static void main(String[] args) throws InterruptedException {
        CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>(
            new Integer[]{1, 2, 3}
        );

        // iterator 생성 (스냅샷: [1, 2, 3])
        var iterator = list.iterator();

        // iterator 순회 전에 원소 추가
        list.add(4);
        list.add(5);
        System.out.println("추가 후 리스트: " + list);  // [1, 2, 3, 4, 5]

        // iterator는 스냅샷 [1, 2, 3]만 순회
        System.out.print("iterator 순회: ");
        while (iterator.hasNext()) {
            System.out.print(iterator.next() + " ");  // 1 2 3 (4, 5 없음!)
        }
        System.out.println();

        // 순회 중 삭제 시도 → UnsupportedOperationException
        try {
            var iter2 = list.iterator();
            iter2.next();
            iter2.remove();  // 예외 발생
        } catch (UnsupportedOperationException e) {
            System.out.println("remove() 불가: " + e.getClass().getSimpleName());
        }
    }
}
```

### 실험 2: 읽기 vs 쓰기 비율별 JMH 벤치마크

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
public class CowalVsOthersBenchmark {

    CopyOnWriteArrayList<Integer> cowal = new CopyOnWriteArrayList<>();
    CopyOnWriteArrayList<Integer> cowalLarge;
    ConcurrentLinkedQueue<Integer> clq = new ConcurrentLinkedQueue<>();

    @Setup
    public void setup() {
        for (int i = 0; i < 100; i++) cowal.add(i);
        cowalLarge = new CopyOnWriteArrayList<>(cowal);
        for (int i = 0; i < 100; i++) clq.offer(i);
    }

    @Benchmark @Threads(7)
    public int cowalRead() {
        return cowal.get((int)(Math.random() * cowal.size()));
    }

    @Benchmark @Threads(1)
    public void cowalWrite() { cowal.add(1); }

    @Benchmark @Threads(7)
    public int clqRead() {
        int sum = 0;
        for (int v : clq) sum += v;
        return sum;
    }

    @Benchmark @Threads(1)
    public void clqWrite() { clq.offer(1); }
}
// 결과: 읽기 집약(7:1)에서 COWAL이 뛰어남
//       쓰기 증가 시 COWAL의 O(N) 복사 비용 증가
```

---

## 📊 성능/비용 비교

```
CopyOnWriteArrayList vs 동기화된 대안 비교:

특성                  | COWAL             | Collections.synchronizedList | ArrayList + ReadWriteLock
─────────────────────┼───────────────────┼──────────────────────────────┼───────────────────────────
읽기 락              | 없음 (락 free)     | 있음 (synchronized)           | ReadLock
쓰기 락              | ReentrantLock      | synchronized                  | WriteLock
읽기 처리량           | 매우 높음          | 제한적 (읽기 직렬화)            | 높음 (읽기 병렬)
쓰기 비용             | O(N) 복사          | O(1) + 동기화                 | O(1) + WriteLock
iterator 일관성      | 완전 스냅샷         | ConcurrentModificationException| 읽기 락 필요
쓰기 < 1% 비율       | ✅ 최적            | 보통                          | 보통
쓰기 > 10% 비율      | ❌ 부적합          | 보통                          | 적합
```

---

## ⚖️ 트레이드오프

```
COWAL을 선택해야 하는 경우:
  읽기가 압도적으로 많음 (99%+ 읽기)
  쓰기가 드물고 배열 크기가 작음
  iterator를 오래 들고 다녀야 하는 경우 (스냅샷 안전성)
  ConcurrentModificationException 방지가 중요
  이벤트 리스너, 플러그인 목록, 설정 목록

COWAL을 피해야 하는 경우:
  쓰기가 빈번함 (초당 수백~수천 이상)
  배열 크기가 큼 (수천 이상 원소)
  실시간 최신 데이터가 필요함 (스냅샷이 곤란)
  메모리가 제한적임 (이전 배열 복수 존재)
  → 대안: ConcurrentLinkedQueue, BlockingQueue, SynchronizedList + ReadWriteLock
```

---

## 📌 핵심 정리

```
CopyOnWriteArrayList 핵심:

핵심 원리:
  쓰기: ReentrantLock → 전체 배열 복사 → 새 원소 추가 → volatile 참조 교체
  읽기: volatile 배열 참조 읽기 → 락 없음
  iterator: 생성 시점의 배열 참조 스냅샷 → 이후 변경 영향 없음

비용:
  읽기: O(1) (get), 락 없음
  쓰기: O(N) (전체 복사), ReentrantLock
  iterator: O(N) 순회, O(1) 스냅샷 생성

최적 사용:
  읽기 >> 쓰기 비율
  소규모 컬렉션 (수십~수백 원소)
  이벤트 리스너, 설정 목록

피해야 할 상황:
  빈번한 쓰기
  대용량 배열
  메모리 민감한 환경
```

---

## 🤔 생각해볼 문제

**Q1.** `CopyOnWriteArrayList.add()`는 `ReentrantLock`을 사용하는데, 여러 스레드가 동시에 `add()`를 호출하면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

모든 쓰기 연산(`add`, `remove`, `set` 등)은 단일 `ReentrantLock`을 공유한다. 여러 스레드가 동시에 `add()`를 호출하면 하나씩 순서대로 처리된다.

처리 과정:
1. Thread A가 lock 획득 → 현재 배열 복사 → A 추가 → setArray(newArr) → unlock
2. Thread B가 lock 획득 → Thread A가 설정한 새 배열 복사 → B 추가 → setArray → unlock
3. ...

중요한 점: 각 쓰기 스레드는 항상 최신 배열(이전 lock 보유자가 설정한)을 복사한다. lock이 직렬화를 보장하므로 "덮어쓰기" 없이 모든 추가가 순서대로 반영된다.

읽기 스레드는 이 과정 동안 lock 없이 계속 읽을 수 있으며, 각 읽기는 그 순간의 array 참조(volatile)를 사용한다.

</details>

---

**Q2.** `CopyOnWriteArrayList`의 `subList()`로 반환된 부분 리스트에서 수정 연산을 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`CopyOnWriteArrayList.subList()`는 `AbstractList.subList()`를 반환하며, 이를 통한 수정은 원본 `CopyOnWriteArrayList`에 반영된다. 내부적으로 원본의 `set()`, `remove()` 등을 호출한다.

그러나 주의할 점:
- subList 생성 후 원본이 변경되면, subList에 접근 시 `ConcurrentModificationException`이 발생할 수 있다
- subList가 참조하는 배열과 현재 배열이 다르면 modCount 검사로 예외 발생
- 스레드 안전성: subList 자체는 스레드 안전하지 않다

실무에서 `CopyOnWriteArrayList`의 subList는 거의 사용되지 않는다. 스냅샷이 필요하면 `new ArrayList<>(list.subList(from, to))`로 독립적인 복사본을 만드는 것이 안전하다.

</details>

---

<div align="center">

**[⬅️ 이전: ConcurrentHashMap](./01-concurrent-hashmap.md)** | **[홈으로 🏠](../README.md)** | **[다음: BlockingQueue 내부 ➡️](./03-blocking-queue-internals.md)**

</div>
