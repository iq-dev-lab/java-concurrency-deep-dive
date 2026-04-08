# happens-before 관계 — 8가지 규칙과 전이적 특성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- happens-before가 정확히 무엇을 보장하는가?
- 8가지 happens-before 규칙은 각각 어떤 상황에 적용되는가?
- 전이적 특성이란 무엇이고, 이를 어떻게 활용하는가?
- volatile 쓰기 이전의 일반 변수 쓰기가 왜 보이는가?
- happens-before가 없으면 어떤 값을 볼 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"가시성이 보장된다"는 말의 정확한 의미는 "happens-before 관계가 성립한다"는 것이다. 이 관계가 없으면 JMM은 어떤 값을 볼지 보장하지 않는다. `volatile`, `synchronized`, `Thread.start()`/`join()`이 동시성 안전성을 제공하는 이유가 모두 happens-before 규칙으로 설명된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수: 프로그램 실행 순서 = happens-before라고 오해

// 스레드 A가 먼저 실행된다고 가정 (하지만 이것은 보장이 아님)
int data = 0;
boolean flag = false;

// Thread A
data = 100;
flag = true;

// Thread B
if (flag) {
    System.out.println(data);  // 100이 보장되는가?
}

오해: "A가 먼저 실행됐으니 data=100이 보인다"
현실: flag도, data도 volatile이 아니면
      Thread B는 flag=true를 봐도 data=0을 볼 수 있음
      (재정렬 또는 캐시로 인해)

또 다른 실수: synchronized만 있으면 다른 블록도 동기화된다고 착각
// Thread A
synchronized (lock) { data = 100; }

// Thread B
// synchronized 없이 data 읽기
System.out.println(data);  // 100 보장 안 됨!
→ B도 같은 lock으로 synchronized 블록에서 읽어야 happens-before 성립
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```
happens-before 체인을 의식하며 코드 작성:

방법 1: volatile로 happens-before 체인 구성
  volatile boolean flag = false;

  // Thread A
  data = 100;    ←─ 프로그램 순서 규칙 (A의 작업)
  flag = true;   ←─ volatile 쓰기

  // Thread B
  if (flag) {    ←─ volatile 읽기: flag 쓰기 happens-before
      // 전이: data = 100 도 보임 (프로그램 순서 + volatile 전이)
      System.out.println(data);  // 100 보장
  }

방법 2: synchronized로 happens-before 구성
  Object lock = new Object();

  // Thread A
  synchronized (lock) { data = 100; }  // monitor unlock

  // Thread B
  synchronized (lock) { System.out.println(data); }  // monitor lock
  // 모니터 unlock →HB→ monitor lock → data = 100 보임

방법 3: Thread.join()으로 happens-before
  Thread worker = new Thread(() -> { data = 100; });
  worker.start();
  worker.join();  // join 후 worker의 모든 작업이 현재 스레드에 보임
  System.out.println(data);  // 100 보장
```

---

## 🔬 내부 동작 원리

### 1. happens-before의 정확한 정의

```
happens-before (→HB→ 로 표기):

정의:
  작업 A →HB→ 작업 B 라면:
  "A의 모든 결과(메모리 쓰기)가 B에게 반드시 보임"
  "B는 A 이전의 상태를 절대 볼 수 없음"

중요: happens-before는 실행 순서가 아님
  A →HB→ B 이더라도 A가 실제로 B보다 먼저 실행되는 것을 강제하지 않음
  단, B가 A의 영향을 받는 값을 읽을 때 A의 결과를 봄을 보장

happens-before가 없을 때 JMM의 보장:
  → "어떤 값이든 볼 수 있음" (특정 값을 보장하지 않음)
  → 컴파일러/CPU가 최적화로 어떤 값이 나타나도 스펙 위반이 아님
  → 실무에서 이것이 "동시성 버그"

happens-before ≠ 타임스탬프:
  물리적 시간 순서와 무관
  논리적 인과관계(causality)를 정의
```

### 2. 8가지 happens-before 규칙

