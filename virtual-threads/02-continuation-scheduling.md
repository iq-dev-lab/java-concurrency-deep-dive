# Continuation과 스케줄링 — 힙 저장과 ForkJoinPool

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Continuation` 객체가 JVM 힙에 Virtual Thread 스택 프레임을 저장하는 방식은?
- ForkJoinPool 캐리어 스레드가 Virtual Thread를 스케줄링하는 원리는?
- Virtual Thread의 Mount(캐리어 할당) / Unmount(캐리어 해제) 과정은?
- VT 스케줄러를 커스터마이징할 수 있는가?
- Continuation이 OS 코루틴/파이버와 어떻게 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Continuation의 동작 원리를 이해하면, Virtual Thread가 어떤 상황에서 Unmount되고 어떤 상황에서 Pinning(캐리어를 놓지 못함)이 발생하는지 정확히 예측할 수 있다. Pinning은 Virtual Thread의 가장 큰 함정이며, 이를 피하려면 JVM이 언제 Unmount를 결정하는지 알아야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Virtual Thread가 경량 스레드(green thread)와 동일하다는 오해
  "과거의 Java green thread처럼 OS 스케줄러를 무시하는 거겠지"
  → 다름: Green Thread는 협력적 멀티태스킹 (명시적 yield 필요)
  → Virtual Thread: 기존 블로킹 API 호출 시 JDK가 자동으로 Unmount
  → 코드 변경 없이 자동으로 Non-Blocking처럼 동작

실수 2: Virtual Thread가 항상 Unmount된다는 가정
  "synchronized 블록에서도 I/O 블로킹 시 Unmount되겠지"
  → 아님! synchronized 내 블로킹 → Pinning (캐리어 고착)
  → Unmount는 특정 조건에서만 발생 (LockSupport.park, NIO 등)
  → synchronized는 JVM 모니터와 연결 → Unmount 불가 (다음 문서)

실수 3: ForkJoinPool 캐리어 스레드를 설정할 필요 없다는 오해
  "JVM이 알아서 최적화하겠지"
  → 기본 캐리어 풀 크기 = Runtime.getRuntime().availableProcessors()
  → Pinning이 많으면 캐리어 부족 → 조정 필요
  → -Djdk.virtualThreadScheduler.parallelism=N 으로 조정 가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Continuation 동작 이해 기반의 코드 설계

// VT 스케줄러 설정 (커스텀)
// JVM 옵션: -Djdk.virtualThreadScheduler.parallelism=8
//           -Djdk.virtualThreadScheduler.maxPoolSize=256

// Unmount가 발생하는 올바른 패턴:
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // 1. Thread.sleep() → LockSupport.park → Unmount ✅
        Thread.sleep(100);

        // 2. ReentrantLock 대기 → AQS park → Unmount ✅
        reentrantLock.lock();
        try { doWork(); } finally { reentrantLock.unlock(); }

        // 3. BlockingQueue.take() → AQS park → Unmount ✅
        Object item = queue.take();

        // 4. Socket I/O (JDK 21 재작성) → Unmount ✅
        Socket socket = new Socket("host", 80);
        socket.getInputStream().read(buffer);
    });
}

// Pinning이 발생하는 패턴 (피해야 함):
executor.submit(() -> {
    synchronized (obj) {          // Pinning 시작
        Thread.sleep(100);        // 캐리어도 블로킹!
        socket.read(buffer);      // 캐리어도 블로킹!
    }                             // Pinning 해제
});
```

---

## 🔬 내부 동작 원리

### 1. Continuation의 구조

```
Continuation (java.lang.Continuation, JVM 내부):

Virtual Thread의 실행 상태를 캡처하는 객체:
  - 스택 프레임들 (힙에 저장, Chunk 형태)
  - 로컬 변수들
  - 실행 재개 지점 (Program Counter)

JVM 힙 저장 방식:
  Continuation이 suspend(park)될 때:
    현재 OS 스레드 스택의 VT 프레임들을
    힙의 StackChunk 객체로 복사

  Continuation이 resume될 때:
    힙의 StackChunk를 OS 스레드 스택으로 복원

StackChunk:
  크기: 초기 ~1KB, 필요 시 확장 (LinkedList 구조)
  내용: 자바 프레임 + 로컬 변수
  GC 친화적: 힙에 있으므로 GC가 관리

OS 스레드 스택 vs VT Continuation:
  OS 스레드 스택: ~1MB 연속 메모리 (스택 영역, 항상 고정)
  VT Continuation: 힙에 동적 할당 (GC로 수거 가능)
                   크기: 수 KB ~ 수십 KB (실제 프레임 수에 비례)

메모리 비교 (1,000,000개 VT):
  OS 스레드 1M개: 1TB RAM (불가)
  VT 1M개: ~각 1KB = ~1GB (힙, 현실적)
```

