# 하드웨어 메모리 모델 — CPU 캐시와 Write Buffer

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CPU L1/L2/L3 캐시는 어떤 구조이고, 왜 가시성 문제를 유발하는가?
- Write Buffer(스토어 버퍼)와 스토어 포워딩이 무엇인가?
- MESI 프로토콜은 캐시 일관성을 어떻게 보장하는가?
- 멀티코어에서 같은 변수를 두 코어가 다른 값으로 보는 상황이 실제로 발생하는가?
- 이 하드웨어 특성이 Java JMM 설계에 어떤 영향을 미쳤는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`volatile`을 왜 써야 하는지, `synchronized`가 가시성을 왜 보장하는지를 "규칙이니까"로 외우는 것과, CPU 캐시 구조에서 필연적으로 발생하는 문제의 해결책으로 이해하는 것은 완전히 다르다. 하드웨어 메모리 모델을 이해하면 JMM의 모든 규칙이 자연스럽게 연결된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: 멀티스레드 환경에서 공유 플래그 변수를 사용

class StopFlag {
    private boolean running = true;  // volatile 없음

    public void stop() { running = false; }

    public void run() {
        while (running) {
            // 작업 수행
        }
        System.out.println("종료됨");
    }
}

원리를 모를 때의 판단:
  "running = false를 분명히 호출했는데 while 루프가 안 끝난다"
  → "버그가 있나? 코드를 다시 확인한다"
  → 원인을 끝내 찾지 못하고 sleep()을 추가하거나 포기

실제 원인:
  Thread A (run() 실행): running을 CPU 레지스터 또는 L1 캐시에 캐시
    → 주 메모리의 변경을 감지하지 못함
  Thread B (stop() 실행): running = false를 자신의 캐시에 씀
    → 주 메모리 반영이 지연됨
  → Thread A는 캐시된 true를 계속 봄
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```
하드웨어 구조를 알 때의 판단:
  "running이 캐시됐을 수 있다 → volatile로 선언해야 한다"

class StopFlag {
    private volatile boolean running = true;  // 가시성 보장

    public void stop() { running = false; }

    public void run() {
        while (running) { /* 작업 */ }
        System.out.println("종료됨");
    }
}

volatile의 효과:
  쓰기: 캐시에 쓰고 즉시 주 메모리에 반영 + 다른 캐시 무효화
  읽기: 항상 주 메모리(또는 최신 캐시)에서 읽음
  → 모든 스레드가 항상 최신 값을 봄

단, volatile은 가시성만 보장
  compound action (read-modify-write)은 원자성 보장 안 됨
  → count++ 같은 연산엔 AtomicInteger 또는 synchronized 필요
  (자세한 내용은 04-volatile-deep-dive.md)
```

---

## 🔬 내부 동작 원리

### 1. CPU 캐시 계층 구조

```
현대 x86-64 멀티코어 프로세서 (8코어 예시):

┌─────────────────────────────────────────────────────────────┐
│                     CPU 칩                                   │
│                                                             │
│  Core 0          Core 1    ...    Core 6          Core 7    │
│  ┌──────┐         ┌──────┐         ┌──────┐         ┌──────┐│
│  │  L1  │         │  L1  │         │  L1  │         │  L1  ││
│  │ 32KB │         │ 32KB │         │ 32KB │         │ 32KB ││
│  │ ~1ns │         │ ~1ns │         │ ~1ns │         │ ~1ns ││
│  ├──────┤         ├──────┤         ├──────┤         ├──────┤│
│  │  L2  │         │  L2  │         │  L2  │         │  L2  ││
│  │256KB │         │256KB │         │256KB │         │256KB ││
│  │ ~4ns │         │ ~4ns │         │ ~4ns │         │ ~4ns ││
│  └──┬───┘         └──┬───┘         └──┬───┘         └──┬───┘│
│     │                │                │                │    │
│  ┌──┴────────────────┴────────────────┴────────────────┴──┐ │
│  │                    L3 캐시 (공유)                        │ │
│  │              8MB ~ 32MB, ~10ns ~ 40ns                  │ │
│  └──────────────────────────┬─────────────────────────────┘ │
└─────────────────────────────┼───────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │    주 메모리 (RAM)  │
                    │    수 GB, ~100ns   │
                    └───────────────────┘

캐시 라인 (Cache Line):
  캐시의 기본 단위 = 64바이트
  변수 하나를 읽으면 해당 변수가 속한 64바이트 블록 전체가 캐시로 로드
  → 인접한 변수도 자동으로 캐시에 올라옴 (공간 지역성)

접근 시간 비교:
  L1 캐시 히트:  ~1ns   (4사이클)
  L2 캐시 히트:  ~4ns   (12사이클)
  L3 캐시 히트:  ~10ns  (30~40사이클)
  RAM 접근:      ~100ns (300사이클)
  → 캐시 히트/미스 여부가 성능을 100~300배 좌우
```

