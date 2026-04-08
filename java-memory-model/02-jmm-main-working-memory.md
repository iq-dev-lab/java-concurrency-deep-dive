# JMM — 주 메모리와 작업 메모리 추상화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JMM이 정의하는 주 메모리와 작업 메모리는 실제 하드웨어의 무엇에 대응하는가?
- JMM이 정의하는 8가지 원자적 연산은 무엇인가?
- JMM이 "허용하는 재정렬"과 "금지하는 재정렬"의 기준은 무엇인가?
- JMM이 없다면 Java 프로그램의 멀티스레드 동작은 어떻게 되는가?
- JMM이 플랫폼(x86, ARM)마다 다른 하드웨어를 어떻게 추상화하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`volatile`이나 `synchronized`가 "왜 이 규칙대로 동작하는가"를 JMM 없이는 설명할 수 없다. JMM은 Java 언어 사양(JLS)의 일부로, "Java 프로그램이 멀티스레드 환경에서 어떤 값을 볼 수 있고 없는지"를 공식적으로 정의한다. 이 추상화가 없으면 코드의 정확성을 논리적으로 보장할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수: "코드 순서대로 실행되겠지"라는 가정

// 스레드 A
data = 42;
ready = true;

// 스레드 B
if (ready) {
    assert data == 42;  // "당연히 42겠지?"
}

현실:
  JMM의 happens-before 관계 없이는
  data = 42와 ready = true의 실행 순서가 역전될 수 있음
  스레드 B가 ready = true를 봐도 data = 0을 볼 수 있음
  → volatile, synchronized 없이는 어떤 값을 볼지 보장 없음

또 다른 실수: JMM을 "캐시 문제"로만 이해
  "volatile을 쓰면 캐시를 우회해서 항상 최신 값을 읽는다"
  → 틀림. volatile은 캐시를 우회하지 않음
  → volatile은 happens-before 관계를 통해 가시성을 보장하는 추상화
  → 하드웨어 레벨에서는 메모리 펜스로 구현되지만 JMM 수준에서는 추상화된 규칙
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// happens-before를 통한 올바른 가시성 보장

class DataSharing {
    private int data = 0;
    private volatile boolean ready = false;  // volatile로 happens-before 확립

    // 스레드 A
    public void write() {
        data = 42;        // 1번
        ready = true;     // 2번: volatile 쓰기
        // happens-before: 1번은 2번 이전에 실행됨 (프로그램 순서 규칙)
        // happens-before: 2번(volatile 쓰기)이 3번(volatile 읽기)보다 먼저
    }

    // 스레드 B
    public void read() {
        if (ready) {      // 3번: volatile 읽기
            // volatile 읽기 → volatile 쓰기 happens-before → data = 42 가 보장됨
            System.out.println(data);  // 4번: 반드시 42 출력
        }
    }
}

// 전이적 happens-before:
// 1 →(프로그램 순서)→ 2 →(volatile)→ 3 →(프로그램 순서)→ 4
// → 1이 4보다 happens-before
// → data = 42가 System.out.println(data)에 가시적
```

---

## 🔬 내부 동작 원리

### 1. JMM의 추상화 모델

```
JMM(Java Memory Model)이 정의하는 두 가지 메모리 영역:

주 메모리 (Main Memory):
  모든 스레드가 공유하는 메모리 영역
  인스턴스 변수, 정적 변수, 배열 원소가 위치
  로컬 변수, 메서드 매개변수, 예외 처리 매개변수는 해당 없음
  하드웨어의 RAM에 대응 (추상적 개념)

작업 메모리 (Working Memory):
  각 스레드가 독립적으로 가지는 메모리 영역
  주 메모리에서 읽어온 변수의 복사본(copy)을 보관
  스레드의 실제 연산은 작업 메모리의 복사본에 대해 수행
  하드웨어의 레지스터 + L1/L2 캐시에 대응 (추상적 개념)

