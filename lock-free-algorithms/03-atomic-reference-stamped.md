# AtomicReference와 AtomicStampedReference — ABA 해결

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AtomicReference`로 참조 타입을 원자적으로 교체하는 방식은?
- ABA 문제가 참조 타입에서 실제 버그로 이어지는 구체적 시나리오는?
- `AtomicStampedReference`의 (참조, 스탬프) 쌍이 ABA를 방지하는 원리는?
- `AtomicMarkableReference`와 `AtomicStampedReference`의 차이와 용도는?
- Java GC 환경에서 ABA 문제가 C/C++ 대비 어떻게 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Lock-Free 자료구조(스택, 큐, 맵)를 직접 구현할 때 ABA 문제를 이해하지 못하면, 드물게 재현되는 치명적 버그를 만들게 된다. `ConcurrentLinkedQueue` 등 JDK 구현체가 어떻게 ABA를 다루는지 이해하면, Lock-Free 알고리즘 리뷰 시 ABA 취약점을 빠르게 찾아낼 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: AtomicReference로 모든 참조 교체가 안전하다고 오해
  AtomicReference<Config> configRef = new AtomicReference<>(old);
  configRef.compareAndSet(old, newConfig);
  "참조를 원자적으로 교체하니까 안전하다"

  → 단순 참조 교체는 안전 (Config 교체 패턴)
  → Lock-Free 자료구조의 노드 포인터 조작에서는 ABA 위험
  → ABA = 같은 메모리 주소/객체가 다른 논리적 상태를 가질 때

실수 2: Java GC가 ABA를 완전히 방지한다고 오해
  "Java는 GC가 있어서 use-after-free가 없다 = ABA 없다"
  → GC가 메모리 해제를 방지 → 포인터 자체의 use-after-free는 없음
  → 하지만 논리적 ABA: 같은 객체가 재삽입되면 여전히 발생
  → 특히 객체 풀(Object Pool) 패턴에서 위험

실수 3: AtomicStampedReference를 모든 상황에 남용
  "안전하게 하려면 항상 stamp를 쓰자"
  → Pair 객체 생성으로 GC 압박 증가
  → ABA가 실제로 문제가 되는 패턴인지 먼저 분석 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// AtomicReference 안전한 사용 패턴

// 패턴 1: 불변 객체 참조 교체 (ABA 무관 - 논리적 동일성)
AtomicReference<ImmutableConfig> config = new AtomicReference<>(initial);
// 설정 업데이트: 새 객체로 교체 (이전 객체는 버려짐)
config.set(new ImmutableConfig(newValues));

// 패턴 2: CAS로 참조 교체 (ABA 위험 여부 분석 후)
ImmutableCache current = cache.get();
ImmutableCache updated = current.withNewEntry(key, value);
if (!cache.compareAndSet(current, updated)) {
    // 다른 스레드가 먼저 변경 → 재시도 또는 포기
}

// 패턴 3: Lock-Free 스택 (ABA 위험 → AtomicStampedReference)
class LockFreeStack<T> {
    static class Node<T> {
        final T value;
        Node<T> next;
        Node(T v, Node<T> n) { value = v; next = n; }
    }

    private final AtomicStampedReference<Node<T>> top =
        new AtomicStampedReference<>(null, 0);

    public void push(T value) {
        int[] stamp = new int[1];
        while (true) {
            Node<T> current = top.get(stamp);
            Node<T> node = new Node<>(value, current);
            if (top.compareAndSet(current, node, stamp[0], stamp[0] + 1))
                return;
        }
    }

    public T pop() {
        int[] stamp = new int[1];
        while (true) {
            Node<T> current = top.get(stamp);
            if (current == null) return null;
            if (top.compareAndSet(current, current.next,
                                  stamp[0], stamp[0] + 1))
                return current.value;
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. AtomicReference 내부 구조

```java
// AtomicReference 소스 (단순화)
public class AtomicReference<V> {
    private static final VarHandle VALUE;
    static {
        try {
            VALUE = MethodHandles.lookup()
                .findVarHandle(AtomicReference.class, "value", Object.class);
        } catch (ReflectiveOperationException e) { throw new Error(e); }
    }

    private volatile V value;

