# CAS — LOCK CMPXCHG와 ABA 문제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CAS는 CPU 명령어 수준에서 어떻게 원자성을 보장하는가?
- x86의 `LOCK CMPXCHG`는 메모리 버스를 어떻게 활용하는가?
- ABA 문제는 어떤 상황에서 실제 버그로 이어지는가?
- `PrintAssembly`로 CAS 명령어를 어떻게 확인하는가?
- CAS 기반 Lock-Free 알고리즘이 락보다 항상 빠른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`AtomicInteger`, `ConcurrentHashMap`, `LongAdder` 등 Java의 동시성 클래스 대부분이 CAS 위에 구현된다. CAS가 CPU 레벨에서 어떻게 동작하는지 이해하면, 왜 고경쟁 환경에서 AtomicLong이 느려지고 LongAdder가 도입됐는지, ABA 문제가 어떤 자료구조에서 실제 위험인지를 직관적으로 파악할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: CAS = 락 없이 완전히 안전하다는 오해
  "AtomicInteger는 락이 없으니까 경쟁이 없다"
  → CAS는 실패 시 재시도 (스핀) → 경쟁이 심하면 CPU 낭비
  → 동시 스레드 수가 많을수록 CAS 실패율 증가 → 처리량 역전

실수 2: ABA 문제를 단순 값 타입에서 무시
  AtomicInteger에서 0 → 1 → 0 변경을 CAS가 감지 못함
  "하지만 최종 값이 0이니 문제없지 않나?"
  → 단순 카운터에서는 대개 문제없음
  → 참조 타입 포인터에서는 심각한 메모리 버그로 이어짐
  → Lock-Free 스택/큐에서 ABA = use-after-free 수준

실수 3: CAS가 항상 O(1)이라고 가정
  "CAS는 단일 명령어니까 항상 상수 시간"
  → CAS 실패 시 루프 재시도 = O(경쟁 수준)
  → 극단적 경쟁에서 라이브락(livelock) 가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// CAS 사용 올바른 패턴

// 패턴 1: 단순 증감 → AtomicInteger (CAS 내장)
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // 내부: CAS 루프

// 패턴 2: 복잡한 상태 변환 → compareAndSet 명시적 사용
AtomicInteger state = new AtomicInteger(0);
int current, next;
do {
    current = state.get();
    next = computeNext(current);  // 새 값 계산
} while (!state.compareAndSet(current, next));
// CAS 실패 시 최신 값으로 재계산 후 재시도 → Optimistic Locking 패턴

// 패턴 3: 참조 타입 + ABA 위험 → AtomicStampedReference
AtomicStampedReference<Node> head = new AtomicStampedReference<>(null, 0);
int[] stamp = new int[1];
Node current = head.get(stamp);
int currentStamp = stamp[0];
Node newHead = new Node(value, current);
head.compareAndSet(current, newHead, currentStamp, currentStamp + 1);
// stamp를 버전 카운터로 사용 → ABA 방지

// 패턴 4: 고경쟁 카운터 → LongAdder
LongAdder adder = new LongAdder();
adder.increment();   // 경합 시 Cell 분산 → CAS 실패율 낮음
long sum = adder.sum();  // 최종 합산 (단, 읽기 중 일관성 미보장)
```

---

## 🔬 내부 동작 원리

### 1. CAS의 CPU 구현 — LOCK CMPXCHG

```
CAS (Compare-And-Swap) 의미론:
  원자적으로:
    if (메모리[addr] == expected) {
        메모리[addr] = newValue;
        return true;
    } else {
        return false;
    }

x86 어셈블리 구현:
  LOCK CMPXCHG [addr], newValue
    1. LOCK 접두사: 해당 명령어 실행 동안 메모리 버스 독점
       (캐시 라인 락: 실제로는 버스 전체가 아닌 캐시 라인 수준)
    2. CMPXCHG: EAX(expected)와 [addr]를 비교
       같으면: [addr] = newValue, ZF(Zero Flag) = 1
       다르면: EAX = [addr] (현재 값 로드), ZF = 0
    3. ZF 확인으로 성공/실패 판단