┌──────────────────────────────────────────────────────────────┐
│                     주 메모리 (Main Memory)                    │
│    변수 x = 0   변수 y = 0   변수 z = 0  ...                    │
└──────────────────┬──────────────────────────┬────────────────┘
                   │ read/write               │ read/write
    ┌──────────────┴──────────┐  ┌────────────┴────────────┐
    │  Thread A 작업 메모리      │  │  Thread B 작업 메모리      │
    │  x의 복사본: 0            │  │  x의 복사본: 0             │
    │  y의 복사본: 0            │  │  y의 복사본: 0             │
    └─────────────────────────┘  └─────────────────────────┘

중요: 두 스레드의 작업 메모리는 서로 직접 접근 불가
  → 반드시 주 메모리를 통해 소통
  → 이 제약이 JMM 규칙의 근거
```

### 2. JMM이 정의하는 8가지 원자적 연산

```
JMM JSR-133 스펙이 정의하는 8가지 기본 연산:

주 메모리 → 작업 메모리 방향:
  read:   주 메모리에서 변수의 값을 읽어 전송 준비 (전송 과정의 첫 단계)
  load:   read로 전송받은 값을 작업 메모리의 변수 복사본에 저장

작업 메모리 내부:
  use:    작업 메모리의 변수 복사본을 읽어 실행 엔진에 전달 (실제 연산)
  assign: 실행 엔진에서 받은 값을 작업 메모리의 변수 복사본에 저장

작업 메모리 → 주 메모리 방향:
  store:  작업 메모리의 변수 복사본을 주 메모리에 전송 준비 (전송 과정의 첫 단계)
  write:  store로 전송받은 값을 주 메모리의 변수에 저장

동기화 관련:
  lock:   주 메모리의 변수를 특정 스레드가 독점 사용하도록 잠금
  unlock: lock으로 잠긴 변수를 해제하여 다른 스레드가 접근 가능하게

변수 읽기 전체 흐름:
  주 메모리 --read--> [전송 중] --load--> 작업 메모리 복사본 --use--> 실행 엔진

변수 쓰기 전체 흐름:
  실행 엔진 --assign--> 작업 메모리 복사본 --store--> [전송 중] --write--> 주 메모리

8가지 연산 규칙:
  ① read와 load, store와 write는 반드시 쌍으로 실행
  ② 작업 메모리에서 use되기 전에 반드시 load가 선행
  ③ assign 없이 store 불가 (assign된 내용만 write 가능)
  ④ volatile 변수의 use 전에 반드시 load (캐시 사용 금지)
  ⑤ lock 후 작업 메모리 초기화 필요 (최신값을 주 메모리에서 다시 read)
  ⑥ unlock 전에 주 메모리에 write 필요 (store + write)
```

### 3. JMM이 허용하는 재정렬과 금지하는 재정렬

```
재정렬(Reordering) 두 가지 수준:

컴파일러 재정렬:
  JIT가 성능을 위해 명령어 순서를 변경
  단일 스레드 의미론을 깨지 않는 범위에서만 허용
  volatile, synchronized 존재 시 재정렬 제한

CPU 재정렬:
  CPU의 Out-of-Order Execution
  Store Buffer, Load Buffer로 인한 비가시성

JMM이 허용하는 재정렬 (단일 스레드에서 결과가 같을 때):
  int a = 1;   // 독립적인 두 쓰기는
  int b = 2;   // 순서가 바뀌어도 결과 동일
  → a와 b가 독립적이면 JMM은 순서 변경 허용

JMM이 금지하는 재정렬 (happens-before 관계에 영향):

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  재정렬 금지 매트릭스                                                      
  │                                                                              
  │  이전 연산\이후 연산       | 일반 읽기 | 일반 쓰기  | volatile 읽기 | volatile 쓰기
  │  ─────────────────────┼─────────┼──────────┼──────────────┼──────────────
  │  일반 읽기              |         |          | ❌ 금지       | ❌ 금지
  │  일반 쓰기              |         |          | ❌ 금지       | ❌ 금지
  │  volatile 읽기         | ❌ 금지  | ❌ 금지   | ❌ 금지       | ❌ 금지
  │  volatile 쓰기         |         |          | ❌ 금지       | ❌ 금지
  └──────────────────────────────────────────────────────────────────────────────┘

  핵심:
  volatile 쓰기 이후의 volatile 읽기는 절대 앞으로 이동 불가
  volatile 읽기 이후의 모든 연산은 뒤로 이동 불가
  → 이것이 happens-before 보장의 재정렬 측면에서의 의미
