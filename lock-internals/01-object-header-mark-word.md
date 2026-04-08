# Object Header와 Mark Word — 락 상태 저장 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 객체의 Object Header는 어떤 구조로 이루어져 있는가?
- Mark Word의 8바이트에 어떤 정보가 담겨 있는가?
- Biased / Thin / Fat Lock 상태별 Mark Word의 비트 레이아웃은?
- `jol-core`로 Object Header를 직접 출력하는 방법은?
- 락 상태 전환 시 Mark Word는 어떻게 변하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`synchronized`의 성능이 "락이 없을 때"와 "경쟁이 심할 때" 극단적으로 다른 이유는 Object Header의 Mark Word가 3단계 락 상태를 저장하기 때문이다. JVM이 알아서 최적화한다는 것을 막연히 아는 것과, Mark Word의 비트가 어떻게 바뀌는지 이해하는 것은 `synchronized` 성능 문제를 진단할 때 완전히 다른 시야를 준다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수: synchronized = 무조건 비싸다는 오해
  "synchronized는 OS Mutex를 쓰니까 항상 느리다"
  → 경쟁이 없을 때 Biased Lock으로 CAS조차 없이 동작
  → 경쟁이 적을 때 Thin Lock으로 CAS만으로 처리
  → OS Mutex(Fat Lock)는 심한 경쟁 시에만 사용

또 다른 실수: ReentrantLock이 항상 synchronized보다 빠르다
  "ReentrantLock은 AQS로 구현돼서 더 세밀하게 제어 가능하니 더 빠르다"
  → 경쟁 없을 때: synchronized(Biased Lock) > ReentrantLock
  → JIT의 Lock Elision이 synchronized에 더 공격적으로 적용됨
  → 실제 성능은 워크로드에 따라 다름
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```
Mark Word 이해 기반의 성능 판단:

경쟁 없는 경우:
  Biased Lock: thread_id 비교만 → 거의 비용 없음
  → 같은 스레드가 반복 진입하는 패턴에 최적화됨

경쟁이 가끔 있는 경우:
  Thin Lock(Lightweight Lock): CAS 연산 1회
  → 실패해도 스핀으로 재시도 → OS Mutex 없음

경쟁이 심한 경우:
  Fat Lock(Heavyweight Lock): OS Mutex
  → 대기 스레드 Sleep → 컨텍스트 스위칭 발생
  → 이 상황이 실제 "synchronized가 느리다"의 원인

진단 방법:
  -XX:+PrintBiasedLockingStatistics   # 바이어스 락 통계
  -XX:+UnlockDiagnosticVMOptions      # 진단 옵션 활성화
  jstack으로 BLOCKED 스레드 수 확인   # Fat Lock 경쟁 지표
```

---

## 🔬 내부 동작 원리

### 1. Java 객체 메모리 레이아웃

```
Java 힙에서 모든 객체의 메모리 구조:

┌────────────────────────────────────────────────┐
│                  Object Header                 │
│  ┌─────────────────────────────────────────┐   │
│  │  Mark Word (8바이트, 64비트 JVM)           │   │
│  │  - 해시코드, 나이, 락 상태, GC 정보 저장       │   │
│  └─────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────┐   │
│  │  Klass Pointer (4~8바이트)                │   │
│  │  - 클래스 메타데이터 포인터                   │   │
│  │  - 압축 포인터(-XX:+UseCompressedOops)     │   │
│  │    시 4바이트, 아니면 8바이트                 │   │
│  └─────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────┐   │
│  │  Array Length (배열인 경우만, 4바이트)       │   │
│  └─────────────────────────────────────────┘   │
├────────────────────────────────────────────────┤
│                  Instance Data                 │
│  (인스턴스 변수들, 타입별 크기)                       │
│  boolean: 1바이트, int: 4바이트, ref: 4~8바이트     │
├────────────────────────────────────────────────┤
│                    Padding                     │
│  (8바이트 정렬을 위한 패딩)                          │
└────────────────────────────────────────────────┘

전형적인 크기 (압축 포인터 활성화 기준):
  빈 Object: 16바이트 (Mark Word 8 + Klass Pointer 4 + 패딩 4)
  Integer 박싱: 16바이트 (헤더 12 + int 4)
  String: 24바이트 (헤더 12 + char[] ref 4 + hash 4 + coder 1 + 패딩 3)
```

