# 블로킹 시 동작 — Unmount와 NIO 재작성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Socket.read()` 같은 블로킹 I/O를 JDK 내부에서 NIO non-blocking으로 변환하는 과정은?
- 데이터가 없으면 Continuation을 저장하고 Unmount하는 전체 흐름은?
- 데이터 도착 시 어떻게 VT가 다시 Mount되는가?
- `Thread.sleep()`이 Virtual Thread에서 캐리어를 블로킹하지 않는 이유는?
- 어떤 JDK API가 VT Unmount를 지원하고 어떤 것이 지원하지 않는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Virtual Thread를 쓰면 블로킹 I/O가 자동으로 Non-Blocking이 된다"는 말이 실제로 무엇을 의미하는지 이해하는 것이 중요하다. JDK 21에서 재작성된 API와 그렇지 않은 API를 구별해야 Pinning을 예방할 수 있고, 어떤 서드파티 라이브러리가 VT와 완전히 호환되는지 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 모든 블로킹 호출이 자동으로 Unmount된다는 오해
  "VT에서 블로킹하면 다 자동으로 Non-Blocking이 되겠지"
  → 아님! JDK가 재작성한 API만 Unmount 가능
  → 예: sun.misc.Unsafe.park() → JDK가 VT 인식하여 Unmount
  → 예: old-style blocking I/O (FileInputStream 일부) → 지원 제한적

실수 2: Thread.sleep()이 VT에서도 캐리어를 블로킹한다는 가정
  "sleep()은 OS 레벨 blocking이니 캐리어가 멈추겠지"
  → VT에서 Thread.sleep() = LockSupport.parkNanos() 내부 사용
  → LockSupport.park()는 VT 인식 → Unmount 발생
  → 캐리어가 sleep 동안 다른 VT를 실행

실수 3: JDBC 드라이버가 자동으로 VT 호환된다는 가정
  "JDK를 21로 올리면 JDBC도 VT-friendly하게 되겠지"
  → JDBC 드라이버가 내부적으로 synchronized 사용 시 Pinning 발생
  → PostgreSQL JDBC: 일부 버전에서 Pinning 있음
  → 드라이버별 VT 호환성 확인 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// VT와 잘 동작하는 패턴

// 패턴 1: java.net.Socket (JDK 21에서 재작성)
Thread.ofVirtual().start(() -> {
    try (Socket socket = new Socket("example.com", 80)) {
        byte[] buf = new byte[4096];
        socket.getInputStream().read(buf);  // Unmount → 데이터 대기
        // 데이터 도착 시 자동으로 Mount, 실행 재개
    } catch (Exception e) { e.printStackTrace(); }
});

// 패턴 2: Thread.sleep() (VT에서 Unmount)
Thread.ofVirtual().start(() -> {
    System.out.println("sleep 전 캐리어: running");
    Thread.sleep(1000);  // VT Unmount (캐리어 해방)
    System.out.println("sleep 후: 다시 Mount");
});

// 패턴 3: java.util.concurrent (모두 LockSupport.park 기반, VT 호환)
BlockingQueue<String> queue = new LinkedBlockingQueue<>();
Thread.ofVirtual().start(() -> {
    String item = queue.take();  // Unmount → 아이템 대기
});

// 패턴 4: HttpClient (Java 11+, VT 호환)
HttpClient httpClient = HttpClient.newHttpClient();
Thread.ofVirtual().start(() -> {
    HttpResponse<String> resp = httpClient.send(
        HttpRequest.newBuilder(URI.create("https://api.example.com")).build(),
        HttpResponse.BodyHandlers.ofString()
    );  // Unmount → 응답 대기
});

// 주의가 필요한 패턴 (Pinning 위험)
// synchronized 내 블로킹 → 반드시 리팩토링
// synchronized (lock) {
//     Thread.sleep(1000);  // Pinning! synchronized 제거 또는 ReentrantLock으로
// }
```

---

## 🔬 내부 동작 원리

### 1. JDK 21에서 재작성된 I/O 경로

```
Socket.read() VT 경로 (JDK 21):

  InputStream.read(buf) 호출
      ↓
  NioSocketImpl.read() (JDK 21 새 구현체)
      ↓
  ① SelectableChannel.read() 시도 (NIO non-blocking)
      ↓ 데이터 있음           ↓ 데이터 없음 (EAGAIN/EWOULDBLOCK)
  즉시 반환 (데이터 읽기)   ② Poller에 등록 (select/epoll)
                                 ↓
                            ③ LockSupport.park(continuation) → VT Unmount
                                 ↓ (캐리어는 다른 VT 실행)
                            ④ Poller: 데이터 도착 감지 (epoll_wait)
                                 ↓
                            ⑤ LockSupport.unpark(vt) → VT 재스케줄링
                                 ↓
                            ⑥ VT Mount → read() 재시도 → 데이터 읽기 → 반환

