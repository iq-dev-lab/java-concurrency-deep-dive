# Double-Checked Locking의 함정 — Java 5 이전과 이후

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 5 이전 DCL이 왜 깨지는가? (부분 초기화 문제)
- `volatile`을 추가하면 어떻게 수정되는가?
- Initialization-on-demand Holder 패턴은 왜 가장 안전한가?
- JVM 클래스 초기화 락이 보장하는 범위는 무엇인가?
- Java에서 안전한 싱글톤을 구현하는 방법들을 비교하면?

---

## 🔍 왜 이 개념이 실무에서 중요한가

DCL(Double-Checked Locking)은 "성능을 위해 스마트하게 synchronized를 줄이자"는 의도로 자주 시도되는 패턴이다. 그러나 Java 5 이전(또는 `volatile` 없이)에는 JMM이 보장하지 않는 방식이다. 이 패턴이 왜 깨지는지 이해하면, JMM의 재정렬 문제와 객체 초기화의 가시성을 깊이 이해할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```java
// 잘못된 DCL (Java 5 이전, 또는 volatile 없는 버전)
class BrokenSingleton {
    private static BrokenSingleton instance;  // volatile 없음!

    public static BrokenSingleton getInstance() {
        if (instance == null) {               // 첫 번째 체크
            synchronized (BrokenSingleton.class) {
                if (instance == null) {       // 두 번째 체크
                    instance = new BrokenSingleton();  // 문제 발생!
                }
            }
        }
        return instance;  // 부분 초기화된 객체 반환 가능!
    }
}

// 표면적으로는 완벽해 보임:
// - 두 번째 체크로 중복 생성 방지
// - synchronized로 동시 접근 방지
// - 첫 번째 체크로 synchronized 비용 절약

// 하지만: instance = new BrokenSingleton()의 내부 과정이 재정렬될 수 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 방법 1: volatile DCL (Java 5+, JSR-133 이후 올바름)
class CorrectDCL {
    private static volatile CorrectDCL instance;  // volatile 필수!

    public static CorrectDCL getInstance() {
        if (instance == null) {
            synchronized (CorrectDCL.class) {
                if (instance == null) {
                    instance = new CorrectDCL();
                }
            }
        }
        return instance;
    }
}

// 방법 2: Initialization-on-demand Holder (가장 권장)
class HolderSingleton {
    private HolderSingleton() {}

    private static class Holder {
        static final HolderSingleton INSTANCE = new HolderSingleton();
        // 클래스 로딩 시 JVM이 초기화 보장 (happens-before)
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE;  // Holder 클래스 로딩 시 초기화
    }
}

// 방법 3: enum 싱글톤 (Effective Java 추천, 직렬화 안전)
enum EnumSingleton {
    INSTANCE;

    public void doSomething() { ... }
}
// 사용: EnumSingleton.INSTANCE.doSomething()
```

---

## 🔬 내부 동작 원리

### 1. 객체 생성 3단계와 재정렬

```
instance = new BrokenSingleton() 의 실제 3단계:

바이트코드 수준 분해:
  ① new BrokenSingleton  → 힙에 메모리 할당 (참조 생성, 모든 필드 = 기본값)
  ② invokespecial <init> → 생성자 실행 (필드 초기화)
  ③ putstatic instance   → instance 변수에 참조 저장

정상 실행 순서: ① → ② → ③

재정렬 발생 가능 순서: ① → ③ → ②
  JIT와 CPU가 ③을 ② 이전으로 이동 가능!
  → instance가 이미 참조를 가리키지만 생성자 실행 미완료

┌────────────────────────────────────────────────────────────────┐
│  재정렬 발생 시 두 스레드의 타임라인:                            │
│                                                               │
│  Thread A (getInstance 호출):                                 │
│    ① 힙에 메모리 할당 (BrokenSingleton 객체, 필드 모두 기본값)  │
│    ③ instance = 참조  ← 재정렬로 ② 이전에 실행!               │
│         ↕ (이 사이에 Thread B가 실행)                          │
│    ② 생성자 실행 (필드 초기화 완료)                            │
│                                                               │
│  Thread B (getInstance 호출):                                 │
│    if (instance == null) → instance != null!                 │
│    return instance  ← 생성자가 완료 안 된 객체 반환!           │
│                        필드 = 기본값 (0, null, false)         │
└────────────────────────────────────────────────────────────────┘

이것이 "부분 초기화된 객체(Partially Constructed Object)" 문제
```