### 2. Mount / Unmount 과정

```
Mount (캐리어 할당):
  스케줄러가 VT를 실행할 캐리어 스레드를 선택
  힙의 Continuation(StackChunk)을 캐리어 스레드 스택으로 복원
  VT의 코드 실행 재개

Unmount (캐리어 해제):
  VT가 park 또는 블로킹 작업을 만남
  실행 중인 캐리어 스레드 스택의 VT 프레임을 힙으로 복사
  캐리어 스레드는 다른 VT를 실행하러 떠남

시각화:
  시간 →
  캐리어 스레드 1: [VT-A 실행] [VT-B 실행] [VT-A 재개] [VT-C 실행]
                         ↑           ↑           ↑
                    VT-A park    VT-B park    VT-A unpark
                    (Unmount)    (Unmount)    (Mount)

VT-A 관점:
  코드 실행 → sleep(100ms) → 100ms 후 재개 → 코드 계속
  내부: park → Unmount → (100ms 힙에서 대기) → unpark → Mount → 재개

Unmount가 발생하는 조건:
  LockSupport.park() 호출 (AQS park, Condition.await, sleep 등)
  NIO 블로킹 I/O (JDK 21에서 VT 인식 재작성)
  특정 JDK API (java.util.concurrent, java.net 등)

Unmount가 발생하지 않는 조건 (Pinning):
  synchronized 블록 내 park → Pinning!
  JNI 코드 실행 중 → Pinning!
  (자세한 내용 다음 문서)
```

### 3. ForkJoinPool 기반 스케줄러

```
Virtual Thread 스케줄러 구조:
  ForkJoinPool (특수 설정):
    parallelism = CPU 코어 수 (기본값)
    maximumPoolSize = 256 (기본값, Pinning 대비)
    mode = FIFO (기본, 캐리어 스레드 작업 선택 방식)

캐리어 스레드 풀 (ForkJoinPool 워커):
  ┌─────────────────────────────────────────────────────────┐
  │  ForkJoinPool (VT 스케줄러)                               │
  │                                                         │
  │  캐리어-0 [====실행중VT====][====실행중VT====]...            │
  │  캐리어-1 [====실행중VT====][====실행중VT====]...            │
  │  캐리어-2 [====실행중VT====][====실행중VT====]...            │
  │  캐리어-3 [====실행중VT====][====실행중VT====]...            │
  │                                                         │
  │  대기 큐: [VT-ready-1][VT-ready-2][VT-ready-3]...         │
  └─────────────────────────────────────────────────────────┘

Work Stealing:
  캐리어가 자신의 큐가 비면 다른 캐리어의 큐에서 VT를 훔침
  → 로드 밸런싱 자동화

스케줄링 과정:
  ① VT 실행 준비 (시작 or unpark 후):
     → ForkJoinPool의 submission queue에 추가
  ② 캐리어 스레드가 큐에서 VT 선택
  ③ Mount: Continuation 복원 → VT 실행
  ④ VT가 park 또는 완료:
     park → Unmount → 큐에서 제거 (unpark 시 재추가)
     완료 → VT 객체 GC 대상

ForkJoinPool 이유:
  Work Stealing으로 로드 밸런싱
  적은 스레드(코어 수)로 많은 작업 처리에 최적화
  JVM 내부 통합 (Thread 구현체로 직접 접근)
```

### 4. 스케줄러 커스터마이징

