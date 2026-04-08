# 데몬 스레드와 JVM 종료 — ShutdownHook 실행 보장

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JVM은 언제 종료되는가? 어떤 조건이 충족돼야 종료가 시작되는가?
- 데몬 스레드와 non-daemon 스레드의 차이는 무엇인가?
- 데몬 스레드가 강제 종료될 때 리소스 정리가 안 되는 이유는?
- `Runtime.addShutdownHook()`이 실행되는 타이밍과 보장 범위는?
- 애플리케이션의 Graceful Shutdown을 어떻게 설계하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

JVM이 종료될 때 "왜 파일이 완전히 쓰이지 않았는가", "왜 트랜잭션이 롤백됐는가", "왜 로그 마지막 부분이 유실됐는가" — 이런 문제는 대부분 데몬 스레드와 ShutdownHook의 동작 원리를 이해하지 못한 데서 온다. Kubernetes에서 `SIGTERM`을 받았을 때 서버가 어떻게 종료되는지도 이 원리에 기반한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 배경 작업 스레드를 daemon으로 만들지 않음
  Thread t = new Thread(longRunningTask);
  t.start();  // daemon = false (기본값)
  // main이 종료되어도 JVM은 t가 끝날 때까지 대기
  // "프로그램이 종료되지 않는다" 문제 발생

실수 2: 데몬 스레드에 리소스 정리 로직 의존
  Thread cleanup = new Thread(() -> {
      // DB 연결 종료, 파일 close, 캐시 flush
  });
  cleanup.setDaemon(true);  // JVM 종료 시 강제 종료됨
  cleanup.start();
  // JVM 종료 시 cleanup 스레드가 실행 중에도 강제 종료
  // → DB 연결 누수, 파일 손상 발생

실수 3: ShutdownHook에서 너무 많은 작업
  Runtime.getRuntime().addShutdownHook(new Thread(() -> {
      // 수 분이 걸리는 데이터 저장 작업
      // JVM 강제 종료 시 ShutdownHook도 중단됨
  }));

실수 4: ShutdownHook에서 데드락
  Runtime.getRuntime().addShutdownHook(new Thread(() -> {
      synchronized (someObject) { ... }  // 다른 ShutdownHook과 경합
  }));
  // ShutdownHook들은 병렬 실행 → 락 경합 → 데드락 → JVM 종료 블로킹
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Graceful Shutdown 올바른 설계 패턴
public class GracefulServer {
    private final ExecutorService workerPool;
    private volatile boolean running = true;

    public GracefulServer() {
        this.workerPool = new ThreadPoolExecutor(
            10, 50, 60L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(200),
            r -> {
                Thread t = new Thread(r, "worker-" + counter.incrementAndGet());
                t.setDaemon(false);  // non-daemon: 작업 완료 보장
                return t;
            },
            new ThreadPoolExecutor.CallerRunsPolicy()
        );

        // ShutdownHook: 종료 신호(SIGTERM 등) 수신 시 처리
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("종료 신호 수신. Graceful Shutdown 시작...");
            running = false;  // 새 작업 수락 중단

            // Thread Pool 종료 (실행 중인 작업 완료 대기)
            workerPool.shutdown();
            try {
                if (!workerPool.awaitTermination(30, TimeUnit.SECONDS)) {
                    System.err.println("30초 내 완료 안 됨. 강제 종료.");
                    workerPool.shutdownNow();
                }
            } catch (InterruptedException e) {
                workerPool.shutdownNow();
            }
            System.out.println("Graceful Shutdown 완료.");
        }, "shutdown-hook"));
    }
}
```

---

## 🔬 내부 동작 원리

### 1. JVM 종료 조건

```
JVM은 다음 조건 중 하나가 충족될 때 종료 시퀀스를 시작한다:

정상 종료 (Orderly Shutdown):
  ① main 스레드 종료 후 non-daemon 스레드가 모두 종료됨
     → 가장 일반적인 종료 조건
  ② System.exit(status) 호출
     → 어느 스레드에서든 호출 가능
     → 즉시 종료 시퀀스 시작

강제 종료 (Abrupt Shutdown):
  ③ Runtime.halt(status) 호출
     → ShutdownHook 실행 안 함
     → 즉시 JVM 프로세스 종료
  ④ OS 시그널 수신
     SIGTERM: 정상 종료 요청 → ShutdownHook 실행
     SIGKILL: 강제 종료 → ShutdownHook 실행 안 함 (OS가 즉시 프로세스 종료)
     Ctrl+C: SIGINT → ShutdownHook 실행