### 2. Mark Word 비트 레이아웃 상세

```
64비트 JVM의 Mark Word (8바이트 = 64비트):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상태 1: Normal (락 없음, 해시코드 있음)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [63..........8][7.......3][2][1][0]
  [  해시코드(56) ][GC나이(4)][0][0][1]
  
  - 해시코드: identity hashCode() 값 (31비트 실제 사용)
  - GC 나이: GC 생존 횟수 (최대 15, Tenured 이후 생략)
  - 하위 2비트: 01 → 바이어스 가능하지만 바이어스 안 됨

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상태 2: Biased Lock (바이어스 락)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [63...................15][14...3][2][1][0]
  [ 스레드 ID(54비트)      ][ 에포크 ][1][0][1]
  
  - 스레드 ID: 락을 독점한 스레드의 ID
  - 에포크: 바이어스 취소 횟수 추적 (같은 클래스 객체 공통)
  - 하위 3비트: 101 → Biased Lock 상태

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상태 3: Thin Lock (경량 락, Lightweight Lock)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [63...........................2][1][0]
  [ Lock Record 포인터(62비트)   ][0][0]
  
  - Lock Record: 스레드 스택의 Lock Record 구조체 주소
  - Lock Record는 원본 Mark Word를 저장
  - 하위 2비트: 00 → Thin Lock 상태

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상태 4: Fat Lock (중량 락, Monitor)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [63...........................2][1][0]
  [ Monitor 포인터(62비트)       ][1][0]
  
  - Monitor: ObjectMonitor 구조체 주소
  - ObjectMonitor: 대기 큐, 소유자 등 포함
  - 하위 2비트: 10 → Fat Lock 상태

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
상태 5: GC 마킹
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [11 → GC 마킹 중]
```

### 3. Biased Lock 동작 과정

```
Biased Lock (바이어스 락):
  목적: 같은 스레드가 반복해서 같은 객체의 락을 획득하는 패턴 최적화
  구현: 락 획득 시 Mark Word에 thread_id만 기록, 이후 진입은 thread_id 비교만

초기 상태 (Biasable):
  Mark Word: [0 | 0 | 0 | 0 | 1 | 01]
  (thread_id=0, epoch=0, biasable=1)

Thread A가 최초 락 획득:
  Mark Word: [Thread_A_ID | epoch | 1 | 01]
  조건: 현재 thread_id = Thread_A_ID → 동일! → 락 획득 성공
  비용: thread_id 비교 1회 (CAS 없음!)

Thread A가 다시 진입:
  Mark Word: [Thread_A_ID | epoch | 1 | 01]
  조건: 현재 thread_id = Thread_A_ID → 동일! → 즉시 진입
  비용: 비교 1회 (거의 0에 가까움)

Thread B가 진입 시도:
  Mark Word의 thread_id != Thread_B_ID → 바이어스 취소(Revocation) 필요
  → Thread A가 락을 보유 중이면: Thin Lock으로 전환
  → Thread A가 락을 해제했으면: 새로운 바이어스 설정 또는 Thin Lock

바이어스 취소:
  STW(Stop-The-World) 발생! (Java 15 이전 방식)
  → Java 15+: 스레드를 멈추지 않는 방식으로 개선
  → Java 21: Biased Lock 기능 완전 제거 (JEP 374)
  
  이유: 현대 JDK에서는 ConcurrentHashMap 등으로 인해
        다중 스레드 패턴이 일반화 → Biased Lock의 이점 < 취소 비용
```

### 4. Lock Record와 Thin Lock