LOCK 접두사의 실제 동작 (현대 x86):
  캐시 일관성 프로토콜(MESI)이 있으므로
  메모리 버스 자체를 잠그지 않음 (구형 방식)
  대신: 해당 캐시 라인을 Modified 상태로 독점
  → 다른 코어가 같은 캐시 라인을 수정하려 하면 대기
  → 캐시 라인 크기(64바이트) 단위 원자성

비용:
  캐시 히트 시: ~20~40 사이클 (경쟁 없을 때)
  캐시 미스 + 경쟁 시: ~100~500 사이클
  vs 일반 load: ~4 사이클

PrintAssembly로 확인:
  AtomicInteger.incrementAndGet() 컴파일 결과:
    lock cmpxchg DWORD PTR [rsi+0xc],ecx
    ← LOCK CMPXCHG 명령어가 직접 보임
```

### 2. CAS 루프 패턴과 스핀의 비용

```
CAS 루프 (Optimistic Locking):

  int expected, next;
  do {
      expected = var.get();      // ① 현재 값 읽기
      next = expected + 1;       // ② 새 값 계산
  } while (!var.compareAndSet(expected, next));  // ③ CAS 시도
  // 실패: ①부터 재시도

경쟁이 없을 때 (스레드 1개):
  CAS 항상 성공 → 루프 1회 → ~20~40 사이클

경쟁이 중간일 때 (스레드 4~8개):
  CAS 가끔 실패 → 평균 1~3회 재시도
  → 처리량이 낮아지기 시작

경쟁이 심할 때 (스레드 32~100개):
  대부분의 CAS 실패 → 수십 번 재시도
  모든 스레드가 같은 캐시 라인을 놓고 경합
  → 처리량이 스레드 수 증가에 역비례
  → 라이브락(livelock) 가능: 모두가 재시도하지만 아무도 진행 못함

CAS 실패율이 높을 때 대안:
  ① LongAdder: Cell 배열로 경합 분산 (Chapter 4-04)
  ② 지수 백오프: 실패 시 랜덤하게 대기 후 재시도
  ③ Lock: 심한 경쟁에서 OS Mutex가 오히려 공정하고 효율적
  ④ Lock-Free 알고리즘 재설계: 경합 지점 자체를 없앰
```

### 3. ABA 문제 상세

```
ABA 문제 정의:
  시간 T1: 스레드 A가 변수 V의 값을 읽음 → V = "A"
  시간 T2: 스레드 B가 V를 "A" → "B"로 변경
  시간 T3: 스레드 B가 V를 "B" → "A"로 복원
  시간 T4: 스레드 A가 CAS(V, "A", new_value) → 성공!
            (V가 중간에 "A" → "B" → "A"로 변했지만 감지 못함)

단순 값 카운터에서의 ABA:
  counter = 0
  Thread A: expected=0, 계산 중...
  Thread B: counter를 0→1→0으로 변경
  Thread A: CAS(0, 1) 성공 → 논리적 문제 없음 (단순 카운터는 중간 상태 무관)

Lock-Free 스택에서의 ABA (실제 버그):
  초기: top → Node(A) → Node(B) → null
  
  Thread A: expected = Node(A), next 계산 = Node(B)
  (스택에서 A를 pop하려고 준비)
  
  Thread B 개입:
    pop: Node(A) 제거 → top = Node(B)
    pop: Node(B) 제거 → top = null
    push: Node(A) 재삽입 (같은 메모리 주소 재사용 가능!)
    → top = Node(A) → null (B는 이미 해제됨!)
  
  Thread A 재개:
    CAS(top, Node(A), Node(B)) → 성공! (포인터가 같음)
    top = Node(B) → 하지만 Node(B)는 이미 해제된 메모리!
    → use-after-free, 메모리 손상, 크래시

Java의 GC가 부분적으로 방지:
  GC 덕분에 참조가 유효한 동안 메모리 해제 안 됨
  하지만 논리적 ABA는 여전히 발생 가능
  (같은 객체가 다른 의미로 재사용되는 경우)