```
규칙 1: 프로그램 순서 규칙 (Program Order Rule)
  동일 스레드 내에서, 앞의 작업 →HB→ 뒤의 작업
  Thread A에서: x = 1 →HB→ y = 2 →HB→ z = 3
  (단, 이 순서가 다른 스레드에게 보인다는 보장은 아님)

규칙 2: 모니터 락 규칙 (Monitor Lock Rule)
  synchronized 블록의 unlock →HB→ 이후의 lock
  
  Thread A: synchronized(lock) { x = 1; }  ← unlock
  Thread B: synchronized(lock) { print(x); }  ← lock
  unlock →HB→ lock → x = 1이 print에 보임

규칙 3: volatile 변수 규칙 (Volatile Variable Rule)
  volatile 변수의 쓰기 →HB→ 이후의 해당 변수 읽기
  
  Thread A: volatileVar = 1;   ← volatile 쓰기
  Thread B: int v = volatileVar;  ← volatile 읽기 (A의 쓰기가 보임)

규칙 4: 스레드 시작 규칙 (Thread Start Rule)
  Thread.start() →HB→ 시작된 스레드의 모든 작업
  
  x = 1;
  thread.start();  ← start() 호출
  → 새 스레드 내부: x = 1이 보임 (start()가 happens-before)

규칙 5: 스레드 종료 규칙 (Thread Termination Rule)
  스레드의 모든 작업 →HB→ Thread.join() 반환
  
  worker.join();   ← join이 반환되면
  → worker의 모든 작업이 현재 스레드에 보임

규칙 6: 스레드 인터럽트 규칙 (Thread Interruption Rule)
  Thread.interrupt() 호출 →HB→ interrupted 스레드의 인터럽트 감지
  
  t.interrupt();   ← 인터럽트 설정
  → t.isInterrupted() 또는 InterruptedException 발생 시 위 작업이 보임

규칙 7: 객체 finalizer 규칙 (Finalizer Rule)
  객체 생성자 완료 →HB→ finalize() 시작
  (finalize는 deprecated, 참고만)

규칙 8: 전이성 (Transitivity)
  A →HB→ B 이고 B →HB→ C 이면 A →HB→ C
  → 이것이 happens-before 체인을 구성하는 핵심 규칙
```

### 3. 전이적 특성과 happens-before 체인

```
전이성의 강력함:
  volatile 하나로 일반 변수까지 보장하는 원리

예시:
  int data = 0;           // 일반 변수
  volatile boolean flag = false;  // volatile 변수

  Thread A:
    data = 42;    ... (1) [프로그램 순서로 (2)보다 먼저]
    flag = true;  ... (2) [volatile 쓰기]

  Thread B:
    if (flag) {   ... (3) [volatile 읽기]
        print(data);  ... (4)
    }

happens-before 체인:
  (1) →HB→ (2): 프로그램 순서 규칙 (같은 스레드)
  (2) →HB→ (3): volatile 변수 규칙 (volatile 쓰기→읽기)
  (3) →HB→ (4): 프로그램 순서 규칙 (같은 스레드)
  
  전이성 적용:
  (1) →HB→ (2) →HB→ (3) →HB→ (4)
  따라서: (1) →HB→ (4)
  
  결론: Thread B의 print(data)는 data = 42가 보임 ✅

이것이 volatile 쓰기 "이전의" 일반 쓰기도 보이는 이유:
  volatile 쓰기 = 메모리 펜스 삽입
  펜스 이전의 모든 쓰기가 Store Buffer에서 플러시됨
  → 일반 변수의 쓰기도 함께 전파됨

CountDownLatch를 통한 happens-before:
  CountDownLatch latch = new CountDownLatch(1);
  
  // Thread A
  prepareData();    // data 준비
  latch.countDown();  // latch 내부: volatile 쓰기
  
  // Thread B
  latch.await();    // latch 내부: volatile 읽기
  useData();        // prepareData()의 결과가 보임
  
  CountDownLatch 내부 AQS가 volatile state를 사용하므로
  happens-before 체인 자동 구성
```

### 4. happens-before가 없는 데이터 레이스

```
데이터 레이스 (Data Race):
  두 스레드가 같은 변수에 접근하고
  적어도 하나가 쓰기이며
  happens-before 관계가 없을 때

데이터 레이스 = JMM 기준으로 undefined behavior에 가까움
  어떤 값이든 나타날 수 있음
  값 파괴(tearing)가 발생할 수 있음 (long/double의 경우)
  (단, byte/int 등은 원자적 읽기/쓰기 보장)

long/double의 Word Tearing:
  32비트 JVM에서 long(64비트) 쓰기는 2번의 32비트 쓰기로 나뉨
  Thread A: long x = 0xFFFF_FFFF_FFFF_FFFFL (2번에 나눠 씀)
  Thread B: 중간에 읽으면: 0xFFFF_FFFF_0000_0000L (절반만 새 값)
  → volatile long 또는 AtomicLong으로 방지

데이터 레이스 탐지:
  ThreadSanitizer (TSan): C/C++ 도구지만 개념 참고
  Java: -XX:+CheckJNIEnvAlignment (제한적)
  실용적: 코드 리뷰 + 정적 분석 도구 (SpotBugs, ErrorProne)
```

---

## 💻 실전 실험