```

### 4. 플랫폼별 JMM 구현 차이

```
JMM은 플랫폼 독립적인 추상화지만, 구현은 플랫폼별로 다름

x86 (Intel/AMD) — TSO 모델:
  허용 재정렬: Store→Load (쓰기 후 읽기 순서 변경)
  금지 재정렬: Load→Load, Load→Store, Store→Store
  
  volatile 쓰기 구현:
    mov [addr], val          ← 실제 쓰기
    lock addl $0x0, (%rsp)  ← StoreLoad 펜스 (MFENCE 대안)
  
  volatile 읽기 구현:
    mov eax, [addr]          ← 일반 읽기와 동일 (x86에서 LoadLoad/LoadStore 펜스 불필요)
  
  → x86에서 volatile 읽기는 사실상 무료

ARM — 약한 메모리 모델:
  더 많은 재정렬 허용
  volatile 쓰기: stlr 명령어 (Store-Release)
  volatile 읽기: ldar 명령어 (Load-Acquire)
  → ARM에서 volatile의 비용이 더 큼

JVM이 이를 처리하는 방법:
  각 플랫폼의 JIT가 JMM 스펙을 만족하는 최적의 펜스 삽입
  → 개발자는 JMM 추상화 수준에서만 생각하면 됨
  → 하드웨어별 세부 구현은 JVM이 담당
```

---

## 💻 실전 실험

### 실험 1: JMM 재정렬 효과 관찰 (litmus test)

```java
// 클래식 멀티스레드 재정렬 테스트
// x86에서는 StoreLoad 재정렬이 발생 가능
public class ReorderingTest {
    static int x = 0, y = 0;
    static int r1 = 0, r2 = 0;

    public static void main(String[] args) throws InterruptedException {
        int reorderCount = 0;

        for (int i = 0; i < 1_000_000; i++) {
            x = 0; y = 0;

            Thread t1 = new Thread(() -> {
                x = 1;   // 쓰기
                r1 = y;  // 읽기
            });
            Thread t2 = new Thread(() -> {
                y = 1;   // 쓰기
                r2 = x;  // 읽기
            });

            t1.start(); t2.start();
            t1.join(); t2.join();

            // r1=0, r2=0이면 재정렬 발생
            // (x=1이 r2=x 전에 실행됐다면 r2=1, y=1이 r1=y 전이면 r1=1)
            if (r1 == 0 && r2 == 0) {
                reorderCount++;
            }
        }
        System.out.println("재정렬 발생 횟수: " + reorderCount + "/1,000,000");
        // 0이 나와도 재정렬이 없는 게 아닐 수 있음 (타이밍 문제)
        // volatile로 x, y를 선언하면 재정렬 현저히 감소
    }
}
```

### 실험 2: 8가지 원자적 연산을 코드로 추적

```java
// 개념 시뮬레이션: JMM 연산 흐름
public class JmmOperationTrace {
    static int sharedVar = 10;  // 주 메모리

    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> {
            // 1. read + load: 주 메모리에서 작업 메모리로
            int localCopy = sharedVar;  // load(read(sharedVar))
            System.out.println("A: 읽음 = " + localCopy);  // use

            // 2. assign: 실행 엔진 → 작업 메모리
            localCopy = localCopy * 2;  // assign

            // 3. store + write: 작업 메모리 → 주 메모리
            sharedVar = localCopy;  // write(store(localCopy))
            System.out.println("A: 씀 = " + sharedVar);
        }, "Thread-A");

        Thread threadB = new Thread(() -> {
            try { Thread.sleep(50); } catch (InterruptedException e) {}
            int localCopy = sharedVar;  // 주 메모리에서 읽기
            System.out.println("B: 읽음 = " + localCopy);  // 20일 수도, 10일 수도 있음
        }, "Thread-B");

        threadA.start();
        threadB.start();
        threadA.join();
        threadB.join();
        System.out.println("최종 sharedVar = " + sharedVar);
    }
}
```

### 실험 3: PrintAssembly로 JMM 구현 확인

```bash
cat > JmmImpl.java << 'EOF'
public class JmmImpl {
    static volatile int vol = 0;
    static int nonvol = 0;