### 2. Write Buffer와 스토어 포워딩

```
Write Buffer (Store Buffer):
  CPU와 캐시 사이의 작은 버퍼
  쓰기 연산을 즉시 캐시에 반영하지 않고 버퍼에 쌓아두었다가 나중에 일괄 처리
  목적: 쓰기 지연(Write Latency) 숨기기, 파이프라인 스톨 방지

  Thread A (Core 0)에서 x = 1 실행:
    ① x = 1을 Store Buffer에 기록 (즉시 반환)
    ② CPU는 다음 명령어 실행 계속
    ③ 나중에 Store Buffer → L1 캐시 → L2 → L3 → RAM으로 전파

  문제: Store Buffer에 아직 있는 동안
    Thread B (Core 1)가 x를 읽으면?
    → Core 1의 캐시에는 아직 x의 이전 값이 있음
    → Thread B는 x = 0(이전 값)을 볼 수 있음

스토어 포워딩 (Store-to-Load Forwarding):
  같은 코어에서 쓰기 후 읽기 시:
    Store Buffer의 값을 직접 Load에 포워딩 (캐시 거치지 않음)
  → 같은 코어 내에서는 항상 자신이 쓴 최신 값을 봄
  → 다른 코어는 해당 값을 못 볼 수 있음 (비대칭 가시성)

┌────────────────────────────────────────────────────────┐
│  Core 0                    Core 1                      │
│  [Store Buffer]            [Store Buffer]              │
│    x=1 (아직 미전파)                                      │
│  [L1 Cache]                [L1 Cache]                  │
│    x=0 (이전 값)              x=0 (이전 값)               │
│                                                        │
│  Thread A: x = 1 씀        Thread B: x 읽음 → 0 봄       │
│  (Store Buffer에 있음)      (자신의 L1에서 읽음)            │
└────────────────────────────────────────────────────────┘
```

### 3. MESI 캐시 일관성 프로토콜

```
MESI: 각 캐시 라인이 가지는 4가지 상태

M (Modified):
  이 코어만 해당 데이터를 가지고 있고, 수정됨
  주 메모리와 불일치 상태
  다른 코어가 읽으려 하면: 주 메모리에 쓰고 S 상태로 전환

E (Exclusive):
  이 코어만 가지고 있고, 수정 안 됨
  다른 코어가 읽으려 하면: S 상태로 전환

S (Shared):
  여러 코어가 같은 데이터를 공유
  수정하려면: 다른 코어의 캐시에 Invalidate 신호 전송
  → 다른 코어의 캐시 라인이 Invalid 상태가 됨

I (Invalid):
  해당 캐시 라인이 무효화됨
  다음 읽기 시 캐시 미스 → 메모리에서 재로드

동작 예시 (두 스레드가 같은 변수 x를 공유):

초기 상태: x = 0, Core0 L1 = S(x=0), Core1 L1 = S(x=0)

Core0가 x = 1 씀:
  ① Core0: S → M 전환 요청
  ② Bus에 Invalidate 신호 브로드캐스트
  ③ Core1: x 캐시 라인 → I (Invalid)
  ④ Core0: x = 1 씀 (M 상태)

Core1이 x 읽음:
  ① Core1: x가 Invalid → 캐시 미스
  ② Bus에 Read 신호
  ③ Core0: M 상태이므로 Write-Back (주 메모리에 쓰기) 후 S로 전환
  ④ Core1: 최신 값(x=1) 로드 → S 상태

MESI의 한계:
  Invalidate 신호가 전달되는 데 시간이 걸림
  그 사이에 Core1이 읽으면 이전 값을 볼 수 있음
  → Store Buffer + MESI 지연 = 가시성 문제의 근본 원인
  → 메모리 펜스(fence)로 Store Buffer 강제 플러시 필요
```