### 실험 1: happens-before 체인 검증

```java
import java.util.concurrent.CountDownLatch;

public class HappensBeforeTest {
    private static int data = 0;
    private static volatile boolean flag = false;

    public static void main(String[] args) throws InterruptedException {
        int iterations = 10_000;
        int failures = 0;

        for (int i = 0; i < iterations; i++) {
            data = 0;
            flag = false;

            Thread writer = new Thread(() -> {
                data = 42;     // 일반 쓰기
                flag = true;   // volatile 쓰기 (happens-before 확립)
            });

            Thread reader = new Thread(() -> {
                while (!flag) { /* spin */ }  // volatile 읽기
                // happens-before 체인으로 data = 42 보장
                if (data != 42) {
                    System.out.println("실패! data = " + data);
                }
            });

            writer.start();
            reader.start();
            writer.join();
            reader.join();
        }
        System.out.println("테스트 완료. 실패 없음 (volatile happens-before 보장)");
    }
}
```

### 실험 2: Thread.start()와 join() happens-before 확인

```java
public class ThreadStartJoinHB {
    private static int before = 0;
    private static int during = 0;
    private static int after = 0;

    public static void main(String[] args) throws InterruptedException {
        before = 10;  // start() 이전 쓰기

        Thread t = new Thread(() -> {
            // Thread.start() →HB→ 이 실행 지점
            // before = 10이 보임 (규칙 4)
            System.out.println("스레드 내부 before = " + before);  // 반드시 10
            during = 20;  // 스레드 내부 쓰기
        });

        t.start();
        t.join();  // 스레드 종료 →HB→ join 반환 (규칙 5)

        // join 이후: during = 20이 보임
        System.out.println("join 후 during = " + during);  // 반드시 20
    }
}
```

### 실험 3: CountDownLatch happens-before 체인

```java
import java.util.concurrent.CountDownLatch;

public class LatchHappensBefore {
    private static int[] data = new int[1000];
    private static CountDownLatch ready = new CountDownLatch(1);

    public static void main(String[] args) throws InterruptedException {
        Thread producer = new Thread(() -> {
            // 데이터 준비 (일반 배열 쓰기)
            for (int i = 0; i < data.length; i++) {
                data[i] = i * 2;
            }
            ready.countDown();  // AQS volatile 쓰기 → happens-before 확립
        });

        Thread consumer = new Thread(() -> {
            try {
                ready.await();  // AQS volatile 읽기
            } catch (InterruptedException e) {}

            // happens-before 체인으로 data[] 전부 보임
            int sum = 0;
            for (int v : data) sum += v;
            System.out.println("sum = " + sum);  // 항상 올바른 값
        });

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

---

## 📊 성능/비용 비교

```
happens-before 확립 방법별 비용 비교:

방법              | happens-before 범위     | 비용   | 비고
─────────────────┼────────────────────────┼───────┼──────────────────────
volatile 쓰기     | 이전 모든 쓰기 → 이후 읽기   | 중간   | StoreLoad 펜스
volatile 읽기     | 이전 읽기 → 이후 쓰기       | 낮음   | x86에서 거의 무료
synchronized 획득 | 이전 unlock → 이후 모든    | 중간~높음| 경합 시 OS Mutex
synchronized 해제 | 이전 모든 → 이후 lock      | 중간   |
Thread.start()   | 이전 모든 → 새 스레드       | 높음   | OS 스레드 생성
Thread.join()    | 스레드 모든 → join 이후    | 중간   | 스레드 대기

규칙별 적용 빈도 (실무):

규칙              | 빈도    | 주 용도
─────────────────┼────────┼──────────────────────────────────
프로그램 순서       | 항상    | 같은 스레드 내 순서 보장
volatile 규칙     | 자주    | 상태 플래그, 초기화 완료 신호
모니터 락 규칙      | 매우 자주| synchronized 블록의 가시성
스레드 시작/종료    | 자주     | 스레드 간 데이터 전달
```

---

## ⚖️ 트레이드오프

```
happens-before 규칙 선택 트레이드오프:

volatile vs synchronized:
  volatile: 단일 변수 가시성 + 순서 보장, 원자성 없음
  synchronized: 가시성 + 원자성 + 상호 배제, 범위 제한 가능

  선택 기준:
    단순 플래그: volatile (경량)
    compound action: synchronized 또는 Atomic 클래스
    복합 상태 갱신: synchronized

CountDownLatch vs Thread.join():
  join(): 단일 스레드 완료 대기, 재사용 불가
  CountDownLatch: 여러 스레드의 특정 지점 완료 대기
  CyclicBarrier: 반복적인 동기화 지점

