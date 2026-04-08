# Lock-Free 자료구조 — ConcurrentLinkedQueue 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Michael-Scott Queue 알고리즘은 어떻게 동시에 enqueue/dequeue를 처리하는가?
- `ConcurrentLinkedQueue`의 head/tail CAS 기반 삽입·삭제 과정은?
- 동시 enqueue 시 "helping" 메커니즘이 tail을 따라가는 방식은?
- Non-Blocking 알고리즘 설계 시 ABA와 메모리 재사용을 어떻게 주의해야 하는가?
- `ConcurrentLinkedQueue`가 `BlockingQueue`와 다른 선택이 되는 상황은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`ConcurrentLinkedQueue`가 락 없이 어떻게 스레드 안전성을 보장하는지 이해하면, Lock-Free 알고리즘의 전형적인 패턴(CAS + helping + 논리적 삭제)을 파악할 수 있다. 이 패턴은 `ConcurrentSkipListMap`, `LinkedTransferQueue` 등에서도 반복된다. 언제 `ConcurrentLinkedQueue`를 선택해야 하고 언제 `BlockingQueue`가 나은지도 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: ConcurrentLinkedQueue가 BlockingQueue의 상위 호환이라는 오해
  "Lock-Free니까 BlockingQueue보다 항상 빠르다"
  → ConcurrentLinkedQueue: 큐가 비었을 때 null 반환 (블로킹 없음)
    → 소비자가 poll()을 루프로 계속 호출 → CPU 낭비 (busy-wait)
  → ArrayBlockingQueue: 큐가 비면 소비자 park → CPU 0
  → 생산자-소비자 패턴: BlockingQueue가 대개 더 적합

실수 2: size()가 O(1)이라고 가정
  ConcurrentLinkedQueue queue = new ConcurrentLinkedQueue();
  if (queue.size() > 1000) backpressure();
  → size()는 O(N): 헤드에서 테일까지 순회하며 카운트
  → 고빈도 호출 시 성능 문제
  → 대안: 별도 AtomicInteger 카운터 유지

실수 3: ConcurrentLinkedQueue에서 복합 연산을 안전하다고 가정
  if (!queue.isEmpty()) {
      Object item = queue.poll();  // TOCTOU: isEmpty → poll 사이에 다른 스레드가 poll 가능
  }
  // item이 null일 수 있음
  → poll()의 null 반환을 직접 확인해야 함
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// ConcurrentLinkedQueue 올바른 사용 패턴

ConcurrentLinkedQueue<Task> queue = new ConcurrentLinkedQueue<>();

// 패턴 1: Non-Blocking 생산자-소비자 (CPU 허용 시)
// 생산자
queue.offer(task);  // 항상 성공 (용량 제한 없음)

// 소비자: busy-wait 대신 적절한 대기
while (running) {
    Task task = queue.poll();
    if (task != null) {
        process(task);
    } else {
        Thread.sleep(1);  // 최소한의 백오프
        // 또는 LockSupport.parkNanos(1_000_000);
    }
}

// 패턴 2: 작업 큐로 BlockingQueue 선호 (CPU 효율)
BlockingQueue<Task> blockingQueue = new LinkedBlockingQueue<>(1000);
// 생산자
blockingQueue.put(task);  // 가득 차면 블로킹
// 소비자
Task task = blockingQueue.take();  // 비었으면 블로킹 (park)

// 패턴 3: ConcurrentLinkedQueue가 적합한 경우
// - 소비자가 다른 작업도 하면서 틈틈이 큐를 확인
// - 큐가 거의 항상 비어있지 않음 (busy-wait 낭비 없음)
// - 비동기 이벤트 버퍼
while (running) {
    Task task;
    while ((task = queue.poll()) != null) {
        process(task);  // 큐를 비울 때까지 처리
    }
    doOtherWork();  // 큐가 비면 다른 작업
}
```

---

## 🔬 내부 동작 원리

### 1. Michael-Scott Queue 알고리즘

```
Michael-Scott (1996) Queue: Lock-Free FIFO의 고전적 알고리즘