JVM 종료 시퀀스:
  1단계: 새로운 ShutdownHook 등록 불가 상태로 전환
  2단계: 모든 등록된 ShutdownHook을 병렬로 시작
  3단계: 모든 ShutdownHook 완료 대기
  4단계: uninvoked finalizer 실행 (runFinalizersOnExit가 true인 경우, 비권장)
  5단계: JVM 프로세스 종료
```

### 2. 데몬 스레드 vs Non-Daemon 스레드

```
Non-Daemon 스레드 (기본값):
  JVM은 non-daemon 스레드가 살아있는 동안 종료하지 않음
  new Thread()로 생성한 스레드의 기본값: setDaemon(false)
  Thread Pool 스레드: 팩토리에 따라 다름 (Executors 기본: non-daemon)

Daemon 스레드:
  thread.setDaemon(true);  // start() 이전에 호출해야 함
  JVM 종료를 막지 않음
  JVM이 종료 결정 시 daemon 스레드는 강제 종료
  → finally 블록도 실행되지 않을 수 있음
  → try-with-resources의 close()도 미보장

  GC 스레드, JIT Compiler 스레드: JVM 내부 daemon 스레드
  Finalizer 스레드: daemon
  Signal Dispatcher: daemon

상속 규칙:
  새 스레드는 생성한 스레드의 daemon 상태를 상속
  main 스레드: non-daemon
  main에서 new Thread() → non-daemon (상속)
  daemon 스레드에서 new Thread() → daemon (상속)

Thread Pool에서:
  Executors.newFixedThreadPool()
    → DefaultThreadFactory 사용
    → 생성 스레드가 daemon이면 → Pool 스레드도 daemon
    → 생성 스레드가 non-daemon이면 → Pool 스레드도 non-daemon
  → main 스레드에서 Pool 생성 → non-daemon Pool 스레드

데몬 스레드 사용이 적합한 경우:
  ① 백그라운드 통계 수집 (데이터 유실 허용)
  ② 로그 비동기 전송 (마지막 로그 유실 허용)
  ③ 캐시 갱신 스레드 (다음 시작 시 재로드 가능)
  ④ Heartbeat 전송 스레드

데몬 스레드 사용이 부적합한 경우:
  ① DB 트랜잭션 처리 (롤백 없이 종료되면 데이터 오염)
  ② 파일 쓰기 (불완전한 파일 생성 위험)
  ③ 결제 처리, 주문 처리 등 중요 비즈니스 로직
```

### 3. ShutdownHook 동작 상세

```
ShutdownHook 등록:
  Runtime.getRuntime().addShutdownHook(Thread hook)
  → hook 스레드는 start()되지 않은 상태여야 함
  → JVM 종료 시 JVM이 start() 호출

ShutdownHook 실행 보장 범위:
  ✅ System.exit() 호출 시
  ✅ SIGTERM 수신 시 (Ctrl+C 포함)
  ✅ main 스레드 종료로 인한 JVM 정상 종료 시
  ❌ SIGKILL (-9) 수신 시 → OS가 즉시 프로세스 종료
  ❌ Runtime.halt() 호출 시
  ❌ JVM 내부 오류 (예: OutOfMemoryError 심각) 시 미보장

ShutdownHook 실행 방식:
  모든 ShutdownHook이 병렬로 시작됨 (순서 보장 없음)
  모든 Hook 완료 시까지 JVM 종료 대기
  Hook이 데드락에 빠지면 JVM이 영원히 종료 안 됨

ShutdownHook 내부 주의사항:
  ① 동기화 주의: 여러 Hook이 병렬 실행 → 공유 자원 접근 시 synchronized
  ② 실행 중인 스레드 고려: 애플리케이션 스레드가 아직 실행 중일 수 있음
  ③ 시간 제한: Kubernetes terminationGracePeriodSeconds (기본 30초)
     → 30초 내 종료 안 되면 SIGKILL → Hook 강제 종료
  ④ 로깅 시스템: Hook에서 Logger 사용 가능하나 로깅 시스템 종료 후 호출되면 문제
```

### 4. Kubernetes/컨테이너 환경의 종료 시퀀스

```
Kubernetes Pod 종료 시퀀스:

1. kubectl delete pod / 롤링 업데이트 등으로 Pod 종료 결정
2. Pod 상태: Terminating
3. Service의 Endpoints에서 해당 Pod IP 제거 (더 이상 트래픽 안 옴)
4. preStop 훅 실행 (있다면)
5. SIGTERM을 PID 1 (컨테이너 프로세스)에 전송
6. terminationGracePeriodSeconds (기본 30초) 타이머 시작
7. 30초 내 프로세스 종료 → 정상 종료
   30초 초과 → SIGKILL → 강제 종료