핵심 구성 요소:
  NioSocketImpl: Socket을 NIO로 구현 (VT 지원)
  Poller: OS별 이벤트 감시 (epoll/kqueue/select)
          I/O 이벤트 → LockSupport.unpark
  LockSupport.park: VT 인식 → Unmount 트리거

JDK 21에서 재작성된 주요 API:
  ✅ java.net.Socket, ServerSocket
  ✅ java.net.SocketChannel (NIO, 기존에도 호환)
  ✅ java.io.InputStream / OutputStream (Socket 기반)
  ✅ Thread.sleep()
  ✅ Object.wait()
  ✅ java.util.concurrent.* (AQS 기반)

부분 지원 또는 주의 필요:
  ⚠️ FileInputStream/FileOutputStream (파일 I/O)
     → 일부 구현에서 NIO로 전환 안 됨 → Pinning 가능
  ⚠️ 구형 socket 구현 (특정 플랫폼)

미지원 (Pinning):
  ❌ synchronized 블록 내 모든 park
  ❌ JNI (네이티브 코드) 실행 중
```

### 2. Thread.sleep()의 VT 동작

```java
// Thread.sleep() 내부 (JDK 21, VT 경로)
public static void sleep(long millis) throws InterruptedException {
    if (Thread.currentThread().isVirtual()) {
        // VT 전용 경로
        VirtualThread.sleepNanos(millis * 1_000_000L);
        return;
    }
    // OS 스레드 경로 (기존)
    sleep0(millis);
}

// VirtualThread.sleepNanos 내부 (단순화)
void sleepNanos(long nanos) throws InterruptedException {
    // 내부적으로 타이머 + LockSupport.parkNanos 사용
    long deadline = System.nanoTime() + nanos;

    // Poller에 타이머 이벤트 등록
    ScheduledFuture<?> timer = timerPool.schedule(
        () -> LockSupport.unpark(this),  // 시간 지나면 unpark
        nanos, NANOSECONDS
    );

    try {
        LockSupport.parkNanos(nanos);  // VT Unmount!
        // 캐리어 해방 → 다른 VT 실행
        // 타이머 만료 → unpark → VT Mount → 여기서 재개
    } finally {
        timer.cancel(false);
    }
}

결과:
  OS 스레드 sleep: OS 레벨 blocking → 해당 OS 스레드 잠
  VT sleep: LockSupport.park → VT Unmount → 캐리어 해방
  → VT 100만 개가 sleep 중이어도 캐리어 스레드는 자유
```

### 3. Object.wait()의 VT 동작

```java
// Object.wait() 내부 (JDK 21)
// VT에서 호출 시:
//   synchronized (lock) {
//       lock.wait();  // ← 여기서 어떻게 동작?
//   }

// 문제: wait()는 synchronized 블록 내에서 호출됨
//       synchronized → JVM 모니터 → VT가 캐리어와 연결
//       wait()는 모니터를 해제하고 대기...
//
// JDK 21 구현:
//   Object.wait() 내부에서 JVM 모니터를 해제
//   VT를 WaitSet에 추가
//   LockSupport.park() → Unmount 가능!
//   (synchronized 블록을 '탈출'한 상태이므로 Unmount 허용)
//
//   notify() 시: WaitSet에서 꺼내 unpark → VT 재실행
//               모니터 재획득 → synchronized 블록 재진입

// 주의: Object.wait()는 VT에서 Unmount 가능
//       하지만 synchronized 블록에서 다른 blocking 호출은 Pinning!
//       ex) synchronized(lock) { socket.read(); }  → Pinning
//       synchronized(lock) { lock.wait(); }        → Unmount OK

// 따라서: synchronized + I/O 블로킹은 Pinning
//         synchronized + wait()는 Unmount (예외)
```

### 4. Poller와 I/O 이벤트 모델

```
JDK 21 Poller 구조:

┌─────────────────────────────────────────────────────────────┐
│  Poller (OS별 구현)                                         │
│                                                             │
│  Linux:   epoll_wait()                                      │
│  macOS:   kqueue()                                          │
│  Windows: I/O Completion Ports (IOCP)                       │
│                                                             │
│  등록: fd(파일 디스크립터) + VT 매핑                         │
│  이벤트 루프: fd 이벤트 감지 → 해당 VT unpark              │
└─────────────────────────────────────────────────────────────┘