```

### 4. ABA 해결 방법: 버전 스탬프

```
AtomicStampedReference<V>:
  (참조, 스탬프) 쌍을 64비트에 pack하여 원자적으로 관리
  스탬프 = 단조 증가 버전 카운터 (변경마다 +1)

  compareAndSet(expectedRef, newRef, expectedStamp, newStamp):
    if (ref == expectedRef && stamp == expectedStamp) {
        ref = newRef;
        stamp = newStamp;
        return true;
    }
    return false;

  ABA 방지 원리:
    A(stamp=0) → B(stamp=1) → A(stamp=2)
    스레드 A: expected=(A, stamp=0) → CAS 실패 (stamp가 2로 변함)
    → ABA 감지 성공

내부 구현 (Java 8+):
  class AtomicStampedReference<V> {
      private volatile Pair<V> pair;  // (reference, stamp) 쌍

      private static class Pair<T> {
          final T reference;
          final int stamp;
          // 64비트에 pack (힙 객체이므로 참조 1개)
      }
  }

  trade-off:
    Pair 객체 생성 = GC 압박
    쓰기마다 새 Pair 객체 생성
    읽기는 volatile 읽기 1회 (빠름)

AtomicMarkableReference:
  (참조, boolean 마크) 쌍
  "삭제 예정" 마킹용 (ConcurrentSkipListMap 내부 사용)
  스탬프 대신 1비트 플래그 → GC overhead 동일
```

---

## 💻 실전 실험

### 실험 1: PrintAssembly로 LOCK CMPXCHG 확인

```bash
cat > CasAssembly.java << 'EOF'
import java.util.concurrent.atomic.AtomicInteger;

public class CasAssembly {
    static AtomicInteger counter = new AtomicInteger(0);

    public static void increment() {
        counter.incrementAndGet();
    }

    public static void main(String[] args) {
        for (int i = 0; i < 200_000; i++) increment();
        System.out.println(counter.get());
    }
}
EOF

javac CasAssembly.java
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintAssembly \
     -XX:CompileOnly=CasAssembly.increment \
     CasAssembly 2>&1 | grep -E "(lock|cmpxchg|xadd)" | head -10

# 예상 출력:
# lock cmpxchg DWORD PTR [rsi+0xc],ecx  ← CAS 명령어
# 또는
# lock xadd  DWORD PTR [rsi+0xc],r11d   ← 최적화된 fetch-and-add
```

### 실험 2: CAS 경쟁 수준별 처리량 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class CasContentionBenchmark {

    AtomicLong shared = new AtomicLong(0);

    @Benchmark @Threads(1)
    public long cas_1thread() { return shared.incrementAndGet(); }

    @Benchmark @Threads(4)
    public long cas_4threads() { return shared.incrementAndGet(); }

    @Benchmark @Threads(8)
    public long cas_8threads() { return shared.incrementAndGet(); }

    @Benchmark @Threads(16)
    public long cas_16threads() { return shared.incrementAndGet(); }

    @Benchmark @Threads(32)
    public long cas_32threads() { return shared.incrementAndGet(); }
}
// 결과: 스레드 증가에 따라 처리량이 역전되는 지점 관찰
// → LongAdder의 필요성을 수치로 확인
```

### 실험 3: ABA 문제 재현 (Lock-Free 스택)