```java
// 기본 스케줄러 설정 (JVM 옵션)
// -Djdk.virtualThreadScheduler.parallelism=8      (캐리어 수)
// -Djdk.virtualThreadScheduler.maxPoolSize=256    (Pinning 시 최대)
// -Djdk.virtualThreadScheduler.minRunnable=1      (최소 실행 가능 캐리어)

// 커스텀 스케줄러로 VT 생성 (실험적 / 고급)
// (Java 21 공개 API 아님, 내부 API 또는 리플렉션 필요)
// 일반적으로 기본 스케줄러로 충분

// 스케줄러 정보 확인 (진단용)
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("실행 중 캐리어: " +
        // 내부 API: VT의 캐리어 스레드 확인
        // jdk.internal.vm.ThreadContainers 등 내부 클래스 사용
        Thread.currentThread()  // VT 자신
    );
});

// JFR로 스케줄링 이벤트 모니터링
// jdk.VirtualThreadStart, jdk.VirtualThreadEnd
// jdk.VirtualThreadPinned (Pinning 감지)
// jcmd <pid> JFR.start duration=10s filename=vt.jfr

// JVM 플래그로 Pinning 추적
// -Djdk.tracePinnedThreads=full  (전체 스택 출력)
// -Djdk.tracePinnedThreads=short (요약)
```

---

## 💻 실전 실험

### 실험 1: Continuation 저장 크기 측정

```java
import java.lang.management.*;

public class ContinuationSizeTest {
    public static void main(String[] args) throws Exception {
        Runtime rt = Runtime.getRuntime();
        int count = 100_000;

        // 기준 메모리
        System.gc();
        long baseMemory = rt.totalMemory() - rt.freeMemory();

        // 100,000개 VT 생성 (모두 sleep 중 = Continuation이 힙에 있음)
        Thread[] vts = new Thread[count];
        for (int i = 0; i < count; i++) {
            vts[i] = Thread.ofVirtual().start(() -> {
                try {
                    // 깊은 콜 스택 (Continuation 크기 증가)
                    level1();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        Thread.sleep(100);  // 모두 sleep 상태가 될 때까지 대기

        long vtMemory = rt.totalMemory() - rt.freeMemory();
        System.out.printf("VT %d개 메모리: %.2f MB%n",
            count, (vtMemory - baseMemory) / 1_048_576.0);
        System.out.printf("VT 1개당 평균: %.2f KB%n",
            (vtMemory - baseMemory) / 1024.0 / count);

        for (Thread vt : vts) vt.interrupt();
        for (Thread vt : vts) vt.join();
    }

    static void level1() throws InterruptedException {
        level2();
    }
    static void level2() throws InterruptedException {
        level3();
    }
    static void level3() throws InterruptedException {
        Thread.sleep(10000);  // Continuation이 힙에 저장되는 지점
    }
}
```

### 실험 2: Mount/Unmount 타이밍 추적

```java
public class MountUnmountTrace {
    public static void main(String[] args) throws Exception {
        // JFR 이벤트로 추적 (JVM 실행 시)
        // java -XX:StartFlightRecording=filename=mount.jfr MountUnmountTrace

        Thread vt = Thread.ofVirtual().name("traced-vt").start(() -> {
            System.out.println("[1] 시작: " + getCarrier());
            try {
                Thread.sleep(100);  // Unmount 발생
                System.out.println("[2] 재개: " + getCarrier());  // 다른 캐리어일 수 있음

                Thread.sleep(100);  // 다시 Unmount
                System.out.println("[3] 재재개: " + getCarrier());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        vt.join();
    }

    // 캐리어 스레드 이름 얻기 (내부 API, 진단용)
    static String getCarrier() {
        // Thread.currentThread()는 VT 자신
        // 실제 캐리어는 JFR이나 -Djdk.tracePinnedThreads로 확인
        return Thread.currentThread().getName() +
               " (VT=" + Thread.currentThread().isVirtual() + ")";
    }
}
```

### 실험 3: 캐리어 풀 크기 영향 측정

