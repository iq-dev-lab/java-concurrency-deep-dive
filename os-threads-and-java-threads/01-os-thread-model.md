# OS 스레드 모델 — 커널 스레드와 1:1 매핑 비용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 커널 스레드와 사용자 스레드는 무엇이 다르고, Java는 어느 쪽을 사용하는가?
- `new Thread().start()`를 호출할 때 JVM 내부에서 무슨 일이 일어나는가?
- Java 스레드 하나가 소비하는 실제 OS 자원은 얼마인가?
- 왜 스레드 수를 무한정 늘릴 수 없는가?
- Virtual Thread는 이 구조의 어떤 문제를 해결하기 위해 탄생했는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"스레드를 늘리면 처리량이 늘어난다"는 직관은 어느 순간 무너진다. 스레드를 1,000개 만들었는데 오히려 느려지거나, OOM이 발생하거나, CPU 사용률이 치솟는 상황을 겪었다면 스레드가 OS 수준에서 무엇을 소비하는지 모른 것이다.

Java 스레드가 OS 스레드에 1:1로 매핑된다는 사실은 단순한 구현 선택이 아니라, 스레드 수 제한, 메모리 설계, Thread Pool 튜닝, 그리고 Virtual Thread 전환 여부까지 결정하는 근본 원리다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: API 서버의 응답 속도를 높이기 위해 스레드 수를 늘림

원리를 모를 때의 판단:
  "스레드가 많을수록 동시 처리가 많아진다"
  → Tomcat maxThreads=1000 설정
  → 부하 테스트 시 오히려 처리량 하락, GC 압박 증가

결과:
  스레드 1,000개 × 1MB 스택 = 1GB 메모리가 스택에만 소비
  OS 스케줄러가 1,000개 스레드를 관리하며 컨텍스트 스위칭 폭증
  실제 I/O 대기 중인 스레드가 CPU를 점유하지 않아도
  스케줄러는 모든 스레드를 주기적으로 확인 → CPU 낭비

또 다른 흔한 실수:
  Thread.sleep(1000) 을 활용하는 코드를 Thread Pool에서 실행
  → 슬리핑 중인 스레드도 OS 자원 점유
  → 풀의 모든 스레드가 sleep 중이면 새 요청이 큐에서 대기
  → 응답 지연 폭발

근본 원인을 모른 채:
  maxThreads를 더 늘리거나, 서버를 Scale Up하는 방향으로 접근
  → 문제를 해결하지 않고 비용만 증가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```
원리를 알 때의 설계 흐름:
  "스레드 하나 = OS 커널 스레드 하나 = 메모리 + 스케줄링 비용"
  → CPU 코어 수와 작업 특성에 맞게 Thread Pool 크기를 결정

CPU 바운드 작업:
  Thread Pool 크기 ≈ CPU 코어 수 (±1)
  → 더 많은 스레드는 컨텍스트 스위칭만 늘림
  적정값: Runtime.getRuntime().availableProcessors()

I/O 바운드 작업:
  Thread Pool 크기 = CPU 코어 수 × (1 + 대기시간/처리시간)
  예: 코어 8개, I/O 대기 90% → 8 × (1 + 9) = 80개
  → 하지만 이것도 수백 개를 넘기면 컨텍스트 스위칭이 압도
  → 근본 해결책은 Virtual Thread (Chapter 6)

스택 메모리 계산:
  -Xss 옵션으로 스레드당 스택 크기 조정
  기본값: 512KB ~ 1MB (OS/JVM 버전별 상이)
  Thread Pool 200개 × 512KB = 100MB (스택만)
  → 힙과 별도로 고려해야 하는 메모리 영역

진단 방법:
  jcmd <pid> Thread.print    # 현재 스레드 목록과 상태
  jstack <pid>               # Thread Dump (스레드 상태 스냅샷)
  cat /proc/<pid>/status | grep Threads  # OS 레벨 스레드 수
```

---