Socket.read() 흐름:
  VT: fd = socket.fd, NIO read → EAGAIN (데이터 없음)
      → Poller.register(fd, READ_INTEREST, thisVT)
      → LockSupport.park() → Unmount

  Poller: epoll_wait([fd → VT, ...]) 블로킹 대기
          fd에 READ 이벤트 도착
          → VT = mapping[fd]
          → LockSupport.unpark(VT)
          → ForkJoinPool에 VT 스케줄링

  VT: Mount → read() 재시도 → 데이터 읽음 → 반환

Poller 스레드:
  백그라운드 데몬 스레드 (OS 스레드)
  CPU를 epoll_wait에서 blocking으로 대기
  이벤트 도착 시 해당 VT를 깨움
  → 수백만 VT의 I/O를 소수의 Poller 스레드가 처리
  → 이것이 Reactive의 Event Loop와 유사한 원리
     (하지만 코드를 Reactive로 바꿀 필요 없음)
```

---

## 💻 실전 실험

### 실험 1: sleep()이 캐리어를 블로킹하지 않음을 확인

```java
public class SleepUnmountVerify {
    public static void main(String[] args) throws InterruptedException {
        int carriers = Runtime.getRuntime().availableProcessors();
        System.out.println("캐리어 스레드 수: " + carriers);

        // 캐리어보다 훨씬 많은 VT를 sleep 시킴
        int vtCount = carriers * 100;
        Thread[] vts = new Thread[vtCount];

        long start = System.nanoTime();
        for (int i = 0; i < vtCount; i++) {
            vts[i] = Thread.ofVirtual().start(() -> {
                try {
                    Thread.sleep(1000);  // VT Unmount (캐리어 해방)
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        // 모든 VT가 sleep 완료될 때까지 대기
        for (Thread vt : vts) vt.join();

        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.printf("%d개 VT, 1000ms sleep: %d ms%n", vtCount, elapsed);
        // 예상: ~1000ms (모두 동시에 sleep → 캐리어가 해방되어 대기)
        // OS 스레드로 같은 수를 만들면 OOM 또는 수천 ms

        // 비교: OS 스레드 sleep (캐리어 블로킹)
        // carriers * 100 OS 스레드를 sleep시키면
        // carriers 이상의 OS 스레드가 블로킹 상태로 메모리 차지
    }
}
```

### 실험 2: Poller 동작 추적

```bash
# JFR로 VT I/O 이벤트 기록
cat > VtIoTrace.java << 'EOF'
import java.io.*;
import java.net.*;

public class VtIoTrace {
    public static void main(String[] args) throws Exception {
        // 간단한 TCP 에코 서버
        ServerSocket server = new ServerSocket(19999);

        // 서버 VT
        Thread.ofVirtual().start(() -> {
            try {
                Socket client = server.accept();
                byte[] buf = new byte[128];
                int n = client.getInputStream().read(buf);
                client.getOutputStream().write(buf, 0, n);
                client.close();
            } catch (Exception e) { e.printStackTrace(); }
        });

        // 클라이언트 VT
        Thread.ofVirtual().start(() -> {
            try {
                Socket s = new Socket("localhost", 19999);
                Thread.sleep(100);  // 잠깐 대기
                s.getOutputStream().write("hello".getBytes());
                byte[] buf = new byte[128];
                int n = s.getInputStream().read(buf);
                System.out.println("받음: " + new String(buf, 0, n));
                s.close();
            } catch (Exception e) { e.printStackTrace(); }
        });

        Thread.sleep(1000);
        server.close();
    }
}
EOF
javac VtIoTrace.java

# JFR로 VT 이벤트 기록
java -XX:StartFlightRecording=duration=10s,filename=vt-io.jfr \
     VtIoTrace

# JFR 분석: jdk.VirtualThreadStart, jdk.VirtualThreadEnd
# jfr print --events jdk.VirtualThreadStart,jdk.VirtualThreadEnd vt-io.jfr
```

### 실험 3: 지원/미지원 API 비교

```java
import java.util.concurrent.*;

public class VtCompatibilityTest {
    public static void main(String[] args) throws Exception {
        int TASKS = 100;
        long ioDelay = 100;  // ms

        // ===== 지원: Thread.sleep() =====
        long start = System.nanoTime();
        try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = new java.util.ArrayList<Future<?>>();
            for (int i = 0; i < TASKS; i++) {
                futures.add(exe.submit(() -> {
                    Thread.sleep(ioDelay);
                    return null;
                }));
            }
            for (var f : futures) f.get();
        }
        System.out.printf("sleep(): %d ms (기대 ~%d ms) ✅%n",
            (System.nanoTime() - start) / 1_000_000, ioDelay);

        // ===== 지원: LinkedBlockingQueue.take() =====
        start = System.nanoTime();
        BlockingQueue<String> queue = new LinkedBlockingQueue<>();
        CountDownLatch latch = new CountDownLatch(TASKS);
        try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < TASKS; i++) {
                exe.submit(() -> {
                    queue.take();  // Unmount
                    latch.countDown();
                    return null;
                });
            }
            Thread.sleep(ioDelay);
            for (int i = 0; i < TASKS; i++) queue.offer("item");
            latch.await();
        }
        System.out.printf("BlockingQueue.take(): %d ms ✅%n",
            (System.nanoTime() - start) / 1_000_000);
    }
}
```

---

## 📊 성능/비용 비교

```
VT 지원 API vs 미지원 비교:

API 종류                        | Unmount | 비고
───────────────────────────────┼─────────┼─────────────────────────────
Thread.sleep(n)                 | ✅       | parkNanos 사용
Object.wait()                  | ✅       | JVM 모니터 해제 후 park
LockSupport.park()             | ✅       | 기본 park
Condition.await()              | ✅       | AQS park
BlockingQueue.take/put         | ✅       | AQS park
ReentrantLock.lock() 대기      | ✅       | AQS park
Semaphore.acquire()            | ✅       | AQS park
CountDownLatch.await()         | ✅       | AQS park
Socket.read/write              | ✅       | JDK 21 NIO 재작성
HttpClient.send() (Java 11+)   | ✅       | NIO 기반
SocketChannel.read/write       | ✅       | NIO (기존에도 호환)
synchronized 내 park           | ❌ Pinning| JVM 모니터 구현 제약
JNI 실행 중                    | ❌ Pinning| 네이티브 코드
FileInputStream.read()         | ⚠️ 일부   | 플랫폼/버전별 상이

처리량 시나리오 (100ms I/O, 1000 요청):
  OS 스레드 풀 10개:   ~10,000ms (직렬화)
  OS 스레드 풀 200개:  ~500ms
  VT (Unmount 지원):   ~100ms (모두 동시 처리)
  VT (Pinning 발생):   ~OS 스레드와 유사 (캐리어 고착)
```

---

## ⚖️ 트레이드오프

```
NIO 재작성의 트레이드오프:

장점:
  기존 블로킹 API 코드 변경 없이 Non-Blocking 효과
  소수의 Poller 스레드로 수백만 I/O 이벤트 처리
  OS의 epoll/kqueue/IOCP를 완전 활용

단점:
  JDK가 재작성하지 않은 API는 Pinning
  서드파티 라이브러리 의존 (JDBC 드라이버, 네이티브 라이브러리)
  파일 I/O는 NIO 완전 전환이 어려움 (OS 제약)

서드파티 라이브러리 호환성:
  호환:
    HikariCP (연결 풀): VT 호환 (자체 synchronized 최소화)
    R2DBC: Reactive이므로 VT와 무관 (단 혼용 주의)
    Netty: 자체 이벤트 루프 → VT와 혼용 시 중복 오버헤드
  주의:
    구형 JDBC 드라이버 (PostgreSQL < 42.6.0): synchronized 많음
    일부 JNI 라이브러리: Pinning 불가피

실용적 가이드:
  JDK 표준 API (java.net, java.util.concurrent): 안전
  JDBC: 최신 버전 확인, -Djdk.tracePinnedThreads=full로 확인
  Netty 직접 사용: VT와 혼용 비권장, 둘 중 하나 선택
```

---

## 📌 핵심 정리

```
VT Unmount / NIO 재작성 핵심:

블로킹 I/O → Non-Blocking 변환 과정:
  Socket.read() → NIO read → EAGAIN
  → Poller에 fd 등록 + LockSupport.park → Unmount
  → Poller: epoll_wait → 이벤트 → LockSupport.unpark
  → Mount → read 재시도 → 데이터 반환

Thread.sleep() VT 동작:
  LockSupport.parkNanos → Unmount
  타이머 만료 → unpark → Mount
  캐리어는 sleep 동안 다른 VT 실행 (블로킹 없음)

Object.wait() 예외:
  synchronized 탈출 → 모니터 해제 → park → Unmount OK
  vs synchronized 내 다른 블로킹 → Pinning

Poller:
  OS별 이벤트 루프 (epoll/kqueue/IOCP)
  fd → VT 매핑
  이벤트 도착 시 해당 VT unpark
  수백만 I/O를 소수 Poller 스레드로 처리

미지원 (Pinning):
  synchronized 블록 내 park
  JNI 실행 중
  → ReentrantLock으로 교체가 해결책
```

---

## 🤔 생각해볼 문제

**Q1.** VT에서 `FileInputStream.read()`를 호출하면 Unmount가 발생하는가?

<details>
<summary>해설 보기</summary>

파일 I/O는 소켓 I/O와 다르게 처리된다. 소켓은 epoll/kqueue로 비동기 이벤트를 감지할 수 있지만, 일반 파일은 "항상 데이터가 있다"고 간주되어 non-blocking read가 즉시 반환한다(데이터가 캐시에 없어도 OS가 블로킹하며 채워줌).

JDK 21에서 `FileInputStream`은 완전히 NIO로 재작성되지 않았다. 플랫폼과 파일 유형에 따라:
- 빠른 SSD에서 캐시된 파일: 거의 즉시 반환 → Pinning 영향 미미
- 느린 I/O (NFS, HDD): 블로킹 → Pinning 발생 가능

실용적 접근: 대용량 파일 I/O가 많은 경우 VT 대신 별도의 I/O 스레드 풀을 사용하거나, `AsynchronousFileChannel` (NIO2)를 직접 사용한다. 일반적인 설정 파일 읽기나 캐시된 파일은 영향이 없다.

</details>

---

**Q2.** Poller가 `epoll_wait()`로 대기하는 동안, 수백만 개의 소켓 fd를 동시에 감시하는 것이 성능 문제가 되지 않는가?

<details>
<summary>해설 보기</summary>

`epoll_wait()`는 Linux의 이벤트 알림 시스템으로, 감시 중인 fd 수에 O(1)로 동작한다 (초기 등록은 O(log n)). 즉 1,000,000개의 fd를 감시해도 이벤트 대기 자체는 O(1)이다.

Nginx, Node.js, Redis가 이 방식으로 수만~수백만 동시 연결을 처리한다. JDK의 Poller도 동일한 원리를 사용한다.

다만 실제 시스템 한계:
- `fs.nr_open` 등 OS 파일 디스크립터 제한 (`ulimit -n`)
- epoll 등록/해제 시 커널 메모리 사용
- 이벤트가 폭발적으로 발생하면 Poller 처리 병목

실무에서 수백만 동시 연결 시: `ulimit -n 1048576` 등 OS 파라미터 튜닝이 필요하다.

</details>

---

**Q3.** Virtual Thread를 사용하는 서비스에서 타임아웃이 필요하면 어떻게 구현하는가?

<details>
<summary>해설 보기</summary>

VT에서 타임아웃은 Java의 표준 방법으로 구현한다:

```java
// 방법 1: ExecutorService.submit + Future.get(timeout)
try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> future = exe.submit(() -> {
        return httpClient.send(req, bodyHandler).body();
    });
    try {
        return future.get(5, TimeUnit.SECONDS);  // 5초 타임아웃
    } catch (TimeoutException e) {
        future.cancel(true);  // VT 인터럽트
        throw new ServiceTimeoutException();
    }
}

// 방법 2: HttpClient에 타임아웃 설정
HttpRequest request = HttpRequest.newBuilder(uri)
    .timeout(Duration.ofSeconds(5))  // 요청 타임아웃
    .build();
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(3))  // 연결 타임아웃
    .build();

// 방법 3: Thread.ofVirtual + join(timeout)
Thread vt = Thread.ofVirtual().start(task);
boolean finished = vt.join(Duration.ofSeconds(5));
if (!finished) {
    vt.interrupt();
    throw new TimeoutException();
}
```

VT와 인터럽트:
- `future.cancel(true)` → VT에 `interrupt()` 전송
- VT가 park/sleep/I/O 대기 중 → `InterruptedException` 발생
- VT가 CPU 바운드 루프 중 → `Thread.interrupted()` 확인 필요

</details>

---

<div align="center">

**[⬅️ 이전: Continuation과 스케줄링](./02-continuation-scheduling.md)** | **[홈으로 🏠](../README.md)** | **[다음: Pinning 문제 ➡️](./04-pinning-problem.md)**

</div>