```bash
# 기본 설정으로 실행
java -Djdk.tracePinnedThreads=full \
     -Djdk.virtualThreadScheduler.parallelism=2 \  # 캐리어 2개
     ContinuationSchedulingTest

# 캐리어 수가 적을 때 I/O 바운드 처리량 측정
cat > ContinuationSchedulingTest.java << 'EOF'
import java.util.concurrent.*;
public class ContinuationSchedulingTest {
    public static void main(String[] args) throws Exception {
        int tasks = 1000;
        long start = System.nanoTime();

        try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = new java.util.ArrayList<Future<?>>();
            for (int i = 0; i < tasks; i++) {
                futures.add(exe.submit(() -> {
                    Thread.sleep(50);  // 50ms I/O 시뮬레이션
                    return null;
                }));
            }
            for (var f : futures) f.get();
        }

        System.out.printf("캐리어=%s, %d 작업: %.2f ms%n",
            System.getProperty("jdk.virtualThreadScheduler.parallelism", "default"),
            tasks, (System.nanoTime() - start) / 1_000_000.0);
        // 캐리어 수와 무관하게 ~50ms (모두 동시 실행)
    }
}
EOF
javac ContinuationSchedulingTest.java
java -Djdk.virtualThreadScheduler.parallelism=1 ContinuationSchedulingTest
java -Djdk.virtualThreadScheduler.parallelism=4 ContinuationSchedulingTest
# 결과: I/O 바운드에서 캐리어 수와 처리 시간 무관 (~50ms)
# CPU 바운드에서는 캐리어 수가 결정적
```

---

## 📊 성능/비용 비교

```
Continuation 저장/복원 비용:

Unmount (스택 → 힙 복사):
  작은 스택 (~10 프레임): ~수 μs
  큰 스택 (~100 프레임): ~수십 μs
  vs OS 컨텍스트 스위칭: ~1~10μs (레지스터만)

Mount (힙 → 스택 복원):
  Unmount와 유사한 비용

비교:
  VT Unmount/Mount: ~수 μs ~ 수십 μs (힙 메모리 복사)
  OS 컨텍스트 스위칭: ~1~10μs (레지스터, TLB 등)
  → VT 전환이 오히려 더 비쌀 수 있음 (스택이 크면)
  → 하지만 VT의 이점은 "I/O 대기 중 캐리어 해방"
     OS 스레드는 I/O 대기 중에도 메모리를 차지하고 스케줄러 부담

VT 스케줄러 (ForkJoinPool) 오버헤드:
  Task 제출: ~수 ns
  Work Stealing: ~수 μs
  Wake-up latency (park → unpark → resume): ~수십 μs ~ 수 ms
```

---

## ⚖️ 트레이드오프

```
Continuation 기반 VT의 트레이드오프:

장점:
  힙 메모리 = GC 관리 → 명시적 해제 불필요
  동적 크기 조정 (실제 사용만큼만 메모리 사용)
  기존 JVM GC와 완전 통합

단점:
  힙 압박: 수백만 VT → GC 부담 증가
  Unmount/Mount 비용: 스택 프레임 복사 (~수십 μs)
  캐시 콜드: Mount 후 새 캐리어 = L1 캐시 콜드

ForkJoinPool 스케줄러 특성:
  Work Stealing: 자동 로드 밸런싱 (장점)
  FIFO 모드: 작업 지연 최소화 (장점)
  공정성 없음: 특정 VT가 밀릴 수 있음 (단점, 드묾)
  커스터마이징 제한: API 미공개, JVM 옵션으로만 조정 (단점)

VT vs 코루틴(Kotlin) vs async/await(Python):
  VT: JVM 관리, 기존 Java API 그대로, GC 통합
  Kotlin 코루틴: 명시적 suspend 함수, 컴파일러 변환
  Python async/await: 명시적 await, 이벤트 루프
  → VT: 코드 변경 최소 (기존 블로킹 API 그대로)
  → 코루틴/async-await: 더 세밀한 제어 가능
```

---

## 📌 핵심 정리

```
Continuation / 스케줄링 핵심:

Continuation:
  VT의 실행 상태 (스택 프레임 + PC)를 힙에 저장하는 객체
  park 시 StackChunk → 힙 복사
  resume 시 힙 → 캐리어 스택 복원

Mount / Unmount:
  Mount: 캐리어 스레드 선택 + Continuation 복원 → VT 실행
  Unmount: VT 스택 → 힙 저장 + 캐리어 해제
  조건: LockSupport.park, NIO I/O, sleep 등

ForkJoinPool 스케줄러:
  캐리어 수 = CPU 코어 수 (기본)
  Work Stealing으로 자동 로드 밸런싱
  Pinning 시 maximumPoolSize(256)까지 캐리어 추가

Unmount 조건 (park):
  Thread.sleep() → ✅
  ReentrantLock.lock() 대기 → ✅
  BlockingQueue.take() 대기 → ✅
  Socket I/O (JDK 21) → ✅
  synchronized 블록 내 park → ❌ (Pinning)
  JNI 실행 중 → ❌ (Pinning)

커스터마이징:
  -Djdk.virtualThreadScheduler.parallelism=N
  -Djdk.virtualThreadScheduler.maxPoolSize=N
  JFR: jdk.VirtualThreadPinned으로 Pinning 감지
```

