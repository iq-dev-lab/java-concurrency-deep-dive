# 명령어 재정렬 — 컴파일러·CPU·JIT의 3단계 재정렬

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 명령어 재정렬은 누가, 어느 단계에서 발생하는가?
- CPU의 Out-of-Order Execution은 어떻게 동작하는가?
- JIT가 재정렬을 적용하는 구체적인 예시는 무엇인가?
- Dekker 알고리즘이 x86에서 왜 의도대로 동작하지 않는가?
- `@Contended`가 False Sharing을 방지하는 원리는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"코드 순서대로 실행된다"는 가정은 단일 스레드에서만 성립한다. 멀티스레드 환경에서는 컴파일러, JIT, CPU의 3단계 재정렬이 발생하며, 이 재정렬이 동시성 버그의 근원이다. happens-before와 메모리 펜스가 이 재정렬을 제한하는 메커니즘이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수: 코드 순서 = 실제 실행 순서라고 가정

// Thread A
x = 1;         // 1번
ready = true;  // 2번

// Thread B
while (!ready) {}  // 3번
assert x == 1;     // 4번

오해: "2번이 실행된 후 3번이 종료되니까 4번에서 x=1"
현실: CPU/컴파일러가 1번과 2번의 순서를 바꿀 수 있음
      → Thread B가 ready=true를 봐도 x=0인 상황 가능
      → volatile 없이는 이 코드가 올바르다고 보장 불가

또 다른 실수: 재정렬이 단순히 느린 하드웨어의 특성이라고 생각
  → 재정렬은 성능 최적화 메커니즘
  → x86에서도, JIT 컴파일 후에도 발생
  → "내 서버에서는 됐다"는 것이 정확성 보장이 아님
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 재정렬을 막는 올바른 설계

// 방법 1: volatile로 재정렬 제한
volatile boolean ready = false;
int x = 0;

// Thread A: volatile 쓰기 이전의 일반 쓰기는 재정렬 안 됨
x = 1;          // volatile 쓰기 이전 → 이후로 이동 불가
ready = true;   // volatile 쓰기

// Thread B: volatile 읽기 이후의 읽기는 이전으로 이동 불가
while (!ready) {}  // volatile 읽기
System.out.println(x);  // volatile 읽기 이후 → x=1 보장

// 방법 2: synchronized로 완전한 순서 보장
Object lock = new Object();
synchronized (lock) {
    x = 1;
    ready = true;
}
synchronized (lock) {
    if (ready) System.out.println(x);
}

// 방법 3: VarHandle로 세밀한 메모리 오더 제어 (Java 9+)
import java.lang.invoke.VarHandle;
// setRelease: 이전 쓰기가 이후로 이동 불가 (StoreStore + StoreLoad 부분)
// getAcquire:  이후 읽기가 이전으로 이동 불가 (LoadLoad + LoadStore)
VH.setRelease(obj, newValue);  // StoreStore 펜스
int v = (int) VH.getAcquire(obj);  // LoadLoad 펜스
```

---

## 🔬 내부 동작 원리

### 1. 3단계 재정렬 발생 지점

```
재정렬이 발생하는 3단계:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
단계 1: 컴파일러 재정렬 (Java 컴파일러, javac)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  javac는 최적화를 거의 하지 않음
  대부분의 최적화는 JIT 단계에서 발생
  일부 데이터 흐름 최적화는 컴파일러 수준에서도 발생

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
단계 2: JIT 컴파일러 재정렬 (HotSpot JIT)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Java 바이트코드 → 기계어 변환 시 적극적인 최적화
  
  ① 레지스터 캐싱:
     변수를 레지스터에 올리고 루프 동안 메모리 참조 제거
     while (running) {} → running이 레지스터에 올라가면 루프 탈출 불가
     
  ② 공통 부분식 제거 (CSE):
     x + y를 여러 번 계산하면 → 한 번 계산하고 재사용
     읽기 순서 변화 유발 가능
     
  ③ 루프 불변 코드 이동 (Loop Invariant Code Motion):
     루프 안의 변하지 않는 계산 → 루프 밖으로 이동
     읽기가 루프 이전으로 이동
     
  ④ 메서드 인라이닝 + 재정렬:
     인라이닝 후 더 넓은 범위에서 최적화 적용

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
단계 3: CPU 재정렬 (Out-of-Order Execution)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  현대 CPU는 명령어를 순서와 다르게 실행해 파이프라인 효율 극대화
  
  명령어 파이프라인 단계:
    Fetch → Decode → Issue → Execute → Commit
    
  Out-of-Order:
    의존성이 없는 명령어들은 병렬로 또는 순서를 바꿔 실행
    결과는 Reorder Buffer를 통해 프로그램 순서대로 Commit
    → 단일 스레드에서는 결과가 동일
    → 다른 코어에서 보면 실행 순서가 달리 보일 수 있음