```
Thin Lock (Lightweight Lock):
  목적: 경쟁이 적은 경우 OS Mutex 없이 CAS만으로 처리
  구현: 스레드 스택에 Lock Record 생성, CAS로 Mark Word 교체

Thread A가 Thin Lock 획득:
  ① 스택에 Lock Record 생성
     Lock Record = { displaced_header: [원본 Mark Word 복사] }
  
  ② CAS로 Mark Word 교체:
     기대값: [Normal Mark Word] (01 상태)
     새 값:  [Lock Record 주소] (00 상태)
  
  ③ CAS 성공: 락 획득
     Mark Word: [Lock Record 주소 | 00]
  
  ④ Thread A가 재진입:
     Lock Record 재생성, Mark Word 변경 없이
     Lock Record에 재진입 카운터 증가

Thread A가 Thin Lock 해제:
  ① CAS로 Mark Word 복원:
     기대값: [Lock Record 주소 | 00]
     새 값:  [displaced_header(원본 Mark Word)]
  ② CAS 성공: 해제 완료

Thread B가 Thin Lock 획득 시도 (Thread A 보유 중):
  CAS 실패 (Mark Word가 이미 Thread A의 Lock Record)
  → 스핀(Spin) 재시도 (적응적 스핀)
  → 스핀 한계 초과 → Fat Lock으로 확장

적응적 스핀 (Adaptive Spinning):
  -XX:+UseAdaptiveSpin (기본값 on)
  이전에 짧은 대기로 락을 획득한 경우 → 더 오래 스핀
  이전에 스핀 후에도 못 얻은 경우 → 빨리 Fat Lock으로 전환
```

---

## 💻 실전 실험

### 실험 1: jol-core로 Object Header 출력

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

```java
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.vm.VM;

public class MarkWordInspector {
    public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();

        // 현재 Mark Word 출력
        System.out.println("=== 초기 상태 (락 없음) ===");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());

        // 해시코드 계산 후
        obj.hashCode();
        System.out.println("=== hashCode() 후 ===");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        // Mark Word에 해시코드가 저장된 것 확인

        // synchronized 후 (Biased Lock 또는 Thin Lock)
        synchronized (obj) {
            System.out.println("=== synchronized 블록 내부 ===");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
            // Mark Word 하위 비트가 변경됨
        }

        System.out.println("=== synchronized 블록 탈출 후 ===");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());

        // VM 정보
        System.out.println(VM.current().details());
    }
}
// 실행: java -XX:+UnlockDiagnosticVMOptions -XX:+PrintBiasedLockingStatistics MarkWordInspector
```

### 실험 2: Biased Lock 통계 확인

```bash
cat > BiasedLockTest.java << 'EOF'
public class BiasedLockTest {
    private static final Object LOCK = new Object();

    public static void main(String[] args) throws InterruptedException {
        // 단일 스레드로 반복 락 획득 → Biased Lock 활성화
        long count = 0;
        long start = System.nanoTime();
        
        for (int i = 0; i < 10_000_000; i++) {
            synchronized (LOCK) {
                count++;
            }
        }
        
        long elapsed = System.nanoTime() - start;
        System.out.printf("10M 락 획득: %.2f ms (평균 %.0f ns/회)%n",
            elapsed / 1_000_000.0, (double) elapsed / 10_000_000);
        System.out.println("count = " + count);
    }
}
EOF

# Java 11 (Biased Lock 있음) vs Java 21 (Biased Lock 제거)
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintBiasedLockingStatistics \
     BiasedLockTest

# Java 17 이후: Biased Lock deprecated, 경고 출력
# Java 21: -XX:+UseBiasedLocking 옵션 자체 제거
```

### 실험 3: 락 상태 전환에 따른 성능 차이

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
public class LockStateBenchmark {

    private final Object lock = new Object();
    private int counter = 0;

    // 단일 스레드: Biased Lock (Java 11) 또는 일반 Thin Lock (Java 21)
    @Benchmark
    @Threads(1)
    public int singleThread() {
        synchronized (lock) { return ++counter; }
    }