## 🔬 내부 동작 원리

### 1. 스레드 모델의 세 가지 방식

```
① 사용자 수준 스레드 (User-Level Thread, M:1 모델)
  ┌─────────────────────────────────┐
  │          사용자 프로세스           │
  │  Thread1  Thread2  Thread3      │
  │    ↓         ↓        ↓         │
  │  [사용자 수준 스케줄러]             │
  └───────────────┬─────────────────┘
                  │ 1개의 OS 스레드로 매핑
            ┌─────┴─────┐
            │  커널 스레드  │
            └───────────┘

  장점: 스레드 생성/전환이 빠름 (시스템콜 불필요)
  단점: 하나가 블로킹되면 전체 프로세스가 블로킹됨
       멀티코어 활용 불가

② 커널 수준 스레드 (Kernel-Level Thread, 1:1 모델)  ← Java가 선택한 방식
  ┌─────────────────────────────────┐
  │          사용자 프로세스           │
  │  Thread1  Thread2  Thread3      │
  └────┬──────────┬──────────┬──────┘
       │          │          │ 각각 1:1 매핑
  ┌────┴──┐  ┌────┴──┐  ┌────┴──┐
  │커널스레드│  │커널스레드│  │커널스레드│
  └───────┘  └───────┘  └───────┘
       ↓          ↓          ↓
  [OS 스케줄러가 CPU 코어에 배치]

  장점: 진정한 병렬 실행, 하나가 블로킹돼도 나머지는 실행 가능
  단점: 스레드 생성이 비쌈 (시스템콜), 스레드당 OS 자원 소비

③ 혼합 모델 (M:N 모델)
  ┌─────────────────────────────────┐
  │  사용자 스레드 M개                 │
  └───────────────┬─────────────────┘
                  │ N개의 OS 스레드로 매핑 (M > N)
         ┌────────┴────────┐
         │커널스레드  커널스레드│
         └─────────────────┘

  Go의 goroutine, Java 21 Virtual Thread가 이 방식
  → 사용자 스레드를 경량으로 다수 생성하고
    소수의 OS 스레드(캐리어 스레드)에 스케줄링
```

### 2. `new Thread().start()` 호출 시 내부 동작

```
Java 코드:
  Thread t = new Thread(() -> System.out.println("Hello"));
  t.start();

JVM 내부 경로:
  Thread.start()
    → JVM native 메서드 호출 (Thread.start0())
    → os::create_thread() [HotSpot 소스: os_linux.cpp]
    → pthread_create() 호출  ← POSIX 스레드 생성 API
    → 커널이 clone() 시스템콜 실행

clone() 시스템콜이 하는 일:
  1. 커널 스레드 자료구조(task_struct) 생성
  2. 스레드 스택 메모리 할당 (-Xss 크기, 기본 512KB~1MB)
  3. 스레드 ID(TID) 부여
  4. OS 스케줄러 실행 큐에 등록
  5. 새 스레드에서 JVM run() 메서드 실행

strace로 관찰:
  $ strace -e trace=clone java ThreadTest
  ...
  clone(child_stack=0x7f8abc000000, flags=CLONE_VM|CLONE_FS|
        CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|
        CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID,
        parent_tid=[12345], tls=0x7f8abc0007c0,
        child_tid=0x7f8abc000790) = 12346
  ← 각 Thread.start()마다 clone() 시스템콜 1회 발생

생성 비용:
  시스템콜 오버헤드: ~5~15 μs
  스택 메모리 할당: OS가 페이지를 예약 (실제 물리 메모리는 lazy 할당)
  커널 자료구조: task_struct ~수 KB
  → Thread.start() 하나에 수십 μs 소비
  → Thread Pool 없이 요청마다 new Thread()는 성능 문제
```

### 3. Java 스레드 하나가 소비하는 자원