```

### 2. CPU Out-of-Order Execution 상세

```
CPU 파이프라인 구조 (단순화):

  명령어 버퍼 → 디코더 → 예약 스테이션 → 실행 유닛들 → Reorder Buffer

  예약 스테이션 (Reservation Station):
    실행 준비된 명령어를 대기
    피연산자가 준비되는 순서대로 실행 유닛에 디스패치
    → 의존성 없는 명령어끼리 순서 뒤바뀜

  Reorder Buffer (ROB):
    모든 명령어는 ROB에 순서대로 등록
    실행 완료된 명령어는 ROB에 결과 저장
    ROB의 앞(oldest)에서부터 순서대로 Commit
    → 외부(다른 코어)에서 보이는 순서는 Commit 순서

  예시:
    명령어 1: LOAD x   (메모리에서 로드, 느림)
    명령어 2: ADD  a,b (레지스터 연산, 빠름)
    명령어 3: STORE y  (메모리에 저장)
    
    OoO 실행: 명령어 2가 먼저 실행 완료 (1번 로드 대기 중)
    Commit: 1번 완료 후 → 1, 2, 3 순서로 Commit (외부에서 이 순서로 보임)

Store Buffer와 Load Buffer:
  Store Buffer: 쓰기를 버퍼링하여 나중에 L1 캐시에 반영
  Load Buffer: 읽기를 버퍼링하여 실제 메모리 접근 지연
  
  이 버퍼들이 다른 코어에서 관찰 시 "재정렬처럼" 보이는 원인:
    Core 0: Store(x=1) → Load(y)  [Store가 버퍼에 있고 Load가 먼저 일어남처럼 보임]
    Core 1: Store(y=1) → Load(x)
    결과: Core 0의 Load(y)=0, Core 1의 Load(x)=0 동시에 가능 (StoreLoad 재정렬)
```

### 3. Dekker 알고리즘과 StoreLoad 재정렬

```
Dekker 알고리즘: 뮤텍스 없이 임계 구역 진입을 보장하는 고전 알고리즘

코드:
  volatile int flag0 = 0, flag1 = 0;

  // Thread 0 (임계 구역 진입 시도)
  flag0 = 1;              // "나 들어가려고 해" 표시
  if (flag1 == 0) {       // Thread 1이 들어가려고 하지 않으면
      // 임계 구역 진입
  }

  // Thread 1 (임계 구역 진입 시도)
  flag1 = 1;              // "나 들어가려고 해" 표시
  if (flag0 == 0) {       // Thread 0이 들어가려고 하지 않으면
      // 임계 구역 진입
  }

이론적으로는 동시 진입 불가 (flag0=1 또는 flag1=1이 먼저 보여야)
하지만 x86의 StoreLoad 재정렬로:
  Thread 0: flag0=1 쓰기가 Store Buffer에 → flag1 읽기가 먼저 일어남
  Thread 1: flag1=1 쓰기가 Store Buffer에 → flag0 읽기가 먼저 일어남
  
  결과: 두 스레드 모두 flag=0을 보고 동시에 임계 구역 진입!

해결: StoreLoad 펜스(MFENCE)로 Store Buffer 플러시
  flag0 = 1;
  MFENCE;              // Store Buffer 강제 플러시
  if (flag1 == 0) { ... }

Java에서:
  volatile int flag0, flag1;
  // volatile 쓰기 → StoreLoad 펜스 → volatile 읽기 순서 보장
  flag0 = 1;      // volatile 쓰기 (StoreLoad 펜스 포함)
  if (flag1 == 0) { ... }  // volatile 읽기 (쓰기 이후)
  // 이제 Dekker 알고리즘이 올바르게 동작
```

### 4. JIT 재정렬 예시: 루프 최적화

```java
// JIT가 재정렬을 적용하는 실제 예시

// 원래 코드:
boolean running = true;
// Thread B
while (running) {
    doSomething();
}