    public final boolean compareAndSet(V expectedValue, V newValue) {
        return VALUE.compareAndSet(this, expectedValue, newValue);
        // 참조 동등성 비교: expectedValue == value (주소 비교)
        // 값 동등성(equals())이 아님!
    }

    public final V getAndSet(V newValue) {
        return (V) VALUE.getAndSet(this, newValue);
    }

    // Java 9+: updateAndGet, accumulateAndGet 지원
    public final V updateAndGet(UnaryOperator<V> updateFunction) {
        V prev = get(), next = null;
        for (boolean haveNext = false;;) {
            if (!haveNext) next = updateFunction.apply(prev);
            if (VALUE.weakCompareAndSetVolatile(this, prev, next))
                return next;
            haveNext = (prev == (prev = get()));
        }
    }
}

참조 동등성 주의:
  String a = new String("hello");
  String b = new String("hello");
  AtomicReference<String> ref = new AtomicReference<>(a);

  ref.compareAndSet(b, "world")  // 실패! (a != b, 다른 객체)
  ref.compareAndSet(a, "world")  // 성공 (동일 참조)

  → CAS는 항상 참조 동등성(==)으로 비교
  → equals()가 아님 → 동등한 내용의 다른 객체는 expected로 사용 불가
```

### 2. ABA 문제가 참조 타입에서 실제 버그가 되는 시나리오

```
시나리오: 객체 풀(Object Pool) + Lock-Free 스택

class ObjectPool {
    AtomicReference<Node> top = new AtomicReference<>(null);

    // 객체 반납 (스택에 push)
    void recycle(Node node) {
        Node current;
        do {
            current = top.get();
            node.next = current;
        } while (!top.compareAndSet(current, node));
    }

    // 객체 대여 (스택에서 pop)
    Node acquire() {
        Node current, next;
        do {
            current = top.get();
            if (current == null) return new Node();
            next = current.next;
        } while (!top.compareAndSet(current, next));
        return current;
    }
}

ABA 시나리오:
  초기 스택: top → A → B → null

  Thread 1: current = A, next = A.next = B (pop 준비, 일시 정지)

  Thread 2:
    acquire() → pop A  (top = B → null)
    acquire() → pop B  (top = null)
    recycle(A) → push A 다시  (A.next = null, top = A)
    (같은 Node A 객체가 재사용됨, but A.next가 변경됨!)

  Thread 1 재개:
    CAS(top, A, B) → 성공! (top이 여전히 A를 가리킴)
    top = B 로 설정
    하지만 B는 이미 pop돼서 누군가 사용 중!
    → 같은 객체를 두 스레드가 동시에 사용 → 데이터 손상

Java GC의 부분적 보호:
  GC가 살아있는 참조를 해제하지 않으므로
  B가 다른 스레드에 반환된 경우 → B는 GC되지 않음
  하지만 Thread 1이 B를 사용하면 논리적으로 같은 객체를 두 곳에서 사용
  = 논리적 ABA 버그 (메모리 안전하지만 데이터 일관성 훼손)
```

### 3. AtomicStampedReference 내부 구현

```java
// AtomicStampedReference 소스 (단순화)
public class AtomicStampedReference<V> {
    private static class Pair<T> {
        final T reference;
        final int stamp;

        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }

        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<>(reference, stamp);
        }
    }

    private volatile Pair<V> pair;
    // 참조와 스탬프를 하나의 Pair 객체로 묶어 단일 volatile 참조로 관리

    public boolean compareAndSet(V expectedReference, V newReference,
                                  int expectedStamp, int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            (
                (newReference == current.reference && newStamp == current.stamp) ||
                casPair(current, Pair.of(newReference, newStamp))
            );
    }

    private boolean casPair(Pair<V> cmp, Pair<V> val) {
        return PAIR.compareAndSet(this, cmp, val);
        // 단일 참조(Pair 객체)에 대한 CAS
    }
}

ABA 방지 원리:
  스탬프 = 단조 증가 버전 카운터

  A(stamp=0) → B(stamp=1) → A(stamp=2)

  Thread 1: expected = (A, stamp=0) 로 기억
  Thread 2: (A,0)→(B,1)→(A,2) 변경
  Thread 1: CAS((A,0), (newRef, newStamp))
    current = (A, stamp=2)
    expectedStamp(0) != current.stamp(2) → CAS 실패!
    → ABA 탐지 성공