### 4. 가시성 문제 발생 조건

```
가시성 문제가 발생하는 3가지 경로:

경로 1: Store Buffer 지연
  Core0: x = 1 (Store Buffer에 기록, 캐시 미전파)
  Core1: x 읽음 → 0 (Core0의 Store Buffer에 접근 불가)
  → 해결: StoreLoad 펜스로 Store Buffer 강제 플러시

경로 2: 컴파일러 레지스터 캐싱
  JIT가 루프 내 변수를 레지스터에 캐시
  while (running) { } → running을 레지스터에 올리고 루프
  → 주 메모리 변경을 읽지 않음
  → 해결: volatile → 레지스터 캐싱 금지

경로 3: CPU 명령어 재정렬
  CPU가 성능 최적화를 위해 명령어 순서를 바꿈
  data = 42;     → 실제 실행 순서가 바뀔 수 있음
  ready = true;
  → 다른 코어에서 ready=true를 봤는데 data가 아직 0일 수 있음
  → 해결: 메모리 펜스 (happens-before 보장)

Java volatile이 이 모두를 해결하는 이유:
  ① 레지스터 캐싱 금지 (컴파일러 최적화 차단)
  ② StoreLoad 펜스 삽입 (Store Buffer 플러시)
  ③ 재정렬 제한 (메모리 펜스로 순서 보장)
```

---

## 💻 실전 실험

### 실험 1: 가시성 문제 재현 (volatile 없음)

```java
public class VisibilityBug {
    // volatile 없음 → 가시성 문제 발생 가능
    private static boolean running = true;
    private static int counter = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int localCount = 0;
            while (running) {
                localCount++;
            }
            counter = localCount;
            System.out.println("Worker 종료, count = " + counter);
        });

        worker.start();
        Thread.sleep(1000);

        System.out.println("running = false 설정");
        running = false;  // Worker 스레드가 이 변경을 못 볼 수 있음

        worker.join(3000);
        if (worker.isAlive()) {
            System.out.println("Worker가 종료되지 않음! (가시성 문제)");
            worker.interrupt();
        }
    }
}
// 실행 옵션: -server -XX:+OptimizeStringConcat
// JIT 최적화 활성화 시 가시성 문제 더 잘 재현됨
// volatile boolean running = true; 로 바꾸면 항상 정상 종료
```

### 실험 2: MESI 상태 변화를 캐시 미스율로 관찰

```bash
# perf stat으로 캐시 미스 측정
# 두 스레드가 같은 변수를 계속 읽고 쓰는 경우 vs 분리된 변수

cat > CacheBench.java << 'EOF'
public class CacheBench {
    // False Sharing: 두 변수가 같은 캐시 라인 (64바이트)
    static volatile long sharedA = 0;
    static volatile long sharedB = 0;

    // 패딩으로 분리: 각 변수가 다른 캐시 라인
    static volatile long paddedA = 0;
    static long[] pad = new long[7];  // 7 × 8 = 56바이트 패딩
    static volatile long paddedB = 0;

    public static void main(String[] args) throws InterruptedException {
        // 테스트 1: 같은 캐시 라인 (MESI Invalidate 폭발)
        long start = System.nanoTime();
        Thread t1 = new Thread(() -> {
            for (long i = 0; i < 100_000_000; i++) sharedA++;
        });
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < 100_000_000; i++) sharedB++;
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.printf("같은 캐시 라인: %.2f ms%n",
            (System.nanoTime() - start) / 1_000_000.0);

        // 테스트 2: 다른 캐시 라인
        start = System.nanoTime();
        t1 = new Thread(() -> {
            for (long i = 0; i < 100_000_000; i++) paddedA++;
        });
        t2 = new Thread(() -> {
            for (long i = 0; i < 100_000_000; i++) paddedB++;
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.printf("다른 캐시 라인: %.2f ms%n",
            (System.nanoTime() - start) / 1_000_000.0);
        // 같은 캐시 라인이 훨씬 느림 → MESI Invalidate 비용
    }
}
EOF
javac CacheBench.java && java -server CacheBench
```