// JIT 최적화 후 (running이 변하지 않는다고 판단 시):
if (running) {
    while (true) {   // running을 레지스터에 올리고 루프
        doSomething();
    }
}
// → Thread A가 running = false를 써도 Thread B는 레지스터 값을 봄 → 무한루프

// 해결: volatile boolean running = true;
// volatile 읽기는 매 루프마다 주 메모리에서 읽음 (레지스터 캐싱 금지)

// 또 다른 JIT 최적화: Loop Hoisting
// 원래:
for (int i = 0; i < array.length; i++) {
    sum += array[i] * multiplier;  // multiplier가 루프 내에서 불변
}
// JIT 최적화:
int m = multiplier;  // 루프 밖으로 이동 (한 번만 읽음)
for (int i = 0; i < array.length; i++) {
    sum += array[i] * m;
}
// multiplier가 volatile이 아니면:
// 루프 실행 중 다른 스레드가 multiplier를 바꿔도 반영 안 됨
```

### 5. @Contended와 False Sharing 방지

```
False Sharing 발생 메커니즘 (MESI 관점):

  CPU 캐시 라인 = 64바이트
  
  class SharedData {
      volatile int a;  // offset 0
      volatile int b;  // offset 4
  }
  // a와 b는 같은 64바이트 캐시 라인에 위치

  Core 0: a = 1;  // Modified → 전체 64바이트 캐시 라인 독점
  Core 1: b = 2;  // Core 0의 캐시 라인이 Invalidate됨
                   // Core 1이 전체 라인을 읽고 → Modified
  Core 0: a = 3;  // Core 1의 캐시 라인이 Invalidate됨
  
  → a와 b는 논리적으로 독립이지만 같은 캐시 라인 때문에
    계속 Invalidate 발생 → 성능 저하

@Contended 해결 (JDK 내부 사용):
  @sun.misc.Contended
  class PaddedData {
      @Contended volatile int a;
      @Contended volatile int b;
  }
  // @Contended가 각 필드 앞뒤에 128바이트 패딩 추가
  // → a와 b가 다른 캐시 라인에 위치 → False Sharing 방지
  // JVM 옵션: -XX:-RestrictContended (JDK 내부 클래스가 아닌 경우 필요)

수동 패딩 (Java 8 이전 방법):
  class ManuallyPadded {
      volatile long a;
      long p1, p2, p3, p4, p5, p6, p7;  // 7 × 8 = 56바이트 패딩
      volatile long b;
      // a와 b가 각각 별도의 64바이트 캐시 라인에 위치
  }

LongAdder 내부:
  Striped64의 Cell 배열:
  @Contended static final class Cell {
      volatile long value;
  }
  // 각 Cell이 독립된 캐시 라인에 위치
  // → 여러 스레드가 다른 Cell에 동시 쓰기 가능 (False Sharing 없음)
```

---

## 💻 실전 실험

### 실험 1: JIT 루프 최적화로 인한 무한루프 재현

```java
public class LoopHoistingDemo {
    static boolean running = true;  // volatile 없음

    public static void main(String[] args) throws InterruptedException {
        Thread looper = new Thread(() -> {
            long count = 0;
            while (running) {  // JIT가 running을 레지스터에 올릴 수 있음
                count++;
            }
            System.out.println("종료! count = " + count);
        });

        looper.start();
        Thread.sleep(100);

        System.out.println("running = false 설정");
        running = false;

        looper.join(2000);
        if (looper.isAlive()) {
            System.out.println("루프 탈출 실패 (JIT 최적화로 인한 가시성 문제)");
            looper.interrupt();
        }
    }
}
// 실행: java -server LoopHoistingDemo
// -server 모드에서 JIT C2 컴파일러가 공격적으로 최적화
// volatile boolean running = true; 로 바꾸면 정상 종료
```

### 실험 2: False Sharing vs 패딩 JMH 벤치마크

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Group)
@Fork(2)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class FalseSharingBenchmark {

    // False Sharing 있음
    static class SharedVars {
        volatile long a = 0;
        volatile long b = 0;
    }

    // False Sharing 없음 (패딩)
    static class PaddedVars {
        volatile long a = 0;
        long p1, p2, p3, p4, p5, p6, p7;  // 패딩
        volatile long b = 0;
    }

    SharedVars shared = new SharedVars();
    PaddedVars padded = new PaddedVars();

    @Benchmark @Group("shared") @GroupThreads(1)
    public void writeSharedA() { shared.a++; }

    @Benchmark @Group("shared") @GroupThreads(1)
    public void writeSharedB() { shared.b++; }

    @Benchmark @Group("padded") @GroupThreads(1)
    public void writePaddedA() { padded.a++; }

    @Benchmark @Group("padded") @GroupThreads(1)
    public void writePaddedB() { padded.b++; }

    // 결과: padded가 shared보다 5~10배 처리량 높음
}
```