### 2. volatile이 DCL을 수정하는 방법

```
volatile instance 선언 시:

  instance = new CorrectDCL(); 의 컴파일 결과:
    ① 힙에 메모리 할당
    ② 생성자 실행 (필드 초기화)
    [StoreStore 펜스]  ← volatile 쓰기 이전 삽입
    ③ instance = 참조 (volatile 쓰기)
    [StoreLoad 펜스]   ← volatile 쓰기 이후 삽입

  StoreStore 펜스의 역할:
    ①, ②의 모든 Store가 ③ 이전에 완료되도록 보장
    → ③(instance에 참조 저장) 이전에 생성자가 반드시 완료
    → 재정렬 불가: ① → ② → [펜스] → ③

  Thread B가 instance != null을 보면:
    volatile 읽기 →HB→ volatile 쓰기(③) →HB→ 생성자 완료(②)
    전이성: Thread B가 instance != null을 보는 시점에
            생성자 완료가 보장됨

  결론: volatile DCL은 Java 5+(JSR-133) 이후 올바름
```

### 3. Initialization-on-demand Holder 패턴 — 왜 안전한가

```
클래스 초기화 규칙 (JVM 스펙 §12.4):

  JVM은 클래스를 처음 사용될 때 초기화한다.
  (처음 사용 = 필드 접근, 메서드 호출, 객체 생성, 클래스 로딩 직접 요청)

  클래스 초기화는 JVM이 단 한 번, 스레드 안전하게 보장한다:
    ① 초기화 시작 전 클래스 레벨 초기화 락(LC, Locking-Context) 획득
    ② 다른 스레드가 같은 클래스를 초기화하려 하면 대기
    ③ 초기화 완료 후 LC 해제
    ④ LC 해제 →HB→ 다른 스레드의 LC 획득 (모니터 락 규칙)

  결과: static final 필드의 초기화는 모든 스레드에 올바르게 보임

HolderSingleton 동작:

  HolderSingleton.getInstance() 최초 호출 시:
    Holder 클래스가 처음 사용됨 → JVM 초기화 시작
    static final INSTANCE = new HolderSingleton() 실행
    (단 한 번, JVM이 스레드 안전하게 보장)
    초기화 완료 → LC 해제

  이후 모든 스레드:
    Holder 클래스는 이미 초기화됨 → 즉시 INSTANCE 반환
    JVM LC 해제 →HB→ INSTANCE 접근 (happens-before 보장)
    synchronized, volatile 불필요

  장점:
    lazy initialization (처음 사용 시 초기화)
    스레드 안전 (JVM 클래스 초기화 락)
    성능 (synchronized 오버헤드 없음)
    코드 단순
    직렬화 문제 없음 (final이 아닌 경우 주의)
```

### 4. 싱글톤 패턴 비교