구조:
  head → [dummy node] → [node1] → [node2] → ... → [nodeN] ← tail

  head: 더미 노드 (실제 데이터 없음, 이미 dequeue된 노드들의 후임)
  tail: 마지막 노드 (가장 최근에 enqueue된 노드, 뒤처질 수 있음)
  노드: (item, next 포인터)

핵심 특성:
  ① tail은 실제 마지막 노드이거나 한 칸 뒤처질 수 있음 (lazy tail update)
  ② 삭제된 head 이전의 노드들은 그 다음 노드를 향해 자기 참조 (sentinel)

ConcurrentLinkedQueue의 Java 구현:
  private static class Node<E> {
      volatile E item;
      volatile Node<E> next;
  }

  private volatile Node<E> head;  // 더미 노드의 후임 = 첫 실제 항목
  private volatile Node<E> tail;  // 마지막 또는 그 전 노드
```

### 2. Enqueue (offer) 상세

```
offer(E e) 알고리즘:

  ① 새 Node 생성: node = new Node(e)
  ② 현재 tail 읽기: t = tail
  ③ t.next 읽기: p = t.next

  ④ tail이 최신인가? (p == null이면 t가 진짜 마지막)
    p == null인 경우 (tail이 최신):
      CAS(t.next, null, node) 성공 → 노드 삽입 완료
        → CAS(tail, t, node) 시도 (tail 전진, 실패해도 무방)
      CAS 실패 → 다른 스레드가 먼저 삽입 → 다시 ②부터

    p != null인 경우 (tail이 뒤처짐, Helping):
      CAS(tail, t, p) 시도 (tail을 실제 마지막으로 전진 도움)
      → 다시 ②부터 (새 tail로 재시도)

시각화:

  초기: head → [D] → [A] ← tail
  T1 삽입 시도: node=[B]
    tail.next == null → CAS([D]→[A].next, null, [B]) 성공
    상태: head → [D] → [A] → [B], tail 아직 [A]
    CAS(tail, [A], [B]) → tail이 [B]로 전진

  T2 삽입 시도 (T1의 CAS 완료 전에 시작):
    t = tail = [A], p = t.next = [B] (! null이 아님)
    → Helping: CAS(tail, [A], [B]) 시도 (T1을 도와줌)
    → 다시 시도: t = tail = [B], p = null
    → CAS([B].next, null, [C]) 성공

Helping 메커니즘의 의의:
  스레드가 다른 스레드의 미완성 작업을 완성시킴
  → "어느 스레드가 항상 진행한다" (Lock-Free Progress 보장)
  → 부분 완료 상태에서 교착 없음
```

### 3. Dequeue (poll) 상세

```
poll() 알고리즘:

  ① head 읽기 (더미 노드)
  ② head.next 읽기 = 첫 번째 실제 항목 노드
  ③ item = first.item
  ④ CAS(first.item, item, null) → 항목을 논리적으로 삭제
  ⑤ 성공 시:
     CAS(head, dummy, first) → first가 새 더미 노드
     → 이전 dummy는 GC 수집 가능
  ⑥ 실패 시: 다른 스레드가 먼저 dequeue → 다시 시도
  ⑦ 큐가 비었으면 null 반환

삭제된 노드 처리:
  poll된 노드: next를 자기 자신으로 설정 (sentinel)
  → "이 노드는 이미 큐에서 제거됨"을 표시
  → 다른 스레드가 이 노드를 만나면 head를 앞으로 전진

논리적 삭제와 물리적 삭제 분리:
  논리적 삭제: item = null (CAS)
  물리적 삭제: head를 다음으로 전진 (별도 CAS)
  → 두 단계로 나눠서 경쟁 감소

remove(Object o) 내부:
  물리적 삭제가 아닌 item을 null로 마킹
  순회 중 item==null인 노드는 건너뜀
  → O(N) 작업, 성능에 주의