    public static void writeVolatile() { vol = 1; }
    public static void writeNormal()   { nonvol = 1; }
    public static int  readVolatile()  { return vol; }
    public static int  readNormal()    { return nonvol; }
}
EOF

javac JmmImpl.java
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintAssembly \
     -XX:CompileCommand=compileonly,JmmImpl.writeVolatile \
     -XX:CompileCommand=compileonly,JmmImpl.writeNormal \
     JmmImpl 2>&1 | grep -E "(mov|lock|mfence|# )" | head -30

# volatile 쓰기: lock addl 가 보임
# 일반 쓰기: lock addl 없음
```

---

## 📊 성능/비용 비교

```
JMM 연산별 상대적 비용 (x86 기준):

연산 유형             | 상대 비용  | 비고
────────────────────┼──────────┼──────────────────────────────
일반 읽기 (캐시 히트)   | 1x       | L1 히트: ~4사이클
일반 쓰기 (Store Buf) | 1x       | Store Buffer 사용: 즉시 반환
volatile 읽기        | 1~2x     | x86에서 거의 무료 (LoadLoad 없어도 됨)
volatile 쓰기        | 100~300x | StoreLoad 펜스 (Store Buffer 플러시)
synchronized 진입/탈출| 50~500x  | 경합 없을 때: Thin Lock CAS
                    |          | 경합 있을 때: OS Mutex (수 μs)

재정렬 종류별 실제 발생 빈도 (x86 기준):

재정렬 유형        | x86에서 발생  | ARM에서 발생  | 해결 방법
────────────────┼─────────────┼─────────────┼──────────────
LoadLoad 재정렬   | 거의 없음     | 자주 발생     | LoadLoad 펜스
LoadStore 재정렬  | 거의 없음     | 자주 발생     | LoadStore 펜스
StoreStore 재정렬 | 없음         | 자주 발생     | StoreStore 펜스
StoreLoad 재정렬  | 자주 발생     | 자주 발생     | StoreLoad 펜스 (가장 비쌈)
```

---

## ⚖️ 트레이드오프

```
JMM 추상화의 장단점:

장점:
  플랫폼 독립적 프로그래밍 모델
  "어떤 CPU에서도 JMM 규칙대로 동작"이 보장
  개발자가 하드웨어 세부 사항을 몰라도 멀티스레드 코드 작성 가능

단점:
  추상화가 하드웨어 실제 비용을 숨김
  "volatile이 느리다"는 것을 직관적으로 알기 어려움
  JMM 규칙을 잘못 이해하면 미묘한 버그 발생 (happens-before 위반)

JMM vs Go Memory Model vs C++ Memory Order:
  Java JMM: happens-before 관계 기반 (순서의 추상화)
  Go: 같은 happens-before 기반이지만 고루틴 최적화
  C++ std::atomic: 6단계 메모리 순서 (relaxed~seq_cst) 명시적 선택
  → Java는 C++보다 단순하지만 VarHandle(Java 9+)로 C++ 수준 제어 가능
```

---

## 📌 핵심 정리

```
JMM 핵심:

추상화 모델:
  주 메모리: 모든 스레드가 공유 (인스턴스/정적 변수)
  작업 메모리: 스레드 전용 (레지스터/L1 캐시에 대응)
  스레드는 작업 메모리를 통해서만 변수를 읽고 씀