happens-before 체인 설계 주의:
  체인이 길어질수록 "의도치 않은 가시성 보장" 발생 가능
  A →HB→ B →HB→ C 에서 A가 C에게 보임은 의도된 것인지 확인
  불필요한 volatile/synchronized 추가는 체인을 복잡하게 만들고 성능 저하
```

---

## 📌 핵심 정리

```
happens-before 핵심:

정의:
  A →HB→ B: B는 A의 모든 쓰기를 볼 수 있음 (가시성 보장)
  관계 없으면: 어떤 값이든 나타날 수 있음

8가지 규칙:
  1. 프로그램 순서: 같은 스레드 내 순서
  2. 모니터 락: unlock →HB→ lock (synchronized)
  3. volatile: 쓰기 →HB→ 읽기
  4. 스레드 시작: start() →HB→ 새 스레드 내부
  5. 스레드 종료: 스레드 내부 →HB→ join()
  6. 인터럽트: interrupt() →HB→ 감지
  7. finalizer: 생성자 →HB→ finalize()
  8. 전이성: A→HB→B, B→HB→C ⟹ A→HB→C

핵심 패턴:
  volatile flag를 통해 일반 변수까지 보장:
    data = X; flag = true; (쓰기)
    if (flag) { data 사용; } (읽기)
    → 전이성으로 data = X가 보임

데이터 레이스:
  happens-before 없는 공유 변수 접근 = undefined
  long/double은 word tearing 발생 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `synchronized` 블록의 unlock이 다음 lock보다 happens-before라고 했다. 그렇다면 `ReentrantLock.unlock()`도 같은 보장을 제공하는가?

<details>
<summary>해설 보기</summary>

그렇다. `ReentrantLock`은 내부적으로 `AbstractQueuedSynchronizer`의 volatile `state` 필드를 사용한다. `unlock()`은 state에 volatile 쓰기를 하고, `lock()`은 state에 volatile 읽기를 한다.

JMM의 volatile 규칙에 의해 `unlock()` →HB→ `lock()`이 성립한다. 따라서 `ReentrantLock`도 `synchronized`와 동일한 가시성 보장을 제공한다.

실제로 JSR-133 스펙은 "잠금(lock)과 잠금 해제(unlock)는 happens-before 관계를 형성한다"고 규정하며, 이는 OS 수준의 mutex뿐 아니라 Java 수준의 모든 락 구현체에 적용된다.

</details>

---

**Q2.** `Thread.start()` 이후에 공유 변수에 쓰면, 새로 시작된 스레드가 그 값을 볼 수 있는가?

<details>
<summary>해설 보기</summary>

보장되지 않는다. `Thread.start()` →HB→ 새 스레드의 첫 번째 작업이 성립하는 것이지, `start()` 이후의 쓰기는 해당 happens-before 체인에 포함되지 않는다.

```java
Thread t = new Thread(() -> {
    System.out.println(data);  // 0이 나올 수 있음!
});
t.start();
data = 100;  // start() 이후 쓰기: 새 스레드에게 보장 안 됨
```

`start()` **이전**의 쓰기는 새 스레드에 보임이 보장된다. `start()` 이후의 쓰기는 별도의 happens-before 관계(volatile, synchronized 등)가 필요하다.

</details>

---

**Q3.** `ConcurrentHashMap.put()`이 완료된 후 다른 스레드에서 `get()`으로 읽으면 happens-before가 보장되는가?

<details>
<summary>해설 보기</summary>

그렇다. `ConcurrentHashMap`은 내부적으로 volatile 읽기/쓰기와 synchronized를 사용하여 happens-before를 보장한다.

Java API 문서에서 "Thread A가 Map에 키-값을 put()하는 모든 작업은 다른 스레드 B가 해당 키를 get()하는 작업보다 happens-before"임을 명시하고 있다.

하지만 "A의 put()이 B의 get()보다 물리적으로 먼저 실행됐음"을 보장하지는 않는다. B가 이미 get()을 호출했고 A가 아직 put()을 안 했다면 B는 null을 받는다. happens-before는 A의 put()이 성공한 경우, B가 그 이후 get()을 호출할 때 반드시 그 값을 본다는 것이다.

이것이 일반 `HashMap`을 멀티스레드 환경에서 쓰면 안 되는 이유다. `HashMap`은 내부에 volatile/synchronized가 없어 happens-before를 보장하지 않는다.

</details>

---

<div align="center">

**[⬅️ 이전: JMM 주 메모리와 작업 메모리](./02-jmm-main-working-memory.md)** | **[홈으로 🏠](../README.md)** | **[다음: volatile 완전 분해 ➡️](./04-volatile-deep-dive.md)**

</div>