```
메모리 구성:

┌──────────────────────────────────────────────────┐
│              Java Thread 하나의 자원               │
│                                                  │
│  JVM Heap 영역:                                  │
│    Thread 객체 자체: ~수백 바이트                   │
│    ThreadLocal 변수들: 사용량에 따라 다름             │
│                                                  │
│  Native 메모리 (Heap 외부):                        │
│    스레드 스택: -Xss 설정값 (기본 512KB~1MB)         │
│      └ Java 메서드 호출 스택 프레임 저장              │
│      └ 지역 변수, 매개변수                           │
│    OS 커널 자료구조 (task_struct): ~수 KB            │
│    JVM 내부 자료구조: ~수 KB                        │
│                                                  │
│  파일 디스크립터:                                   │
│    스레드 자체는 FD를 소비하지 않음                   │
│    (프로세스 공유)                                  │
└──────────────────────────────────────────────────┘

실제 메모리 계산:
  스레드 200개 × 1MB 스택 = 200MB (Native 메모리)
  → -Xmx 설정과 별개로 추가로 필요한 메모리
  → 컨테이너 메모리 한계 설정 시 반드시 고려

OS 스레드 수 제한:
  /proc/sys/kernel/threads-max        # 전체 스레드 수 한계
  /proc/sys/vm/max_map_count          # 메모리 맵 수 한계
  ulimit -u                           # 프로세스당 스레드 한계

Java 스레드 생성 한계 실험:
  스택 크기를 줄이면 더 많은 스레드 생성 가능
  java -Xss128k → 스레드당 128KB 스택 → 더 많이 생성 가능하나
  재귀 깊이 제한 발생 (StackOverflowError 빨리 발생)
```

### 4. 1:1 매핑의 한계 — Virtual Thread가 탄생한 이유

```
시나리오: HTTP 요청 처리 서버 (Thread per Request 모델)

  요청 1 → Thread 1: [DB 쿼리 실행 중... 20ms 대기]
  요청 2 → Thread 2: [외부 API 호출 중... 100ms 대기]
  요청 3 → Thread 3: [파일 읽기 중... 50ms 대기]
  ...
  요청 N → Thread N: [I/O 대기 중...]

Thread들이 대부분의 시간을 I/O 대기에 소비하는 동안:
  ① 스레드는 WAITING/TIMED_WAITING 상태
  ② CPU를 사용하지 않지만 스레드 자체는 살아있음
  ③ OS 스케줄러는 여전히 이 스레드를 관리해야 함
  ④ 스택 메모리 1MB는 여전히 점유 중

1,000 RPS 처리, 평균 응답시간 100ms(I/O 대기 90ms):
  동시에 진행 중인 요청 = 1,000 × 0.1 = 100개
  필요한 스레드 = 100개 (이상적)
  실제: 여유분 포함 200~500개 설정

문제는 10,000 RPS, 평균 응답 500ms 시나리오:
  동시 진행 중인 요청 = 10,000 × 0.5 = 5,000개
  5,000개 OS 스레드 × 1MB = 5GB (스택만!)
  OS 스케줄러 부하 폭발

Virtual Thread의 해결책:
  Java 스레드 → JVM Continuation 객체 (수 KB, Heap에 저장)
  소수의 OS 캐리어 스레드(CPU 코어 수)가 수백만 VT를 스케줄링
  I/O 대기 시 Continuation을 Heap에 저장하고 다른 VT 실행
  → OS 스레드 자원 소비 없이 수백만 동시 연결 처리 가능
  (자세한 내용은 Chapter 6에서 다룹니다)
```

---

## 💻 실전 실험

### 실험 1: new Thread() 호출 시 clone() 시스템콜 확인