```

### 4. ABA 방지 설계

```
ConcurrentLinkedQueue의 ABA 방지 전략:

AtomicStampedReference 없이 ABA를 방지하는 두 가지 방법:

① GC 기반:
  dequeue된 노드를 메모리 풀에 반환하지 않음
  → 참조가 살아있는 한 GC가 해제하지 않음
  → 같은 메모리 주소에 다른 내용이 재할당되는 것을 막음 (C/C++에서의 ABA)
  Java에서: 노드 재사용 없음 → 객체 주소 기반 ABA 없음

② Sentinel 마킹:
  dequeue된 노드: node.next = node (자기 참조)
  → 다른 스레드가 이 노드를 만나면 큐에서 제거된 것을 인식
  → 큐 탐색 시 sentinel 노드 건너뜀

③ 논리적 삭제:
  item = null로 삭제 마킹
  → item이 null인 노드 = 이미 dequeue됨
  → 다른 스레드가 item=null CAS에 성공했어도 논리적으로 무효

한계:
  remove(o) 같은 내부 삭제: item=null 마킹 후 물리 제거
  iterator(): 스냅샷 아님, 순회 중 변경 반영됨 (weakly consistent)
```

---

## 💻 실전 실험

### 실험 1: ConcurrentLinkedQueue vs LinkedBlockingQueue 처리량

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Group)
@Fork(2)
public class QueueBenchmark {

    ConcurrentLinkedQueue<Integer> clq = new ConcurrentLinkedQueue<>();
    LinkedBlockingQueue<Integer> lbq = new LinkedBlockingQueue<>(10000);

    @Benchmark @Group("clq") @GroupThreads(4)
    public void clq_produce() { clq.offer(1); }

    @Benchmark @Group("clq") @GroupThreads(4)
    public Integer clq_consume() { return clq.poll(); }

    @Benchmark @Group("lbq") @GroupThreads(4)
    public void lbq_produce() { lbq.offer(1); }

    @Benchmark @Group("lbq") @GroupThreads(4)
    public Integer lbq_consume() { return lbq.poll(); }
}
// 결과: 생산=소비 균형 상황에서 clq ≈ lbq (큐 비지 않음)
// 소비 > 생산 (큐가 자주 빔): lbq가 CPU 효율적 (park)
// clq poll()이 null이면 소비자가 busy-wait
```

### 실험 2: 내부 구조 시각화

```java
import java.util.concurrent.ConcurrentLinkedQueue;
import java.lang.reflect.*;

public class ClqInternals {
    public static void main(String[] args) throws Exception {
        ConcurrentLinkedQueue<Integer> queue = new ConcurrentLinkedQueue<>();

        // 헤드와 테일 필드 접근 (내부 구조 탐색용)
        Field headField = ConcurrentLinkedQueue.class.getDeclaredField("head");
        Field tailField = ConcurrentLinkedQueue.class.getDeclaredField("tail");
        headField.setAccessible(true);
        tailField.setAccessible(true);

        // 항목 추가
        for (int i = 1; i <= 5; i++) {
            queue.offer(i);
            Object head = headField.get(queue);
            Object tail = tailField.get(queue);
            System.out.printf("offer(%d): head=%s, tail=%s%n",
                i, nodeItem(head), nodeItem(tail));
        }

        // 항목 제거
        while (!queue.isEmpty()) {
            int val = queue.poll();
            Object head = headField.get(queue);
            Object tail = tailField.get(queue);
            System.out.printf("poll()=%d: head=%s, tail=%s%n",
                val, nodeItem(head), nodeItem(tail));
        }
    }

    static String nodeItem(Object node) throws Exception {
        if (node == null) return "null";
        Field item = node.getClass().getDeclaredField("item");
        item.setAccessible(true);
        return String.valueOf(item.get(node));
    }
}
```

---

## 📊 성능/비용 비교