---

## 🤔 생각해볼 문제

**Q1.** Virtual Thread의 스택 프레임이 힙에 저장될 때, GC가 수행되면 어떻게 되는가? VT 실행이 중단되는가?

<details>
<summary>해설 보기</summary>

GC는 park된(힙에 있는) VT의 Continuation 객체를 일반 힙 객체처럼 처리한다. VT의 StackChunk가 살아있는 참조(VT 객체에서 참조됨)이므로 GC가 수거하지 않는다.

GC가 발생할 때:
1. Park된 VT: 힙에서 계속 대기 (GC가 이동시킬 수 있지만 참조 업데이트)
2. Mount된 VT: 캐리어 스레드 스택에서 실행 중 → GC가 중단시킬 수 있음 (STW)
3. GC STW(Stop-The-World) 동안: 모든 VT(Mount된 것 포함)가 일시 정지

`ZGC`, `Shenandoah` 같은 동시 GC를 사용하면 STW 시간을 최소화할 수 있다. Virtual Thread가 많은 서비스에서 GC 파라미터 튜닝이 중요한 이유다.

</details>

---

**Q2.** ForkJoinPool이 Virtual Thread 스케줄러로 선택된 이유는? `ThreadPoolExecutor`를 사용하면 안 되는가?

<details>
<summary>해설 보기</summary>

ForkJoinPool이 선택된 주된 이유는 **Work Stealing**이다. VT는 예측할 수 없는 시점에 Unmount/Mount되므로, 캐리어들 간에 부하가 불균등해질 수 있다. Work Stealing은 바쁜 캐리어의 큐에서 한가한 캐리어가 VT를 빼앗아 실행함으로써 자동으로 로드 밸런싱한다.

`ThreadPoolExecutor`는 공유 큐 방식이라 모든 스레드가 하나의 큐에 접근한다. 이는 경합을 유발하고 VT 수가 많아질수록 큐 접근 병목이 생긴다. ForkJoinPool의 각 캐리어별 개인 큐(deque) + Work Stealing이 훨씬 효율적이다.

또한 ForkJoinPool은 JVM 내부에서 특별 지원을 받아, VT의 park/unpark 사이클과 긴밀하게 통합되어 있다.

</details>

---

**Q3.** Virtual Thread의 스택 크기는 동적으로 늘어난다고 했다. 최대 크기는 얼마이며, OS 스레드 스택과 충돌할 가능성은?

<details>
<summary>해설 보기</summary>

VT의 Continuation은 힙에 있는 StackChunk의 연결 리스트다. 스택이 깊어지면 새 StackChunk가 링크되어 추가된다. 이론적 최대 크기는 JVM 힙 크기(-Xmx)에 의존하며, 사실상 -Xmx 범위 내에서 무제한이다.

OS 스레드 스택과 충돌 없음: VT의 스택은 힙(heap 영역)에 있고, OS 스레드의 스택은 스택 영역(native stack)에 있다. 두 영역은 완전히 분리되어 있다.

단, VT가 Mount될 때 현재 실행 중인 캐리어 스레드의 스택에 일부 프레임이 올라온다. 이 때 캐리어의 스택 크기(-Xss) 제한이 관련될 수 있다. 매우 깊은 재귀 호출은 여전히 `StackOverflowError`를 발생시킬 수 있다. VT라고 무한 재귀가 허용되지는 않는다.

</details>

---

<div align="center">

**[⬅️ 이전: 왜 Virtual Thread인가](./01-why-virtual-thread.md)** | **[홈으로 🏠](../README.md)** | **[다음: 블로킹 시 동작 — Unmount와 NIO ➡️](./03-blocking-unmount.md)**

</div>