```bash
# 실험용 Java 파일 작성
cat > ThreadTest.java << 'EOF'
public class ThreadTest {
    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            final int id = i;
            threads[i] = new Thread(() -> {
                System.out.println("Thread " + id + " running, TID: "
                    + Thread.currentThread().threadId());
                try { Thread.sleep(10000); } catch (InterruptedException e) {}
            });
            threads[i].start();
        }
        Thread.sleep(10000);
    }
}
EOF

# 컴파일 및 실행 (strace로 clone() 추적)
javac ThreadTest.java
strace -e trace=clone,exit_group -f java ThreadTest 2>&1 | grep clone

# 출력 예시 (스레드마다 clone() 한 번씩):
# [pid 12345] clone(child_stack=..., flags=CLONE_VM|...|CLONE_THREAD, ...) = 12346
# [pid 12345] clone(child_stack=..., flags=CLONE_VM|...|CLONE_THREAD, ...) = 12347
# [pid 12345] clone(child_stack=..., flags=CLONE_VM|...|CLONE_THREAD, ...) = 12348
```

### 실험 2: 스레드당 메모리 소비 측정

```bash
cat > ThreadMemoryTest.java << 'EOF'
import java.lang.management.ManagementFactory;

public class ThreadMemoryTest {
    public static void main(String[] args) throws InterruptedException {
        Runtime rt = Runtime.getRuntime();
        
        // 기준 메모리 측정
        rt.gc();
        long baseMemory = rt.totalMemory() - rt.freeMemory();
        System.out.printf("기준 메모리: %.1f MB%n", baseMemory / 1024.0 / 1024.0);
        System.out.printf("기준 스레드 수: %d%n",
            ManagementFactory.getThreadMXBean().getThreadCount());
        
        // 스레드 500개 생성
        Thread[] threads = new Thread[500];
        for (int i = 0; i < 500; i++) {
            threads[i] = new Thread(() -> {
                try { Thread.sleep(30000); } catch (InterruptedException e) {}
            });
            threads[i].start();
        }
        
        Thread.sleep(1000); // 안정화 대기
        rt.gc();
        long afterMemory = rt.totalMemory() - rt.freeMemory();
        int threadCount = ManagementFactory.getThreadMXBean().getThreadCount();
        
        System.out.printf("500개 생성 후 힙 메모리: %.1f MB (+%.1f MB)%n",
            afterMemory / 1024.0 / 1024.0,
            (afterMemory - baseMemory) / 1024.0 / 1024.0);
        System.out.printf("현재 스레드 수: %d%n", threadCount);
        System.out.println("※ 스택 메모리(Native)는 힙과 별도로 소비됩니다");
        System.out.printf("※ 스택 메모리 추정: 500 × 512KB = %.0f MB (Native 영역)%n",
            500 * 512.0 / 1024.0);
        
        // OS 레벨 확인 방법 출력
        long pid = ProcessHandle.current().pid();
        System.out.printf("%n# OS 레벨 스레드 확인:%n");
        System.out.printf("cat /proc/%d/status | grep Threads%n", pid);
        System.out.printf("ps -o pid,nlwp,vsz,rss -p %d%n", pid);
    }
}
EOF

# -Xss 크기를 다르게 해서 최대 스레드 수 비교
java -Xss1m -Xmx256m ThreadMemoryTest    # 1MB 스택
java -Xss256k -Xmx256m ThreadMemoryTest  # 256KB 스택 → 더 많이 생성 가능
```

### 실험 3: 스레드 생성 비용 측정

```bash
cat > ThreadCreationBench.java << 'EOF'
public class ThreadCreationBench {
    public static void main(String[] args) throws InterruptedException {
        int count = 1000;
        
        // Thread 생성 비용
        long start = System.nanoTime();
        Thread[] threads = new Thread[count];
        for (int i = 0; i < count; i++) {
            threads[i] = new Thread(() -> {});
        }
        long createTime = System.nanoTime() - start;
        
        // Thread 시작 비용 (start() = clone() 시스템콜)
        start = System.nanoTime();
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();
        long startTime = System.nanoTime() - start;
        
        System.out.printf("Thread 객체 생성 1000개: %.2f ms (평균 %.2f μs/개)%n",
            createTime / 1_000_000.0, createTime / 1_000.0 / count);
        System.out.printf("Thread.start() 1000개:  %.2f ms (평균 %.2f μs/개)%n",
            startTime / 1_000_000.0, startTime / 1_000.0 / count);
    }
}
EOF

javac ThreadCreationBench.java && java ThreadCreationBench

# 예상 출력:
# Thread 객체 생성 1000개: 0.84 ms (평균 0.84 μs/개)
# Thread.start() 1000개:  45.23 ms (평균 45.23 μs/개)
# → start()가 생성보다 ~50배 비쌈 (시스템콜 비용)
```