Java 애플리케이션 관점:
  SIGTERM 수신 → JVM ShutdownHook 실행
  ShutdownHook: 실행 중인 요청 완료 대기 (최대 30초)
  
  주의: Step 3 (Endpoints 제거)와 Step 5 (SIGTERM)는 거의 동시에 발생
  → SIGTERM 받는 순간에도 아직 이전에 라우팅된 요청 처리 중일 수 있음
  → preStop에 sleep(5) 추가로 여유 확보 권장

Spring Boot + Kubernetes Graceful Shutdown:
  application.yml:
    server:
      shutdown: graceful       # SIGTERM 시 새 요청 거절 + 진행 중 완료 대기
    spring:
      lifecycle:
        timeout-per-shutdown-phase: 20s  # 각 단계 최대 대기시간
    
  → Spring이 ApplicationContext 종료 단계에서
    SmartLifecycle 빈들을 역순으로 stop()
    → Server 먼저 닫고 → DB Pool 닫고 → 기타 리소스 정리
```

### 5. JVM 내장 Daemon 스레드들

```
JVM이 자동으로 생성하는 daemon 스레드:

"Signal Dispatcher"
  OS 시그널 수신 및 JVM 내부 디스패치

"Finalizer"
  finalize() 메서드 실행 큐 처리
  성능 문제로 Java 18에서 deprecated

"Reference Handler"
  WeakReference, SoftReference 등 처리

"GC 스레드들" (GC 방식에 따라 다름)
  G1GC: G1 Concurrent Refinement Thread, G1 Main Marker
  ZGC:  ZGC Thread
  이들은 모두 daemon → JVM 종료 시 강제 종료

"JIT Compiler 스레드"
  C1 Compiler Thread, C2 Compiler Thread (TieredCompilation)
  daemon → JVM 종료 시 강제 종료

확인 방법:
  jstack <pid> | grep "daemon"
  또는
  ManagementFactory.getThreadMXBean().getThreadInfo(ids)
```

---

## 💻 실전 실험

### 실험 1: 데몬/non-daemon 스레드 JVM 종료 영향 관찰

```java
public class DaemonVsNonDaemon {
    public static void main(String[] args) throws InterruptedException {
        // Non-daemon 스레드 (JVM 종료를 막음)
        Thread nonDaemon = new Thread(() -> {
            System.out.println("Non-daemon 시작");
            try {
                Thread.sleep(5000);
                System.out.println("Non-daemon 완료 (5초 후)");
            } catch (InterruptedException e) {
                System.out.println("Non-daemon 인터럽트!");
            }
        }, "non-daemon-thread");
        nonDaemon.setDaemon(false);

        // Daemon 스레드 (JVM 종료 시 강제 종료)
        Thread daemon = new Thread(() -> {
            try {
                System.out.println("Daemon 시작");
                Thread.sleep(10000);
                System.out.println("Daemon 완료 (이 메시지가 출력되면 안 됨)");
            } catch (InterruptedException e) {
                System.out.println("Daemon 인터럽트! (강제 종료)");
            } finally {
                System.out.println("Daemon finally (출력 미보장)");
            }
        }, "daemon-thread");
        daemon.setDaemon(true);

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("ShutdownHook 실행!");
        }, "shutdown-hook"));

        nonDaemon.start();
        daemon.start();

        System.out.println("main 스레드 종료");
        // main 종료 → non-daemon 스레드(5초)가 끝날 때까지 JVM 대기
        // non-daemon 종료 → daemon 스레드 강제 종료 → ShutdownHook 실행
    }
}
```

### 실험 2: SIGTERM 수신 시 ShutdownHook 동작

```bash
cat > SigTermTest.java << 'EOF'
public class SigTermTest {
    public static void main(String[] args) throws InterruptedException {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("ShutdownHook 시작");
            try {
                // 진행 중인 작업 완료 시뮬레이션
                System.out.println("작업 완료 대기 중...");
                Thread.sleep(3000);
                System.out.println("작업 완료. 리소스 정리.");
            } catch (InterruptedException e) {
                System.out.println("ShutdownHook 인터럽트!");
            }
            System.out.println("ShutdownHook 완료");
        }, "shutdown-hook"));

        System.out.println("애플리케이션 시작. PID: " + ProcessHandle.current().pid());
        System.out.println("kill -TERM <PID> 로 종료 테스트");
        
        // 무한 대기 (SIGTERM을 기다림)
        Thread.sleep(Long.MAX_VALUE);
    }
}
EOF