    // 2스레드: Thin Lock (약한 경쟁)
    @Benchmark
    @Threads(2)
    public int twoThreads() {
        synchronized (lock) { return ++counter; }
    }

    // 8스레드: Fat Lock (강한 경쟁)
    @Benchmark
    @Threads(8)
    public int eightThreads() {
        synchronized (lock) { return ++counter; }
    }
}
// 결과: 1스레드 >> 2스레드 > 8스레드
// 경쟁이 심해질수록 Fat Lock으로 전환 → 성능 급락
```

---

## 📊 성능/비용 비교

```
락 상태별 비용 비교:

상태          | 조건           | 비용                | 비용 수준
─────────────┼───────────────┼────────────────────┼──────────
Biased Lock  | 같은 스레드 재진입| thread_id 비교 1회   | ~1사이클
             | (Java 15 이전) | CAS 없음, STW 없음   |
Thin Lock    | 무경쟁          | CAS 1회            | ~10~20사이클
             |               | Lock Record 생성    |
Thin Lock    | 약한 경쟁       | CAS 실패 + 스핀      | ~50~200사이클
             | (스핀 성공)     | (적응적 스핀)         |
Fat Lock     | 강한 경쟁       | OS Mutex 획득       | ~수 μs (수천 사이클)
             |               | 컨텍스트 스위칭        | 경합 시 불확정

Java 버전별 Biased Lock 변화:
  Java 6~14: Biased Lock 기본 활성화 (-XX:+UseBiasedLocking)
  Java 15:   Deprecated, 성능 이점이 없어짐
  Java 17:   기본 비활성화 (-XX:-UseBiasedLocking)
  Java 21:   옵션 자체 제거 (JEP 374)
  
  현대 JDK(21+)의 단일 스레드 synchronized:
    Thin Lock(CAS) 사용 → Biased Lock 없이도 빠름
    Lock Elision(JIT)으로 아예 제거될 수도 있음
```

---

## ⚖️ 트레이드오프

```
Biased Lock의 역사적 트레이드오프:

도입 이유 (Java 6):
  당시 Java 컬렉션(Vector, Hashtable)이 모두 synchronized
  단일 스레드 사용 패턴이 많았음
  → 같은 스레드 반복 진입 최적화로 큰 성능 이득

제거 이유 (Java 21):
  ConcurrentHashMap 등 현대 컬렉션 도입으로 단일 스레드 synchronized 사용 감소
  JVM Biased Lock 취소 코드 유지 비용 증가
  GraalVM, Project Loom과의 통합 복잡성
  → 이점 < 유지 비용

Mark Word 구조의 트레이드오프:
  장점: 모든 객체가 락 정보를 포함 → 추가 메모리 구조 불필요
  단점: Mark Word 8바이트에 해시코드, 나이, 락 상태를 공유
        → 해시코드 계산 후에는 Biased Lock 불가 (Mark Word 자리 차지)
        → GC 나이와 락 정보가 같은 공간 사용 → 저장/복원 필요
```

---

## 📌 핵심 정리

```
Object Header와 Mark Word 핵심:

Object Header 구조:
  Mark Word(8B) + Klass Pointer(4~8B) + [Array Length(4B)]
  모든 Java 객체에 존재

Mark Word 4가지 상태:
  Normal: [해시코드|나이|01]
  Biased:  [thread_id|epoch|101] → 같은 스레드 반복 진입 최적화 (Java 21에서 제거)
  Thin:    [LockRecord포인터|00] → CAS 기반, 약한 경쟁
  Fat:     [Monitor포인터|10] → OS Mutex, 강한 경쟁

락 전환 방향:
  최초 진입 → Biased(Java 15↓) 또는 Thin(Java 21)
  다른 스레드 진입 → Thin Lock (CAS)
  경쟁 심화 → Fat Lock (OS Mutex, 컨텍스트 스위칭)

Java 21 변화:
  Biased Lock 완전 제거
  Thin Lock과 Lock Elision으로 대부분 케이스 처리