---

## 📊 성능/비용 비교

```
스레드 모델별 특성 비교:

특성                  | OS 스레드 (Platform Thread) | Virtual Thread (Java 21+)
─────────────────────┼───────────────────────────┼──────────────────────────
생성 비용              | ~수십 μs (clone() 시스템콜) | ~수 μs (Heap 객체 생성)
메모리 (스택)          | 512KB ~ 1MB (Native)       | 수 KB ~ 수십 KB (Heap)
최대 생성 수           | 수천 ~ 수만 (OS 한계)        | 수백만 (Heap 여유만큼)
컨텍스트 스위칭         | OS 스케줄러 (μs 단위)        | JVM 스케줄러 (더 가벼움)
멀티코어 활용           | OS가 코어에 직접 배치         | 캐리어 스레드 통해 배치
I/O 블로킹 시          | 스레드 전체 대기              | Unmount → 다른 VT 실행
synchronized 사용      | 자유롭게 사용 가능            | Pinning 주의 필요

스레드 수에 따른 처리량 변화 (I/O 바운드 워크로드 기준):

스레드 수    | 처리량 (상대적) | 비고
───────────┼──────────────┼────────────────────────────
10          | 100          | CPU 활용 미흡
50          | 420          | 점점 증가
100         | 700          | 코어 포화 시작 (8코어 기준)
200         | 820          | 컨텍스트 스위칭 오버헤드 증가
500         | 750          | 역전 — 오버헤드가 이득을 초과
1000        | 620          | 메모리 압박 + 스케줄링 오버헤드
```

---

## ⚖️ 트레이드오프

```
Java가 1:1 매핑(커널 스레드)을 선택한 이유:
  ① 진정한 멀티코어 병렬 실행 보장
  ② OS 스케줄러의 성숙한 CPU 배치 알고리즘 활용
  ③ 하나의 스레드가 블로킹되어도 다른 스레드는 계속 실행
  ④ 구현 단순성 (OS가 스케줄링 전담)

1:1 매핑의 한계:
  ① 스레드당 메모리 비용이 크다 (1MB 스택 × N)
  ② 스레드 생성/소멸 비용 (시스템콜)
  ③ 컨텍스트 스위칭 비용 (OS 스케줄러 개입)
  ④ 수천 개 이상의 스레드에서 성능 역전

언제 OS 스레드를 그대로 써야 하는가:
  - CPU 바운드 작업 (코어 수에 맞춘 Thread Pool)
  - synchronized 블록을 많이 사용하는 레거시 코드
  - OS 스레드의 스케줄링 우선순위를 조절해야 할 때
  - Java 21 미만 환경

언제 Virtual Thread로 전환해야 하는가:
  - I/O 바운드 작업이 대부분인 서버
  - 동시 연결 수가 수천 ~ 수만을 넘어서는 경우
  - Thread Pool 튜닝이 점점 어려워지는 상황
  → Chapter 6에서 Virtual Thread 상세 분석
```

---

## 📌 핵심 정리