javac SigTermTest.java
java SigTermTest &
PID=$!

sleep 2
echo "SIGTERM 전송 (PID: $PID)"
kill -TERM $PID

# 출력 예시:
# 애플리케이션 시작. PID: 12345
# ShutdownHook 시작
# 작업 완료 대기 중...
# 작업 완료. 리소스 정리.
# ShutdownHook 완료
```

### 실험 3: 여러 ShutdownHook 병렬 실행 확인

```java
public class MultiHookTest {
    public static void main(String[] args) {
        for (int i = 1; i <= 3; i++) {
            final int id = i;
            Runtime.getRuntime().addShutdownHook(new Thread(() -> {
                System.out.printf("Hook %d 시작 (스레드: %s)%n",
                    id, Thread.currentThread().getName());
                try {
                    Thread.sleep(id * 500L);
                } catch (InterruptedException e) {}
                System.out.printf("Hook %d 완료%n", id);
            }, "hook-" + i));
        }
        System.out.println("main 종료");
        // 출력: 3개 Hook이 병렬로 시작되고 순서 무보장
    }
}
```

---

## 📊 성능/비용 비교

```
종료 시나리오별 ShutdownHook 실행 여부:

종료 원인                        | ShutdownHook | 비고
───────────────────────────────┼──────────────┼────────────────────────────
main 스레드 정상 종료              | ✅           | non-daemon 모두 종료 후
System.exit(0)                 | ✅           | 즉시 Hook 실행
Ctrl+C (SIGINT)                | ✅           | Hook 완료 대기
SIGTERM                        | ✅           | Hook 완료 대기
SIGKILL (-9)                   | ❌           | OS가 즉시 종료
Runtime.halt()                 | ❌           | 강제 종료
OOM (심각)                      | 미보장         | JVM 상태에 따라 다름
StackOverflow (심각)            | 미보장         | JVM 상태에 따라 다름

Kubernetes terminationGracePeriodSeconds 기준:

기간      | 적합한 워크로드
─────────┼──────────────────────────────────
5초       | 상태 없는 경량 서비스
30초(기본)| 일반적인 웹 서비스
60초      | DB 트랜잭션, 파일 처리
120초+    | 배치 작업, 대용량 데이터 처리
```

---

## ⚖️ 트레이드오프

```
Daemon 설정 트레이드오프:

setDaemon(true):
  장점: JVM 종료를 막지 않음, 리소스 자동 회수
  단점: 진행 중인 작업 강제 중단, finally 미보장

setDaemon(false) (기본):
  장점: 작업 완료 보장, 리소스 정리 가능
  단점: 잊고 종료하지 않으면 JVM이 영원히 실행됨

ShutdownHook 설계:
  짧고 빠르게: Hook은 최대한 빨리 완료해야 함
  타임아웃 내에: 환경(Kubernetes)의 시간 제한 안에 완료
  실패해도 안전하게: Hook 실패 시 다른 Hook은 계속 실행됨
  락 최소화: 병렬 실행되므로 데드락 주의

Spring Boot Graceful Shutdown과의 관계:
  Spring은 ApplicationContext에 ShutdownHook을 자동 등록
  ApplicationContext.close() → Bean들의 @PreDestroy 실행
  → server.shutdown=graceful 시 웹 서버도 우아하게 종료
  → 직접 ShutdownHook 추가 시 Spring Hook과의 순서 주의
```

---

## 📌 핵심 정리

```
데몬 스레드와 JVM 종료 핵심:

JVM 종료 조건:
  non-daemon 스레드가 모두 종료됨 (main 포함)
  System.exit() 호출
  SIGTERM 등 OS 시그널 수신

Daemon vs Non-Daemon:
  daemon: JVM 종료 시 강제 종료 (finally 미보장)
  non-daemon: 완료될 때까지 JVM 대기
  기본값: non-daemon (main 스레드 → 상속)

ShutdownHook:
  Runtime.getRuntime().addShutdownHook(thread)
  SIGTERM, System.exit() 시 실행 (SIGKILL은 실행 안 됨)
  병렬 실행, 순서 미보장
  Hook이 블로킹되면 JVM 종료도 블로킹

Graceful Shutdown 설계:
  SIGTERM → ShutdownHook에서 Pool.shutdown() 
  awaitTermination(30, SECONDS)으로 완료 대기
  Spring Boot: server.shutdown=graceful 설정
  Kubernetes: terminationGracePeriodSeconds 맞추기