```
Queue 종류별 특성 비교:

특성                     | ConcurrentLinkedQueue| ArrayBlockingQueue | LinkedBlockingQueue
────────────────────────┼──────────────────────┼────────────────────┼───────────────────
알고리즘                  | Lock-Free (M-S Queue)| 단일 ReentrantLock  | head/tail 분리 락
용량                     | 무한 (힙 한계)          | 고정 (생성 시 설정)    | 선택적 (무제한 기본)
큐 빌 때 소비자            | null 반환 (non-blocking)| park (blocking)  | park (blocking)
큐 가득 찰 때 생산자        | 불가 (무한 용량)         | park (blocking)    | park (무한 시 없음)
size()                  | O(N)                 | O(1)               | O(1)
peek()                  | O(1)                 | O(1)               | O(1)
iterator 일관성          | Weakly consistent     | Weakly consistent  | Weakly consistent
CPU 효율 (큐 빌 때)       | 나쁨 (busy-wait 위험)   | 좋음 (park)         | 좋음 (park)
처리량 (경쟁 없음)         | 높음                   | 높음                | 높음

적합한 사용 사례:
  ConcurrentLinkedQueue: 소비자가 다른 작업도 하는 이벤트 버퍼
  ArrayBlockingQueue:    고정 크기, 백프레셔 필요한 생산자-소비자
  LinkedBlockingQueue:   가변 크기, 생산자-소비자 기본 선택
```

---

## ⚖️ 트레이드오프

```
Lock-Free 자료구조의 장단점:

장점:
  OS 스케줄러 불확실성에 강함 (스레드 선점돼도 전체가 멈추지 않음)
  락 경합으로 인한 컨텍스트 스위칭 없음
  낮은~중간 경쟁에서 우수한 처리량

단점:
  구현 복잡성: ABA, Helping, 논리적 삭제 등 고려 사항 많음
  메모리 재사용 제한: 노드를 풀링하기 어려움 (ABA 위험)
  불공정성: 특정 스레드가 계속 CAS 실패 가능 (기아)
  size() 비용: ConcurrentLinkedQueue는 O(N)

Lock-Free vs Blocking 큐 선택:

  소비 속도 < 생산 속도인 경우:
    BlockingQueue + 용량 제한 → 백프레셔 자동 적용
    ConcurrentLinkedQueue → 메모리 무제한 증가 위험

  소비자가 항상 바쁜 경우:
    ConcurrentLinkedQueue → null 반환 시 즉시 다음 작업
    BlockingQueue → 큐가 비면 불필요하게 park

  실시간 응답성 요구:
    ConcurrentLinkedQueue → OS 스케줄링 없이 진행 보장
    BlockingQueue → park/unpark 지연

실무 권장:
  대부분의 생산자-소비자: LinkedBlockingQueue / ArrayBlockingQueue
  ConcurrentLinkedQueue: 블로킹 자체가 문제인 특수 경우
  직접 Lock-Free 구현: 최후의 수단
```

---

## 📌 핵심 정리

```
ConcurrentLinkedQueue 핵심:

Michael-Scott Queue:
  head(dummy) → node1 → node2 → ... → nodeN ← tail
  tail은 실제 마지막이거나 한 칸 뒤처질 수 있음

Enqueue:
  tail.next CAS(null, newNode) → 성공 시 tail 전진
  tail 뒤처진 경우: Helping으로 tail 전진 후 재시도

Dequeue:
  head.next의 item을 null로 CAS (논리 삭제)
  head를 다음 노드로 전진 (물리 삭제)

ABA 방지:
  GC 기반 (노드 재사용 없음)
  Sentinel 마킹 (dequeue된 노드.next = 자기 자신)
  논리적 삭제 (item = null)

주의사항:
  size() = O(N) → 고빈도 호출 금지
  큐 비었을 때 poll() = null (blocking 없음)
  생산자-소비자: 대부분 BlockingQueue가 더 적합
  iterator: weakly consistent (스냅샷 아님)
```

---

## 🤔 생각해볼 문제

**Q1.** `ConcurrentLinkedQueue.size()`가 O(N)인데 왜 O(1)로 최적화하지 않는가? (별도 AtomicInteger 카운터 유지)