### 실험 3: PrintAssembly로 메모리 펜스 확인

```bash
# hsdis (HotSpot Disassembler) 설치 후
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintAssembly \
     -XX:CompileOnly=VolatileTest.write \
     VolatileTest 2>&1 | grep -A 5 -E "mov|lock|mfence"

# volatile 쓰기에서 나타나는 어셈블리:
# mov    DWORD PTR [rax], 0x1     ← 실제 쓰기
# lock addl $0x0,(%rsp)           ← x86에서 StoreLoad 펜스 구현
#                                   (MFENCE보다 빠른 대안)

# non-volatile 쓰기:
# mov    DWORD PTR [rax], 0x1     ← 쓰기만 있고 펜스 없음
```

---

## 📊 성능/비용 비교

```
캐시 계층별 접근 비용:

계층            | 용량        | 지연시간   | 사이클   | 비고
───────────────┼────────────┼──────────┼────────┼──────────────────
L1 Cache       | 32KB       | ~1ns     | 4      | 코어 전용
L2 Cache       | 256KB      | ~4ns     | 12     | 코어 전용
L3 Cache       | 8~32MB     | ~10~40ns | 30~100 | 코어 공유
RAM            | GB~TB      | ~100ns   | 300    |
Remote NUMA    | (다른 소켓)  | ~200ns   | 600+   | NUMA 아키텍처

MESI 프로토콜 비용:

상황                              | 비용
─────────────────────────────────┼──────────────────────────
L1 히트 (Shared/Exclusive)        | ~4사이클
L1 히트 (Modified, 같은 코어)       | ~4사이클
다른 코어에서 수정 → Invalidate      | ~100~200사이클 (L3 경유)
주 메모리에서 재로드                  | ~300사이클

volatile 쓰기 vs 일반 쓰기:
  일반 쓰기:    ~1사이클 (Store Buffer 활용)
  volatile 쓰기: ~100~300사이클 (StoreLoad 펜스로 Store Buffer 플러시)
  → volatile은 정확성을 위해 성능을 희생
  → 핫 경로(hot path)의 volatile은 신중하게 사용
```

---

## ⚖️ 트레이드오프

```
캐시 아키텍처의 트레이드오프:

캐시가 있어서 얻는 것:
  단일 스레드 성능 극대화 (RAM의 100~300배 빠른 접근)
  파이프라인 효율화 (Write Buffer로 쓰기 지연 숨김)

캐시 때문에 잃는 것:
  멀티코어 가시성 보장 필요 → 메모리 펜스 비용
  캐시 일관성 프로토콜(MESI) 오버헤드
  False Sharing으로 인한 의도치 않은 성능 저하

x86 vs ARM의 차이:
  x86 (Intel/AMD): TSO(Total Store Order) 메모리 모델
    → Store Buffer가 있지만 상대적으로 강한 순서 보장
    → volatile 쓰기: LOCK ADDL (MFENCE보다 빠름)
  ARM: 더 약한 메모리 모델
    → 더 많은 재정렬 허용
    → 명시적 DMB/DSB 펜스 필요
  → JVM이 플랫폼별로 다른 펜스 명령어를 삽입하는 이유
```

---

## 📌 핵심 정리

```
하드웨어 메모리 모델 핵심:

캐시 계층:
  L1(~1ns) → L2(~4ns) → L3(~40ns) → RAM(~100ns)
  단위: 64바이트 캐시 라인

가시성 문제의 3가지 원인:
  ① Store Buffer 지연: 쓰기가 다른 코어에 즉시 전파 안 됨
  ② 컴파일러 레지스터 캐싱: JIT가 변수를 레지스터에 묶음
  ③ CPU 명령어 재정렬: 성능 최적화로 순서가 바뀜

MESI 프로토콜:
  Modified / Exclusive / Shared / Invalid
  쓰기 시 다른 코어의 캐시 라인 Invalidate
  Invalidate 전파 지연 → 가시성 간격 존재

해결책:
  volatile → 레지스터 캐싱 금지 + StoreLoad 펜스
  synchronized → 블록 진입/탈출 시 메모리 펜스
  AtomicXxx → CAS + 펜스

Java에서:
  volatile 쓰기 → lock addl (x86 StoreLoad 펜스)
  volatile 읽기 → LoadLoad/LoadStore 펜스 (x86에서 거의 무료)
```