비용 분석:
  읽기: volatile Pair 참조 1회 (빠름)
  쓰기: 새 Pair 객체 생성 + CAS
        → 매 성공적 쓰기마다 Pair 객체 생성 → GC 압박
  → 고빈도 업데이트에서 GC 오버헤드 고려
```

### 4. AtomicMarkableReference 비교

```java
// AtomicMarkableReference: (참조, boolean) 쌍
public class AtomicMarkableReference<V> {
    private static class Pair<T> {
        final T reference;
        final boolean mark;
        // 내부: AtomicStampedReference와 동일하지만 stamp 대신 boolean
    }

    public boolean compareAndSet(V expectedReference, V newReference,
                                  boolean expectedMark, boolean newMark) { ... }
}

용도: 삭제 마킹 (Logical Deletion)
  ConcurrentSkipListMap 내부에서 노드 삭제 시:
  1단계: 노드에 "삭제 예정" 마크 설정 (mark = true)
         CAS((node, false), (node, true))
  2단계: 마크된 노드를 리스트에서 물리적 제거
         다른 스레드가 마크된 노드를 삽입 시도하면 실패

  이유: 물리 삭제와 삽입이 동시에 일어나면 삽입이 유실될 수 있음
  → 마킹으로 "이 노드는 곧 삭제됨"을 알려 삽입 시도를 막음

AtomicStampedReference vs AtomicMarkableReference:
  StampedRef: 여러 번 변경을 추적해야 할 때 (Lock-Free 스택)
  MarkableRef: 삭제 마킹처럼 1비트 상태만 필요할 때
  성능: 동일 (내부 구조 같음)
```

---

## 💻 실전 실험

### 실험 1: ABA 문제 발생과 AtomicStampedReference 방지 비교

```java
import java.util.concurrent.atomic.*;

public class AbaComparison {
    // ABA 취약한 스택
    static final AtomicReference<Integer> badTop = new AtomicReference<>(1);

    // ABA 안전한 스택
    static final AtomicStampedReference<Integer> safeTop =
        new AtomicStampedReference<>(1, 0);

    public static void main(String[] args) throws InterruptedException {
        // === ABA 취약 버전 시뮬레이션 ===
        System.out.println("=== AtomicReference (ABA 취약) ===");
        Integer expected = badTop.get();  // Thread 1: 값 1 읽음
        System.out.println("T1이 읽음: " + expected);

        // Thread 2 개입: 1→2→1
        badTop.set(2);
        badTop.set(1);
        System.out.println("T2 개입 후: " + badTop.get() + " (ABA 발생!)");

        boolean result = badTop.compareAndSet(expected, 99);
        System.out.println("T1 CAS(1→99): " + result + " (ABA 미감지, 성공!)");
        System.out.println("최종값: " + badTop.get());

        // === ABA 안전 버전 ===
        System.out.println("\n=== AtomicStampedReference (ABA 방지) ===");
        int[] stamp = new int[1];
        Integer safeExpected = safeTop.get(stamp);
        int expectedStamp = stamp[0];
        System.out.println("T1이 읽음: value=" + safeExpected + ", stamp=" + expectedStamp);

        // Thread 2 개입: (1,0)→(2,1)→(1,2)
        safeTop.compareAndSet(1, 2, 0, 1);
        safeTop.compareAndSet(2, 1, 1, 2);
        safeTop.get(stamp);
        System.out.println("T2 개입 후: value=" + safeTop.getReference() + ", stamp=" + stamp[0]);

        result = safeTop.compareAndSet(safeExpected, 99, expectedStamp, expectedStamp + 1);
        System.out.println("T1 CAS: " + result + " (stamp 불일치로 실패! ABA 감지)");
        System.out.println("최종값: " + safeTop.getReference() + ", stamp=" + safeTop.getStamp());
    }
}
```

### 실험 2: ABA 안전 Lock-Free 스택 성능 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(4)
public class StackBenchmark {

    // AtomicReference 스택 (ABA 취약)
    static class FastStack {
        static class Node { int v; Node next; Node(int v, Node n){this.v=v;next=n;} }
        AtomicReference<Node> top = new AtomicReference<>();
        void push(int v){Node n=new Node(v,null);Node c;do{c=top.get();n.next=c;}while(!top.compareAndSet(c,n));}
        int pop(){Node c,nx;do{c=top.get();if(c==null)return-1;nx=c.next;}while(!top.compareAndSet(c,nx));return c.v;}
    }

    // AtomicStampedReference 스택 (ABA 안전)
    static class SafeStack {
        static class Node { int v; Node next; Node(int v, Node n){this.v=v;next=n;} }
        AtomicStampedReference<Node> top = new AtomicStampedReference<>(null, 0);
        void push(int v){int[]s=new int[1];while(true){Node c=top.get(s);Node n=new Node(v,c);if(top.compareAndSet(c,n,s[0],s[0]+1))return;}}
        int pop(){int[]s=new int[1];while(true){Node c=top.get(s);if(c==null)return-1;if(top.compareAndSet(c,c.next,s[0],s[0]+1))return c.v;}}
    }

    FastStack fast = new FastStack();
    SafeStack safe = new SafeStack();

    @Setup public void setup(){for(int i=0;i<100;i++){fast.push(i);safe.push(i);}}

    @Benchmark public void fastStackOps() { fast.push(1); fast.pop(); }
    @Benchmark public void safeStackOps() { safe.push(1); safe.pop(); }
}
// 결과: SafeStack이 Pair 객체 생성으로 약 10~20% 느림
// ABA 안전성의 비용
```