```java
import java.util.concurrent.atomic.AtomicReference;

public class AbaDemo {
    static class Node {
        final int value;
        Node next;
        Node(int v, Node n) { value = v; next = n; }
        public String toString() { return "Node(" + value + ")"; }
    }

    static AtomicReference<Node> top = new AtomicReference<>(null);

    static void push(int v) {
        Node n = new Node(v, null);
        Node cur;
        do {
            cur = top.get();
            n.next = cur;
        } while (!top.compareAndSet(cur, n));
    }

    static Node pop() {
        Node cur, next;
        do {
            cur = top.get();
            if (cur == null) return null;
            next = cur.next;
        } while (!top.compareAndSet(cur, next));
        return cur;
    }

    public static void main(String[] args) throws InterruptedException {
        // ABA 시나리오 시뮬레이션
        push(1); push(2); push(3);  // 스택: 3→2→1
        System.out.println("초기: top=" + top.get());

        Node expected = top.get();  // Thread A: Node(3) 읽음, 잠깐 멈춤

        // Thread B: 3을 pop하고, 2를 pop하고, 3을 다시 push
        pop(); pop(); push(3);
        System.out.println("B 개입 후: top=" + top.get() + " (value=3이지만 next=null)");

        // Thread A: CAS 성공 (top이 아직 Node(3)처럼 보임)
        Node newTop = expected.next;  // 이미 해제된 Node(2)를 next로 설정
        boolean success = top.compareAndSet(expected, newTop);
        System.out.println("Thread A CAS: " + success);
        System.out.println("결과 top: " + top.get());
        // top.next = null이어야 하는데 dangling pointer 상태
    }
}
```

---

## 📊 성능/비용 비교

```
CAS vs synchronized 비용 (단일 변수 업데이트):

스레드 수    | CAS (AtomicLong) | synchronized | LongAdder
───────────┼──────────────────┼─────────────┼──────────────
1           | ~25ns            | ~20ns        | ~20ns
2           | ~30ns            | ~40ns        | ~22ns
4           | ~50ns            | ~80ns        | ~23ns
8           | ~100ns           | ~150ns       | ~24ns
16          | ~300ns           | ~250ns       | ~25ns
32          | ~800ns           | ~350ns       | ~26ns
64          | ~2000ns          | ~400ns       | ~27ns

CAS: 경쟁 심화 → 재시도 폭증 → 처리량 급락
synchronized: OS Mutex로 공정 대기 → 경쟁 수 증가에도 선형 저하
LongAdder: Cell 분산으로 경쟁 거의 없음 → 거의 일정

LOCK CMPXCHG 비용 (x86):
  캐시 히트, 경쟁 없음: ~20~40 사이클
  캐시 미스:            ~200 사이클
  같은 캐시 라인 경쟁: ~100~500 사이클 (MESI Invalidate 대기)
```

---

## ⚖️ 트레이드오프

```
CAS vs Lock 선택 기준:

CAS (Lock-Free) 적합:
  단일 변수 업데이트 (카운터, 플래그, 참조 교체)
  낮은~중간 경쟁 환경 (스레드 수 < CPU 코어 수 정도)
  진행 보장(Lock-Free Progress): 한 스레드의 지연이 다른 스레드를 막지 않음
  OS 스케줄링 불확실성에 강함 (스레드가 언제 선점돼도 전체가 멈추지 않음)

Lock 적합:
  복합 상태 업데이트 (여러 변수 일관되게 변경)
  고경쟁 환경 (CAS 실패율 > 50% 수준)
  공정성 요구 (기아 방지)
  임계 구역이 길 때 (CAS 스핀이 CPU 낭비)

ABA 고려 사항:
  단순 숫자 카운터: ABA 무관 (논리적 영향 없음)
  참조 타입 포인터: ABA 위험 → AtomicStampedReference 필요
  Lock-Free 자료구조: 항상 ABA 가능성 검토

GC 언어(Java) vs 네이티브 언어의 차이:
  C/C++: ABA = use-after-free 위험 (Hazard Pointer, RCU 필요)
  Java: GC가 생존 객체 해제 방지 → 메모리 안전
        하지만 논리적 ABA(같은 객체 재사용)는 여전히 발생 가능
```

---

## 📌 핵심 정리

```
CAS / LOCK CMPXCHG 핵심:

CPU 구현:
  LOCK CMPXCHG: 캐시 라인 독점 → 원자적 비교+교체
  성공: ZF=1, 실패: ZF=0 + 현재 값을 EAX에 로드

CAS 루프:
  do { read → compute → CAS } while (실패)
  경쟁 없음: ~20~40 사이클
  고경쟁: CAS 실패 폭증 → 처리량 역전

ABA 문제:
  A → B → A 변경을 CAS가 감지 못함
  단순 값: 대개 무해
  참조/포인터: 논리적 버그 또는 메모리 손상
  해결: AtomicStampedReference (버전 스탬프 포함)

선택 기준:
  단일 변수, 낮은 경쟁 → AtomicXxx (CAS)
  단일 변수, 고경쟁   → LongAdder (Cell 분산)
  복합 상태           → synchronized / ReentrantLock
  참조 ABA 위험       → AtomicStampedReference
```