```
방법 1: Eager Initialization
  class EagerSingleton {
      private static final EagerSingleton INSTANCE = new EagerSingleton();
      public static EagerSingleton getInstance() { return INSTANCE; }
  }
  장점: 단순, 스레드 안전
  단점: 클래스 로딩 시 즉시 생성 (lazy 아님)
  적합: 항상 사용되는 싱글톤

방법 2: synchronized getInstance()
  class SyncSingleton {
      private static SyncSingleton instance;
      public static synchronized SyncSingleton getInstance() {
          if (instance == null) instance = new SyncSingleton();
          return instance;
      }
  }
  장점: lazy, 안전
  단점: 매 호출마다 synchronized → 성능 저하
  적합: 호출 빈도가 낮을 때

방법 3: volatile DCL
  private static volatile VolDCL instance;
  // 이전 절 참조
  장점: lazy, 안전 (Java 5+), 성능 (초기화 후 락 없음)
  단점: 코드 복잡, volatile 쓰기 비용

방법 4: Holder 패턴 (권장)
  // 이전 절 참조
  장점: lazy, 안전, 성능, 코드 단순
  단점: 내부 클래스 사용 필요

방법 5: enum 싱글톤 (Effective Java 3rd 권장)
  enum EnumSingleton {
      INSTANCE;
      // 필드, 메서드 추가 가능
  }
  장점: 직렬화 완벽 처리, 리플렉션 공격 방어, 스레드 안전
  단점: 상속 불가, 다른 클래스를 extends 불가

Spring Context에서의 싱글톤:
  ApplicationContext의 싱글톤 빈은 컨테이너가 관리
  → 위 패턴 없이도 스레드 안전
  → 단, 빈 내부 상태(인스턴스 변수)는 별도 동기화 필요
```

---

## 💻 실전 실험

### 실험 1: 부분 초기화 객체 관찰 시뮬레이션

```java
// 실제로 부분 초기화 객체를 관찰하기는 매우 어려움 (타이밍 의존)
// 이론적 시뮬레이션
public class PartialInitSimulation {
    static class HeavyObject {
        final int a, b, c;
        final String data;

        HeavyObject() {
            // 생성자에서 복잡한 초기화 수행
            this.a = compute(1);
            this.b = compute(2);
            this.c = compute(3);
            this.data = "initialized";
        }

        private int compute(int n) {
            // 의도적 지연 (재정렬 발생 가능성 증가)
            return n * n;
        }
    }

    // volatile 없는 잘못된 DCL
    private static HeavyObject instance;

    public static HeavyObject getBroken() {
        if (instance == null) {
            synchronized (PartialInitSimulation.class) {
                if (instance == null) {
                    instance = new HeavyObject();  // 재정렬 위험
                }
            }
        }
        return instance;  // 부분 초기화 객체 반환 가능
    }

    // volatile DCL (올바름)
    private static volatile HeavyObject safeInstance;

    public static HeavyObject getSafe() {
        if (safeInstance == null) {
            synchronized (PartialInitSimulation.class) {
                if (safeInstance == null) {
                    safeInstance = new HeavyObject();  // volatile로 보호
                }
            }
        }
        return safeInstance;
    }
}
```

### 실험 2: Holder 패턴 성능 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(8)
public class SingletonBenchmark {

    // 방법 1: synchronized
    static class SyncSingleton {
        private SyncSingleton() {}
        private static SyncSingleton instance;
        public static synchronized SyncSingleton getInstance() {
            if (instance == null) instance = new SyncSingleton();
            return instance;
        }
    }

    // 방법 2: volatile DCL
    static class DCLSingleton {
        private DCLSingleton() {}
        private static volatile DCLSingleton instance;
        public static DCLSingleton getInstance() {
            if (instance == null) {
                synchronized (DCLSingleton.class) {
                    if (instance == null) instance = new DCLSingleton();
                }
            }
            return instance;
        }
    }

    // 방법 3: Holder
    static class HolderSingleton {
        private HolderSingleton() {}
        private static class Holder {
            static final HolderSingleton INSTANCE = new HolderSingleton();
        }
        public static HolderSingleton getInstance() { return Holder.INSTANCE; }
    }

    @Benchmark
    public Object syncSingleton() { return SyncSingleton.getInstance(); }

    @Benchmark
    public Object dclSingleton() { return DCLSingleton.getInstance(); }

    @Benchmark
    public Object holderSingleton() { return HolderSingleton.getInstance(); }
}
// 결과: holderSingleton ≈ dclSingleton >> syncSingleton
// Holder와 DCL은 초기화 후 락 없음 → 비슷한 성능
// syncSingleton은 매 호출마다 synchronized 비용
```

### 실험 3: jol-core로 객체 생성 과정 관찰

```java
// maven: org.openjdk.jol:jol-core:0.17
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.info.GraphLayout;