주의:
  데몬 스레드에 중요 리소스 정리 로직 두지 않기
  ShutdownHook에서 락 사용 최소화 (데드락 위험)
  SIGKILL은 어떤 것도 실행 안 됨 (컨테이너 강제 종료)
```

---

## 🤔 생각해볼 문제

**Q1.** `System.exit(0)`과 `Runtime.halt(0)`의 차이는 무엇인가? 어느 상황에서 `halt()`를 사용하는가?

<details>
<summary>해설 보기</summary>

`System.exit(0)`은 내부적으로 `Runtime.getRuntime().exit(0)`을 호출하며, **ShutdownHook을 실행하고 uninvoked finalizer를 처리한 후** JVM을 종료한다. 정상적인 종료 방식이다.

`Runtime.halt(0)`은 ShutdownHook을 실행하지 않고 **즉시** JVM 프로세스를 종료한다. 운영체제의 `_exit()` 시스템콜에 해당한다.

`halt()`를 사용하는 경우:
- ShutdownHook이 데드락에 빠졌을 때 강제 탈출
- 보안상 이유로 정리 없이 즉시 종료해야 할 때
- 테스트 환경에서 종료 속도를 높일 때

실무에서 `halt()`는 거의 사용하지 않는다. ShutdownHook이 잘 설계돼 있다면 `System.exit()`으로 충분하다. `halt()`를 남발하면 리소스 누수와 데이터 손실이 발생한다.

</details>

---

**Q2.** Thread Pool이 non-daemon 스레드로 구성된 경우, `main()` 종료 후에도 JVM이 계속 실행된다. 이것이 의도된 동작인가, 버그인가? 어떻게 올바르게 설계해야 하는가?

<details>
<summary>해설 보기</summary>

**의도된 동작이자 설계 결정**이다. Thread Pool의 Core Thread들은 기본적으로 non-daemon이므로 JVM이 종료되지 않는다. 웹 서버, 배치 서버 등에서는 main()이 서버를 초기화하고 종료되더라도 서버 스레드들이 계속 실행되는 것이 정상이다.

하지만 일회성 CLI 프로그램이나 테스트에서 이것이 버그처럼 느껴진다면, 다음 방법 중 하나를 선택한다:

```java
// 방법 1: Pool 종료 명시
executor.shutdown();
executor.awaitTermination(30, TimeUnit.SECONDS);

// 방법 2: daemon Thread Pool 사용 (리소스 정리 미보장)
ThreadFactory daemonFactory = r -> {
    Thread t = new Thread(r);
    t.setDaemon(true);
    return t;
};

// 방법 3: ShutdownHook에서 Pool 종료
Runtime.getRuntime().addShutdownHook(new Thread(executor::shutdown));
```

장기 실행 서버라면 ShutdownHook에서 Pool을 종료하는 방법 3이 가장 Graceful하다.

</details>

---

**Q3.** Kubernetes에서 `terminationGracePeriodSeconds=30`으로 설정했는데 ShutdownHook에서 30초 이상 걸리는 작업이 있다면 어떻게 되는가? 어떻게 대응해야 하는가?

<details>
<summary>해설 보기</summary>

30초 후 Kubernetes가 **SIGKILL**을 전송하고 컨테이너가 강제 종료된다. ShutdownHook은 중간에 끊기고 실행 중인 작업도 강제 종료된다.

대응 전략:

1. **`terminationGracePeriodSeconds` 늘리기**: 실제 소요 시간 + 여유분으로 설정
   ```yaml
   spec:
     terminationGracePeriodSeconds: 120
   ```

2. **ShutdownHook 내부에서 타임아웃 설정**: 남은 시간 내에 완료하도록
   ```java
   executor.shutdown();
   executor.awaitTermination(25, TimeUnit.SECONDS); // 30초보다 짧게
   executor.shutdownNow(); // 나머지 강제 취소
   ```

3. **새 요청 조기 차단**: preStop 훅으로 5초 sleep을 추가하면 endpoints 제거가 SIGTERM보다 먼저 완료되어, SIGTERM 수신 시점에는 이미 새 요청이 안 들어오는 상태

4. **작업 체크포인트**: 중간에 끊겨도 재시작 시 이어서 처리 가능하도록 설계 (멱등성, 재시도 가능 작업)

</details>

---

<div align="center">

**[⬅️ 이전: 스레드 상태 머신](./04-thread-state-machine.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — 하드웨어 메모리 모델 ➡️](../java-memory-model/01-hardware-memory-model.md)**

</div>