---

## 🤔 생각해볼 문제

**Q1.** 단일 스레드 프로그램에서는 `volatile`이 필요 없는가? 그 이유는?

<details>
<summary>해설 보기</summary>

단일 스레드에서는 `volatile`이 필요 없다. 가시성 문제는 여러 스레드(= 여러 코어)가 동시에 같은 변수에 접근할 때 발생한다.

단일 스레드에서는:
- Store Buffer의 값은 스토어 포워딩으로 즉시 사용 가능 (자신의 Store Buffer)
- 컴파일러 재정렬이 단일 스레드 의미론을 깨지 않도록 JIT가 보장
- MESI 프로토콜 문제 없음 (하나의 코어만 사용)

하지만 `volatile`을 단일 스레드에서 써도 기능상 문제는 없다. 다만 불필요한 메모리 펜스로 성능이 떨어질 수 있다. Signal Handler와의 통신처럼 같은 스레드 내부의 특수한 케이스에서는 `volatile`이 의미가 있을 수 있지만, 일반 Java 코드에서는 단일 스레드에서 `volatile`은 필요 없다.

</details>

---

**Q2.** 같은 캐시 라인에 있는 두 변수를 서로 다른 스레드가 수정하면 왜 느려지는가? (False Sharing)

<details>
<summary>해설 보기</summary>

MESI 프로토콜은 **캐시 라인 단위**로 동작한다. Core0가 변수 A를 수정하면 해당 캐시 라인(64바이트) 전체가 Modified 상태가 되고, Core1의 같은 캐시 라인(B가 포함된)이 Invalidate된다.

Core1이 B를 수정하면 반대로 Core0의 캐시 라인이 Invalidate된다. A와 B는 논리적으로 독립적이지만 물리적으로 같은 캐시 라인에 있어서, 서로의 쓰기가 상대 코어의 캐시를 계속 무효화시킨다.

이를 False Sharing(위공유)이라 한다. 해결책:
1. `@Contended` 어노테이션 (`-XX:-RestrictContended` JVM 플래그 필요)
2. 변수 사이에 수동 패딩(long 배열 7개, 총 56바이트)
3. `LongAdder`처럼 변수 자체를 분산 설계

JDK 내부에서 `Striped64`(LongAdder 기반)의 `Cell` 클래스가 `@Contended`를 사용하는 이유다.

</details>

---

**Q3.** x86 아키텍처는 ARM보다 강한 메모리 순서를 보장한다. 그렇다면 x86에서만 테스트한 코드가 ARM(M1/M2 Mac, AWS Graviton)에서 다르게 동작할 수 있는가?

<details>
<summary>해설 보기</summary>

있다. x86의 TSO(Total Store Order) 모델은 Store→Load 재정렬만 허용하고 나머지는 순서를 보장한다. ARM은 이보다 약한 모델을 사용하여 더 많은 재정렬을 허용한다.

JVM은 플랫폼별로 다른 펜스 명령어를 삽입하여 JMM 스펙을 보장하므로, **`volatile`과 `synchronized`를 올바르게 사용한 코드는 플랫폼 무관하게 동작해야 한다**.

문제가 생기는 경우:
- JNI나 `sun.misc.Unsafe`로 직접 메모리를 조작할 때
- `volatile` 없이 멀티스레드 코드를 작성할 때 "x86에서 우연히 됐던" 코드
- Java 9 이전의 `VarHandle` 없이 작성된 Lock-Free 코드

현대 클라우드 환경에서 AWS Graviton(ARM64)이 많이 사용되므로 이 차이가 실제 운영 장애로 이어질 수 있다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: JMM — 주 메모리와 작업 메모리 ➡️](./02-jmm-main-working-memory.md)**

</div>