---

## 📊 성능/비용 비교

```
AtomicReference vs AtomicStampedReference 비교:

특성                    | AtomicReference | AtomicStampedReference
───────────────────────┼────────────────┼───────────────────────────
ABA 방지                | ❌              | ✅ (stamp 불일치 감지)
쓰기당 객체 생성         | ❌ (없음)       | ✅ (Pair 객체 1개)
GC 압박                 | 낮음           | 중간 (쓰기 빈도에 비례)
읽기 비용               | volatile 1회    | volatile 1회 (동일)
쓰기 비용               | CAS 1회         | Pair 생성 + CAS
코드 복잡성             | 낮음           | 중간 (int[] stamp 배열)

사용 사례별 권장:

상황                              | 권장
─────────────────────────────────┼───────────────────────────────────
불변 설정 객체 교체                | AtomicReference (ABA 무관)
Lock-Free 스택/큐 노드 포인터      | AtomicStampedReference
ConcurrentSkipList 노드 삭제 마킹 | AtomicMarkableReference
단순 참조 플래그                   | AtomicReference (재삽입 없으면)
객체 풀 free list                 | AtomicStampedReference (재사용)
```

---

## ⚖️ 트레이드오프

```
ABA 방지의 비용-이득 분석:

AtomicStampedReference 도입이 필요한 경우:
  Lock-Free 자료구조에서 노드가 재사용됨 (객체 풀, free list)
  같은 참조가 다른 논리적 상태로 재삽입될 수 있음
  ABA로 인한 데이터 손상이 용납 불가한 경우

AtomicReference로 충분한 경우:
  참조가 한 방향으로만 변함 (null → non-null, non-null → 다른 값)
  재삽입이 없는 불변 객체 교체 패턴
  단순 "게시(publication)" 패턴

GC 언어(Java) vs 네이티브(C++):
  C++: ABA = 해제된 메모리 재할당 → use-after-free → 크래시 또는 취약점
       Hazard Pointer, RCU, Epoch-Based Reclamation으로 방지
  Java: GC가 참조 보유 중인 객체 해제 방지 → 메모리 안전
        논리적 ABA(같은 객체 재삽입)만 위험
        → AtomicStampedReference로 충분

실무 대안:
  Lock-Free 구현 대신 java.util.concurrent 클래스 사용
  ConcurrentLinkedQueue, ConcurrentSkipListMap 등 이미 ABA 처리됨
  직접 Lock-Free 구현은 최후 수단으로
```

---

## 📌 핵심 정리