8가지 원자적 연산:
  주→작업: read, load
  작업 내: use, assign
  작업→주: store, write
  동기화:  lock, unlock

JMM의 보장:
  동일 스레드 내 프로그램 순서는 유지 (단일 스레드 의미론)
  happens-before 관계가 있으면 가시성 보장
  happens-before 관계 없이는 어떤 값도 볼 수 있음

플랫폼 독립성:
  JVM이 각 플랫폼의 펜스 명령어를 삽입
  x86: lock addl (volatile 쓰기)
  ARM: stlr/ldar (Store-Release/Load-Acquire)
```

---

## 🤔 생각해볼 문제

**Q1.** JMM에서 "주 메모리"와 실제 RAM은 1:1 대응인가?

<details>
<summary>해설 보기</summary>

아니다. JMM의 주 메모리와 작업 메모리는 **추상적 개념**이다. 실제 하드웨어는 L1/L2/L3 캐시, Store Buffer, Load Buffer, 레지스터가 복잡하게 얽혀 있다.

JMM의 추상화:
- 주 메모리 ≈ RAM + L3(공유 캐시) (정확하지는 않음)
- 작업 메모리 ≈ L1/L2 캐시 + Store Buffer + 레지스터

JMM은 이런 복잡한 하드웨어를 단순한 두 계층으로 추상화하여 개발자가 "가시성 보장이 되는가"를 happens-before 규칙으로만 판단할 수 있게 한다. 하드웨어의 정확한 동작을 몰라도 JMM 규칙을 따르면 모든 JVM 구현에서 올바르게 동작하도록 JVM이 책임진다.

</details>

---

**Q2.** `final` 필드는 JMM에서 어떤 특별한 보장을 받는가?

<details>
<summary>해설 보기</summary>

`final` 필드는 생성자 완료 후 **모든 스레드가 올바르게 초기화된 값을 볼 수 있음**을 JMM이 보장한다. 이를 "final field freezing"이라 한다.

일반 필드는 생성자에서 초기화한 후 다른 스레드에서 보지 못할 수 있다(참조가 생성자 완료 전에 다른 스레드로 escape한 경우). 하지만 `final` 필드는:

1. 생성자 내부에서 final 필드에 대한 모든 쓰기가 완료되기 전에 다른 스레드가 읽을 수 없음
2. final 필드를 포함하는 객체의 참조가 escape하더라도 final 필드는 올바른 값을 보장

이 때문에 불변 객체(Immutable Object)의 `final` 필드는 `volatile` 없이도 안전하게 공유할 수 있다. `String`, `Integer` 등 JDK 불변 클래스들이 `final` 필드를 사용하는 이유다.

</details>

---

**Q3.** 두 스레드가 `synchronized` 없이 서로 다른 변수에만 접근한다면 JMM 관점에서 안전한가?

<details>
<summary>해설 보기</summary>

서로 완전히 독립된 변수에 접근한다면 race condition은 없다. 하지만 **가시성 문제**는 여전히 존재할 수 있다.

예시: 스레드 A가 변수 a를 쓰고, 스레드 B가 변수 b를 쓴 후 a를 읽는다면, A의 쓰기가 B에게 보이지 않을 수 있다. `volatile`이나 `synchronized`를 통한 happens-before 관계가 없으면 어떤 순서도 보장되지 않는다.

"서로 다른 변수"라고 해도 **같은 객체의 서로 다른 필드**라면 False Sharing으로 인한 성능 문제가 발생할 수 있고, 한 필드를 읽고 다른 필드를 쓰는 패턴이 compound action이 되어 원자성 문제가 생길 수 있다.

완전히 독립적인(공유 상태 없는) 변수끼리만 접근한다면 JMM 상 안전하다.

</details>

---

<div align="center">

**[⬅️ 이전: 하드웨어 메모리 모델](./01-hardware-memory-model.md)** | **[홈으로 🏠](../README.md)** | **[다음: happens-before 관계 ➡️](./03-happens-before.md)**

</div>