```
OS 스레드 모델과 Java 스레드 핵심:

Java 스레드 = OS 커널 스레드 (1:1 매핑)
  new Thread().start()
    → JVM native → pthread_create() → clone() 시스템콜
    → OS 커널 스레드 생성, OS 스케줄러 관리

스레드 하나의 비용:
  스택 메모리: 512KB ~ 1MB (Native, Heap 외부)
  생성 비용: ~수십 μs (시스템콜 포함)
  OS 자료구조: task_struct ~수 KB

한계:
  수천 개 이상의 스레드에서 성능 역전
  메모리: 1,000 스레드 × 1MB = 1GB (스택만)
  컨텍스트 스위칭 오버헤드 폭증

설계 원칙:
  CPU 바운드: Thread Pool = CPU 코어 수
  I/O 바운드: Thread Pool = 코어 수 × (1 + 대기/처리 비율)
  극한 I/O 동시성: Virtual Thread (Java 21)

진단:
  jcmd <pid> Thread.print    # 스레드 목록과 상태
  cat /proc/<pid>/status | grep Threads  # OS 레벨 스레드 수
```

---

## 🤔 생각해볼 문제

**Q1.** `-Xss` 옵션으로 스레드 스택 크기를 줄이면 더 많은 스레드를 만들 수 있다. 그렇다면 `-Xss64k`처럼 극단적으로 줄이면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

스택 크기를 줄이면 각 스레드의 **호출 스택 깊이가 줄어든다**. 재귀 호출이나 깊은 메서드 체인에서 `StackOverflowError`가 훨씬 빨리 발생한다.

실용적인 최솟값:
- 단순 작업 스레드: 256KB~512KB
- 재귀가 없고 호출 체인이 얕은 코드: 128KB 가능
- Spring/Hibernate 같은 프레임워크 사용 시: 프레임워크 자체 스택 소비가 크므로 기본값 권장

`-Xss64k`는 JVM이 자체적으로 거부하거나 즉시 `StackOverflowError`를 낸다. 실무에서 `-Xss256k`~`-Xss512k` 범위가 메모리 절약과 안전성의 균형점이다.

</details>

---

**Q2.** `Thread.start()`와 `Thread.run()`의 차이는 무엇인가? `run()`을 직접 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`start()`는 새 OS 스레드를 생성하고 그 스레드에서 `run()`을 실행한다. `clone()` 시스템콜이 발생하고 병렬로 실행된다.

`run()`을 직접 호출하면 **현재 스레드에서 단순 메서드 호출**이 된다. 새 OS 스레드가 생성되지 않으며 순차 실행된다.

```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));
t.start(); // 출력: Thread-0 (새 스레드)
t.run();   // 출력: main (현재 스레드에서 실행)
```

실무에서 `run()` 직접 호출은 멀티스레딩이 전혀 일어나지 않는 버그를 만든다. Runnable 구현체를 테스트할 때 의도적으로 쓰는 경우를 제외하고는 항상 `start()`를 사용해야 한다.

</details>

---

**Q3.** Java 프로세스의 스레드 수를 OS 명령어로 확인하는 방법이 여러 가지다. `ps`, `/proc/pid/status`, `jstack`이 보여주는 스레드 수가 다를 수 있는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

각 도구가 보여주는 스레드의 범위가 다르기 때문이다.

- `/proc/<pid>/status`의 `Threads` 필드: OS가 관리하는 **모든 커널 스레드 수** (JVM 내부 스레드 포함)
- `ps -o nlwp`: 동일하게 OS 레벨 커널 스레드 수
- `jstack` / `jcmd Thread.print`: **JVM이 인식하는 Java 스레드만** 출력 (JVM GC 스레드, Compiler 스레드 등은 포함되나 일부 native 스레드는 제외될 수 있음)
- `ManagementFactory.getThreadMXBean().getThreadCount()`: Java 스레드만 카운트

JVM이 시작할 때 GC 스레드(ZGC, G1 등), JIT Compiler 스레드, Finalizer 스레드 등이 추가로 생성되어 OS 레벨 스레드 수가 Java 코드에서 생성한 수보다 항상 더 많다. Java 21에서 ZGC 사용 시 10~20개의 내부 스레드가 기본으로 생성된다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 컨텍스트 스위칭 비용의 실체 ➡️](./02-context-switching-cost.md)**

</div>