```
AtomicReference / AtomicStampedReference 핵심:

AtomicReference:
  참조 동등성(==)으로 CAS
  불변 객체 교체, 단방향 변경에 적합
  ABA 취약: 재삽입이 있는 자료구조에서 위험

ABA 문제:
  A → B → A 변경을 CAS가 감지 못함
  참조 타입: 같은 객체 재삽입 → 논리적 상태 불일치
  Java: 메모리는 안전, 논리적 데이터 손상 위험

AtomicStampedReference:
  (reference, stamp) Pair를 단일 volatile로 관리
  stamp = 단조 증가 버전 카운터
  ABA: stamp 불일치로 탐지
  비용: 쓰기마다 Pair 객체 생성 → GC 압박

AtomicMarkableReference:
  (reference, boolean) Pair
  논리적 삭제 마킹 용도 (ConcurrentSkipListMap 내부)

선택 원칙:
  재삽입 가능 Lock-Free 자료구조 → AtomicStampedReference
  삭제 마킹 패턴 → AtomicMarkableReference
  단순 참조 교체 → AtomicReference
  직접 구현 전: java.util.concurrent 클래스 먼저 고려
```

---

## 🤔 생각해볼 문제

**Q1.** `AtomicStampedReference`를 사용해도 스탬프가 결국 오버플로우하면 ABA 문제가 재발할 수 있지 않은가?

<details>
<summary>해설 보기</summary>

이론적으로는 맞다. `stamp`는 `int`(32비트)이므로 2^32회 변경 후 오버플로우하여 같은 값으로 돌아온다. 하지만 실제로는 무시 가능하다.

계산: 1나노초마다 CAS가 성공한다고 가정해도 2^32 / 10^9 ≈ 4.3초. 하지만 실제로는:
1. CAS는 나노초 단위보다 훨씬 느리고
2. 하나의 참조에 이런 속도로 쓰기가 발생하지 않으며
3. Thread 1이 read와 CAS 사이에 2^32번의 변경이 일어날 동안 정지해 있어야 함

현실적으로 int 스탬프의 오버플로우 ABA는 발생하지 않는다. 더 걱정해야 할 것은 높은 쓰기 빈도로 인한 Pair 객체 생성 비용이다.

</details>

---

**Q2.** `ConcurrentLinkedQueue`는 내부적으로 ABA 문제를 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

`ConcurrentLinkedQueue`는 Michael-Scott 알고리즘을 기반으로 하며, `AtomicStampedReference` 없이 ABA를 다룬다.

핵심 설계:
1. **노드를 해제하지 않음**: pop된 노드를 즉시 재사용하지 않고 GC에 맡긴다. GC가 수거하기 전까지 해당 포인터는 유효하지만, 큐에서 제거됐음을 `item = null`로 표시한다.

2. **"자기 참조" 탈출 마커**: 삭제된 노드의 `next`를 자기 자신으로 설정하여 이 노드가 큐에서 제거됐음을 나타낸다. 다른 스레드가 이 노드를 만나면 head를 앞으로 전진시킨다.

3. **Helping 메커니즘**: tail이 뒤처진 경우, 다른 스레드가 tail을 앞으로 전진시켜 도와준다.

결국 Java의 GC와 "논리적 삭제 마킹"의 조합으로 `AtomicStampedReference`의 비용 없이 안전하게 동작한다.

</details>

---

**Q3.** `AtomicReference`의 `updateAndGet()`은 ABA가 발생해도 항상 올바른 결과를 반환하는가?

<details>
<summary>해설 보기</summary>

함수의 의미론에 따라 다르다. `updateAndGet(fn)`은 현재 값에 함수를 적용하여 새 값으로 원자적으로 교체한다. CAS가 실패하면 최신 값으로 재계산 후 재시도한다.

ABA가 발생해도 **함수 결과가 현재 상태에만 의존**한다면 문제없다:
```java
// 안전: 현재 값만 보고 새 값 결정
ref.updateAndGet(v -> v.withNewField(x));
```

하지만 **함수가 "중간 상태가 없었어야 함"을 가정**한다면 문제가 된다:
```java
// 위험: 트랜잭션 로그에 의존하는 경우
// 중간에 다른 트랜잭션이 있었다면 논리적으로 잘못된 결과
ref.updateAndGet(v -> v.applyTransaction(txn));
```

결론: `updateAndGet()`은 최신 값을 기반으로 순수 함수를 적용하는 경우 ABA와 무관하다. "중간에 변경이 없었어야 한다"는 불변식이 필요한 경우 ABA 취약점이 된다.

</details>

---

<div align="center">

**[⬅️ 이전: AtomicInteger/VarHandle](./02-atomic-integer-varhandle.md)** | **[홈으로 🏠](../README.md)** | **[다음: LongAdder와 Striped64 ➡️](./04-longadder-striped64.md)**

</div>