<details>
<summary>해설 보기</summary>

별도 카운터를 유지하면 enqueue/dequeue마다 카운터 CAS가 추가로 필요하다. 이는 두 가지 문제를 야기한다.

첫째, 성능: 큐 노드 CAS + 카운터 CAS = CAS 2회. Lock-Free 알고리즘에서 두 CAS를 원자적으로 수행하는 것은 불가능하다 (CAS2는 없음). 즉, 노드 삽입 성공 후 카운터 업데이트 전에 스레드가 선점되면 `size()`가 잠시 부정확해진다.

둘째, 의미론: 어차피 완전한 일관성을 보장할 수 없다면, size()를 O(1)로 만들어도 "순간적으로 정확한 크기"는 보장할 수 없다. `ConcurrentLinkedQueue`의 Javadoc은 "size()는 다른 스레드 활동 때문에 부정확할 수 있다"고 명시한다.

따라서 카운터 유지의 비용(추가 CAS, 메모리)이 일관성 이점보다 크지 않다는 설계 결정이다. 정확한 크기가 필요하면 별도 `AtomicInteger`를 관리하거나 `LinkedBlockingQueue`(size() = O(1))를 사용한다.

</details>

---

**Q2.** `ConcurrentLinkedQueue`에서 `iterator()`가 "weakly consistent"라는 의미는?

<details>
<summary>해설 보기</summary>

"Weakly consistent" iterator는 다음 속성을 가진다:
- iterator 생성 당시 큐에 있던 요소들을 정확히 한 번 반환한다
- iterator 생성 이후의 변경사항(추가/제거)이 반영될 수도 있고 안 될 수도 있다
- `ConcurrentModificationException`을 던지지 않는다

이는 스냅샷(snapshot) iterator와 다르다. 스냅샷은 생성 시점의 상태를 정확히 복사하여 그 이후 변경의 영향을 받지 않는다. `CopyOnWriteArrayList`의 iterator가 스냅샷 방식이다.

`ConcurrentLinkedQueue`가 weakly consistent인 이유: 순회 중 노드들은 물리적으로 큐에 남아 있지만 item이 null로 마킹되어 논리적으로 삭제될 수 있다. iterator는 item이 null인 노드를 건너뛰므로, 순회 도중 dequeue된 요소는 보이지 않을 수 있다.

</details>

---

**Q3.** Lock-Free 알고리즘이 "항상 적어도 하나의 스레드가 진행한다"고 보장하는데, 현실적으로 라이브락이 문제가 될 수 있는가?

<details>
<summary>해설 보기</summary>

이론적으로 Lock-Free는 라이브락을 배제하지 않는다. Lock-Free 정의("시스템 전체에서 항상 누군가 진행")는 특정 스레드가 영원히 CAS 실패를 경험하는 것을 허용한다.

현실에서 라이브락이 문제가 되는 조건:
1. 매우 많은 스레드가 하나의 공유 자원에 동시에 CAS를 시도
2. CAS 실패 후 재시도가 즉각적 (백오프 없음)
3. 스케줄러가 모든 스레드에 동시에 타임슬라이스를 부여

실제로 x86의 `LOCK CMPXCHG`는 직렬화되므로 하나가 반드시 성공하고, 나머지는 대기한다. 진짜 라이브락보다는 "특정 불운한 스레드가 계속 CAS 실패"(기아, Starvation)가 더 현실적이다.

해결책:
- 지수 백오프(Exponential Backoff): 실패 시 랜덤하게 대기 후 재시도
- `LongAdder`처럼 경합 지점 자체를 분산
- Wait-Free 알고리즘(구현 매우 복잡): 모든 스레드가 유한 시간 내 완료

</details>

---

<div align="center">

**[⬅️ 이전: LongAdder와 Striped64](./04-longadder-striped64.md)** | **[홈으로 🏠](../README.md)** | **[다음: VarHandle 메모리 오더링 ➡️](./06-varhandle-memory-ordering.md)**

</div>