### 실험 3: PrintAssembly로 JIT 재정렬 확인

```bash
# JIT가 루프를 어떻게 최적화하는지 어셈블리로 확인
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintAssembly \
     -XX:+LogCompilation \
     -XX:CompileOnly=LoopHoistingDemo.lambda \
     LoopHoistingDemo 2>&1 | grep -A 20 "# {method}"
# volatile 없는 경우: 루프 내 메모리 접근 없음 (레지스터 사용)
# volatile 있는 경우: 매 루프마다 메모리 접근 명령어
```

---

## 📊 성능/비용 비교

```
재정렬 유형별 x86 / ARM 발생 빈도:

재정렬 유형        | x86 발생  | ARM 발생  | 메모리 펜스
─────────────────┼──────────┼──────────┼─────────────────────────
LoadLoad          | ✗ 거의없음 | ✅ 자주   | LoadLoad 펜스
LoadStore         | ✗ 거의없음 | ✅ 자주   | LoadStore 펜스
StoreStore        | ✗ 없음    | ✅ 자주   | StoreStore 펜스
StoreLoad         | ✅ 자주   | ✅ 자주   | StoreLoad 펜스 (MFENCE)
컴파일러 레지스터 캐싱| ✅ 자주   | ✅ 자주   | volatile 키워드

False Sharing 성능 영향 (JMH 결과 참고치):

상황                         | 처리량 (ops/s) | 비고
──────────────────────────── ┼──────────────┼──────────────────
단일 스레드 쓰기              | 500,000,000  | 기준
두 스레드, 다른 캐시 라인     | 450,000,000  | 거의 저하 없음
두 스레드, 같은 캐시 라인     | 50,000,000   | 10배 저하!
두 스레드, @Contended 적용   | 440,000,000  | 거의 회복
```

---

## ⚖️ 트레이드오프

```
재정렬 제한의 비용:

volatile 쓰기 (StoreLoad 펜스):
  비용: ~100~300 사이클
  얻는 것: StoreLoad 재정렬 방지, Store Buffer 플러시

메모리 펜스 강도별 트레이드오프:
  약한 순서 (relaxed):   빠름, 재정렬 허용
  acquire/release:      중간, StoreLoad 제외 재정렬 방지
  순차 일관성 (seq_cst): 가장 안전, 가장 느림 (volatile 쓰기 수준)

VarHandle로 세밀한 제어 (Java 9+):
  VH.set(obj, val)           // plain (재정렬 허용)
  VH.setOpaque(obj, val)     // 컴파일러 재정렬만 방지
  VH.setRelease(obj, val)    // StoreStore + StoreLoad 부분 방지
  VH.setVolatile(obj, val)   // 완전한 volatile (가장 비쌈)

False Sharing 해결 트레이드오프:
  @Contended: 캐시 라인 낭비 (128바이트 패딩)
  수동 패딩: 유지보수 어려움
  구조 재설계: 근본 해결 (변수 분리 또는 Thread-Local 사용)
```

---

## 📌 핵심 정리

```
명령어 재정렬 핵심:

3단계 재정렬:
  컴파일러(javac): 미미
  JIT(HotSpot C1/C2): 레지스터 캐싱, 루프 최적화, CSE
  CPU(OoO Execution): StoreLoad 재정렬 (x86도 발생)

재정렬 방지:
  volatile: 4가지 펜스 삽입, JIT 레지스터 캐싱 금지
  synchronized: 진입/탈출 시 펜스
  VarHandle: 세밀한 메모리 오더 제어

Dekker 알고리즘:
  StoreLoad 재정렬로 인해 volatile 없이는 올바르지 않음
  x86에서 Store Buffer 지연이 핵심 원인

False Sharing:
  다른 스레드가 같은 캐시 라인의 다른 변수를 수정
  → Invalidate 반복 → 성능 10배 저하
  해결: @Contended, 수동 패딩, 구조 재설계

JIT 최적화 확인:
  -XX:+PrintAssembly: 생성된 기계어 확인
  -XX:+LogCompilation: JIT 컴파일 로그
```