public class ObjectCreationTrace {
    static class Target {
        int x = 10;
        int y = 20;
        String name = "hello";
    }

    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(Target.class).toPrintable());
        // 출력: 각 필드의 오프셋과 크기
        // Object Header(12바이트) + x(4) + y(4) + name(4) = 24바이트

        Target t = new Target();
        System.out.println(GraphLayout.parseInstance(t).toFootprint());
        // 전체 객체 그래프의 메모리 사용량

        // 기본값 확인: 생성자 실행 전 모든 필드는 기본값
        // JVM은 new로 할당 시 zeroing: int=0, ref=null, boolean=false
    }
}
```

---

## 📊 성능/비용 비교

```
싱글톤 패턴별 성능 비교 (8스레드, JMH 기준):

패턴              | 처리량 (ops/ms) | 비고
─────────────────┼────────────────┼───────────────────────────────
Eager (필드 접근)  | 1,000,000+     | 락/체크 없음, 가장 빠름
Holder 패턴       | 900,000+       | 클래스 로딩 완료 후 락 없음
volatile DCL      | 800,000+       | volatile 읽기 비용 (미미)
synchronized      | 50,000~100,000 | 매 호출 synchronized 비용
enum 싱글톤       | 900,000+       | 내부적으로 Eager와 동일

초기화 후 동작 비교:
패턴              | 초기화 후 비용         | 추가 동기화
─────────────────┼──────────────────────┼──────────────────
Eager             | 필드 읽기만           | 없음
Holder            | 클래스 필드 읽기       | 없음
volatile DCL      | volatile 읽기 1회     | 없음 (null 체크)
synchronized      | synchronized 획득/해제 | 항상 발생
```

---

## ⚖️ 트레이드오프

```
각 방법의 선택 기준:

Eager Initialization:
  사용 조건: 싱글톤이 항상 사용됨, 생성 비용이 크지 않음
  선택 이유: 가장 단순하고 명확

Holder 패턴:
  사용 조건: lazy initialization 필요, JVM 기반 Java 환경
  선택 이유: 단순하고, 안전하고, 성능도 좋음
  → 특별한 이유 없으면 이것이 기본 선택

volatile DCL:
  사용 조건: 이미 DCL 코드가 있거나 외부 라이브러리 요구사항
  주의: Java 5+ 환경에서만 올바름 (구버전 JVM 주의)

enum 싱글톤:
  사용 조건: 직렬화/역직렬화 안전성 요구, 리플렉션 방어 필요
  선택 이유: Effective Java 권장, 가장 강한 보장

피해야 할 패턴:
  volatile 없는 DCL: 항상 잘못됨 (Java 5 이전/이후 무관)
  synchronized 메서드: 매 호출 비용, Holder로 대체 가능
```

---

## 📌 핵심 정리

```
DCL 핵심:

부분 초기화 문제:
  instance = new Obj() → ①메모리 할당 → ②생성자 → ③참조 저장
  JIT/CPU가 ① → ③ → ② 순서로 재정렬 가능
  Thread B가 ③과 ② 사이에 instance를 읽으면 → 부분 초기화 객체

volatile DCL 수정:
  volatile instance 선언
  → ③ 이전에 StoreStore 펜스 삽입
  → ② 생성자 완료 전에 ③ 실행 불가
  → Java 5+(JSR-133) 이후 올바름

Holder 패턴 (권장):
  static 내부 클래스의 static final 필드
  JVM 클래스 초기화 시 단 한 번, 스레드 안전하게 실행
  초기화 락 해제 →HB→ 이후 접근
  synchronized, volatile 불필요

우선순위:
  1. enum 싱글톤 (직렬화/리플렉션 안전)
  2. Holder 패턴 (단순, 안전, 성능)
  3. volatile DCL (기존 코드 호환)
  4. synchronized (단순하지만 성능 미흡)
  ✗ volatile 없는 DCL (항상 잘못됨)