---

## 🤔 생각해볼 문제

**Q1.** `AtomicInteger.compareAndSet(expected, newValue)`가 실패했을 때 현재 값은 어떻게 알 수 있는가? 실패 후 최신 값을 읽는 올바른 방법은?

<details>
<summary>해설 보기</summary>

`compareAndSet()`이 실패하면 false를 반환하고, `CMPXCHG` 명령어는 현재 메모리 값을 EAX 레지스터에 로드한다. JVM 수준에서는 실패 시 현재 값을 직접 반환하지 않는다.

최신 값을 얻으려면 `get()`을 다시 호출해야 한다:
```java
int current;
do {
    current = counter.get();          // 최신 값 읽기
    int next = current + 1;
} while (!counter.compareAndSet(current, next));
```

`getAndIncrement()`처럼 "이전 값을 반환"하는 메서드들은 내부적으로 `compareAndExchange()`(Java 9+)를 사용하여 실패 시 현재 값을 직접 반환받는다. 이 방식이 별도 `get()` 호출보다 한 번의 메모리 접근을 절약한다.

</details>

---

**Q2.** `LOCK CMPXCHG`는 "캐시 라인 락"을 사용한다고 했다. 그렇다면 두 변수가 같은 캐시 라인에 있을 때 서로 다른 변수에 대한 CAS가 상호 차단되는가?

<details>
<summary>해설 보기</summary>

그렇다. `LOCK` 접두사는 해당 캐시 라인 전체를 독점한다. 같은 64바이트 캐시 라인에 있는 두 변수 A와 B에 대해 Thread1이 A에 `LOCK CMPXCHG`를 실행하는 동안, Thread2의 B에 대한 `LOCK CMPXCHG`는 차단된다.

이것이 False Sharing의 또 다른 측면이다. 논리적으로 독립된 두 `AtomicInteger`가 같은 캐시 라인에 있으면 서로의 CAS 연산이 직렬화되어 성능이 저하된다.

`LongAdder`의 `Cell` 배열이 `@Contended`로 패딩되어 각 Cell이 별도 캐시 라인에 위치하는 이유다. Cell끼리 캐시 라인을 공유하면 분산 효과가 사라진다.

</details>

---

**Q3.** CAS는 진행 보장(Lock-Free Progress)을 제공한다고 했다. 하지만 라이브락이 발생할 수 있다고도 했다. 이 둘은 모순이 아닌가?

<details>
<summary>해설 보기</summary>

모순이 아니다. "Lock-Free"의 정확한 의미는 "시스템 전체 수준에서 항상 적어도 하나의 스레드가 진행한다"이다. 즉, 일부 스레드가 계속 실패하더라도 다른 누군가는 반드시 성공한다.

라이브락 상황에서도:
- 모든 스레드가 동시에 CAS를 시도하면 그 중 정확히 하나만 성공한다
- Lock-Free 정의상 "전체가 막히는 일은 없다"
- 하지만 특정 스레드가 계속 실패할 수 있다 (기아, Starvation)

엄밀히 말해 "Starvation-Free"(기아 없음)는 더 강한 보장으로, CAS 루프가 제공하지 않는다. 극단적 경쟁에서 특정 스레드가 수백 번 CAS 실패를 경험할 수 있다. Fair Lock(`ReentrantLock(true)`)은 Starvation-Free이지만 Lock-Free는 아니다. 이 두 속성은 다른 차원의 보장이다.

</details>

---

<div align="center">

**[⬅️ 이전: Ch3 락 성능 비교](../lock-internals/07-lock-performance-benchmark.md)** | **[홈으로 🏠](../README.md)** | **[다음: AtomicInteger/VarHandle ➡️](./02-atomic-integer-varhandle.md)**

</div>