진단:
  jol-core: Mark Word 직접 출력
  -XX:+PrintBiasedLockingStatistics: 통계 (Java 15 이하)
  jstack BLOCKED 스레드: Fat Lock 경쟁 진단
```

---

## 🤔 생각해볼 문제

**Q1.** `System.identityHashCode(obj)`를 호출하면 Mark Word의 해시코드 필드에 값이 저장된다. 이 상태에서 해당 객체를 Biased Lock으로 사용할 수 없는 이유는?

<details>
<summary>해설 보기</summary>

Mark Word는 8바이트로 여러 상태를 공유한다. Biased Lock 상태에서 Mark Word는 thread_id를 저장하는 데 대부분의 비트를 사용한다. 하지만 Normal 상태에서 `identityHashCode()`가 호출되면 해시코드가 Mark Word에 저장된다.

이미 해시코드가 저장된 Mark Word를 Biased Lock 상태로 전환하면 해시코드가 사라진다. 이는 허용되지 않는다(`identityHashCode()`는 객체 생명주기 동안 항상 같은 값을 반환해야 하므로).

따라서 `identityHashCode()`가 한 번이라도 호출된 객체는 Biased Lock 상태로 전환되지 않는다. JVM은 이 경우 Thin Lock으로 바로 진입한다. 이것이 `HashMap`의 키로 사용되는 객체들이 Biased Lock 혜택을 받기 어려운 이유 중 하나다.

</details>

---

**Q2.** 재진입(Reentrant) synchronized는 어떻게 처리되는가? Thin Lock 상태에서 같은 스레드가 두 번 `synchronized` 블록에 진입하면?

<details>
<summary>해설 보기</summary>

Thin Lock에서의 재진입:
- Thread A가 첫 번째 진입: Mark Word = [Lock Record 1 주소 | 00]
- Thread A가 두 번째 진입: 새로운 Lock Record 2를 스택에 생성
  - Lock Record 2의 `displaced_header`를 0(null)으로 설정 (재진입 표시)
  - Mark Word는 변경하지 않음 (여전히 Lock Record 1 주소)

해제 순서:
- 두 번째 블록 탈출: Lock Record 2가 0이므로 Mark Word 변경 없이 폐기
- 첫 번째 블록 탈출: CAS로 원본 Mark Word 복원

Fat Lock에서의 재진입:
- ObjectMonitor의 `_recursions` 카운터 증가/감소
- 락 소유 스레드가 재진입 시 카운터만 증가, OS Mutex 재획득 없음

이것이 Java의 synchronized가 자동으로 재진입을 지원하는 내부 메커니즘이다.

</details>

---

**Q3.** GC가 객체를 이동시킬 때(Compaction), 해당 객체의 Mark Word에 Lock Record 포인터가 저장된 경우 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

GC가 살아있는 객체를 이동시킬 때, 해당 객체가 Thin Lock 상태(Mark Word에 Lock Record 포인터)라면:

1. GC는 객체 이동 전에 Mark Word를 저장
2. 새 위치로 객체 복사
3. 새 위치의 Mark Word를 포워딩 포인터(old→new)로 설정
4. GC 완료 후 참조 업데이트

락을 보유 중인 스레드 관점:
- 스레드의 Lock Record에는 원본 Mark Word가 저장됨
- GC 후 Mark Word 복원 시 원본 값을 사용
- GC는 STW(Stop-The-World) 중에 발생하므로 락 보유 스레드는 일시 정지 상태

실제로 현대 GC(G1, ZGC 등)는 Lock을 고려한 정교한 Mark Word 처리 로직을 갖추고 있다. ZGC의 Colored Pointers처럼 Mark Word 비트를 GC 목적으로 재사용하기도 하며, 이 때문에 Mark Word 레이아웃이 GC 종류에 따라 다를 수 있다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: synchronized 락 확장 ➡️](./02-synchronized-lock-escalation.md)**

</div>