---

## 🤔 생각해볼 문제

**Q1.** JIT 컴파일러가 코드를 재정렬할 때, 단일 스레드 의미론은 항상 보장된다고 했다. 어떻게 이를 보장하는가?

<details>
<summary>해설 보기</summary>

JIT 컴파일러는 **데이터 의존성 분석**을 통해 단일 스레드 의미론을 보장한다.

두 명령어 사이에 데이터 의존성이 있으면(Read-After-Write, Write-After-Read, Write-After-Write) 재정렬하지 않는다. 의존성이 없는 명령어만 재정렬한다.

```java
int a = x + 1;   // (1) x를 읽어 a에 저장
int b = a + 2;   // (2) a를 읽어 b에 저장 - (1)에 의존
int c = y + 3;   // (3) y를 읽어 c에 저장 - (1),(2)와 독립

// JIT는 (3)을 (1) 이전으로 이동 가능
// 하지만 (2)를 (1) 이전으로 이동 불가 (데이터 의존성)
```

이 분석은 단일 스레드 내에서 완벽하게 동작하지만, 다른 스레드가 공유 변수를 수정하는 경우는 고려하지 않는다. 따라서 멀티스레드 환경에서는 `volatile`, `synchronized` 등으로 명시적으로 재정렬을 제한해야 한다.

</details>

---

**Q2.** CPU의 Speculative Execution(투기적 실행)은 재정렬과 어떤 관계가 있는가?

<details>
<summary>해설 보기</summary>

Speculative Execution은 분기(if/else) 예측에 기반해 결과가 결정되기 전에 미리 명령어를 실행하는 기법이다. OoO Execution의 확장으로 볼 수 있다.

재정렬과의 관계:
- Speculative Execution으로 인해 분기 이후의 명령어가 분기 이전에 실행될 수 있음
- 분기 예측이 맞으면: 정상 동작 (성능 향상)
- 분기 예측이 틀리면: Rollback (투기적 결과 폐기)

보안 영향(Spectre/Meltdown):
- 투기적으로 실행된 명령어가 캐시 상태를 변경
- 투기 실행이 롤백되어도 캐시 상태는 유지
- → 캐시 타이밍 사이드 채널로 비밀 정보 유출 가능

Java JMM 관점:
- JVM은 투기적 실행의 메모리 오더 영향을 처리하는 펜스를 삽입
- volatile/synchronized는 투기 실행에 의한 부작용도 방지
- Spectre 관련 JVM 패치로 일부 성능 저하 발생 (Intel CPU)

</details>

---

**Q3.** `@Contended` 없이 수동으로 패딩을 추가하는 방법은 Java 버전에 따라 달라진다. Java 8에서 사용하던 방법이 Java 17에서 왜 효과가 없을 수 있는가?

<details>
<summary>해설 보기</summary>

Java 8까지 흔히 사용하던 수동 패딩 방법:
```java
class Padded {
    volatile long value;
    long p1, p2, p3, p4, p5, p6, p7;  // 56바이트 패딩
}
```

Java 17에서 JIT가 사용되지 않는 필드를 제거(Dead Field Elimination)할 수 있다. `p1~p7`이 실제로 읽히거나 쓰이지 않으면 JIT가 최적화로 제거하여 패딩 효과가 사라질 수 있다.

해결책:
1. `@Contended` 사용 (JVM이 패딩 유지 보장)
2. 필드를 실제로 사용하는 것처럼 만들어 JIT 제거 방지 (트릭, 권장 않음)
3. JDK 17+에서는 `@Contended`가 가장 안전하고 명확한 방법

`@Contended`의 접근 제어:
- JDK 내부 클래스: 자동으로 효과 있음
- 외부 코드: `-XX:-RestrictContended` JVM 플래그 필요
- Java 9+ 모듈 시스템: `--add-opens` 필요할 수 있음

</details>

---

<div align="center">

**[⬅️ 이전: volatile 완전 분해](./04-volatile-deep-dive.md)** | **[홈으로 🏠](../README.md)** | **[다음: Double-Checked Locking ➡️](./06-double-checked-locking.md)**

</div>