```

---

## 🤔 생각해볼 문제

**Q1.** 리플렉션(Reflection)으로 `private` 생성자를 강제 호출하면 싱글톤이 깨진다. 이를 방어하는 방법은?

<details>
<summary>해설 보기</summary>

리플렉션으로 싱글톤을 깨는 공격:
```java
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);
Singleton second = c.newInstance();  // 두 번째 인스턴스 생성!
```

방어 방법:

1. **생성자에서 이미 생성된 경우 예외 발생**:
```java
private Singleton() {
    if (Holder.INSTANCE != null) {
        throw new IllegalStateException("Already instantiated");
    }
}
```
단점: Holder.INSTANCE 참조가 생성자보다 나중에 설정되면 타이밍 문제 가능

2. **enum 싱글톤**: 리플렉션으로 enum 인스턴스 생성 자체를 JVM이 거부한다. `EnumSingleton.class.newInstance()`는 항상 `IllegalArgumentException` 발생. 가장 강력한 방어.

3. **Java 9+ 모듈 시스템**: 모듈이 `exports`하지 않은 패키지의 클래스에 리플렉션 접근을 기본적으로 거부.

실무에서는 대부분 리플렉션 공격을 걱정할 필요가 없지만, 보안이 중요한 코드라면 `enum` 싱글톤이 가장 안전하다.

</details>

---

**Q2.** `volatile`이 추가되면 DCL이 올바르게 동작한다고 했다. 그렇다면 `final` 필드만 있는 객체는 `volatile` 없이도 DCL이 안전한가?

<details>
<summary>해설 보기</summary>

이론적으로는 `final` 필드만 있는 객체는 생성자 완료 후 모든 스레드에게 올바르게 보이는 것이 JMM의 "final field freezing" 규칙으로 보장된다.

그러나 DCL에서 `volatile` 없이도 안전한지는 **참조(instance 변수)의 가시성**이 별도로 문제다. 객체의 `final` 필드가 올바르게 보이더라도, `instance` 참조 자체가 부분 초기화 순간에 노출되는 문제(instance 포인터가 보이지만 아직 생성자가 실행 중)는 `final` 필드 규칙과 무관하다.

결론: **`final` 필드만 있어도 `volatile` DCL은 여전히 volatile이 필요하다**. 안전하게 하려면 Holder 패턴을 사용하거나 volatile을 추가한다. 단, "안전한 publication"(생성자 완료 후 참조를 volatile이나 synchronized를 통해 공개)의 경우에는 final 필드 규칙이 완전한 보호를 제공한다.

</details>

---

**Q3.** Spring `@Bean`으로 등록한 싱글톤은 위 패턴 없이도 스레드 안전한가? Spring은 내부적으로 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

Spring ApplicationContext가 관리하는 싱글톤 빈의 **생성과 등록** 과정은 스레드 안전하다. Spring은 빈 초기화 완료 후 내부 registry에 등록할 때 synchronized와 volatile을 활용하여 happens-before를 보장한다.

하지만 **빈 내부의 인스턴스 변수**는 Spring이 보호하지 않는다:

```java
@Service
class UserService {
    private List<User> cache = new ArrayList<>();  // 스레드 안전하지 않음!

    public void addUser(User u) {
        cache.add(u);  // 여러 스레드가 동시에 호출하면 race condition
    }
}
```

빈 자체(참조)는 안전하게 공유되지만, 빈이 가진 상태(인스턴스 변수)는 별도로 동기화가 필요하다. 싱글톤 빈에서는 인스턴스 변수에 가변 상태를 두지 않는 것이 권장 설계다. 상태가 필요하면 `ThreadLocal`, `prototype` 스코프, 또는 명시적 동기화를 사용한다.

</details>

---

<div align="center">

**[⬅️ 이전: 명령어 재정렬](./05-instruction-reordering.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — Object Header와 Mark Word ➡️](../lock-internals/01-object-header-mark-word.md)**

</div>
