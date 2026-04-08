# 데드락 발생과 분석 — jstack과 Thread Dump

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 데드락 4가지 조건(상호 배제, 점유 대기, 비선점, 순환 대기)은 무엇인가?
- `jstack`, `jcmd Thread.print`로 Thread Dump를 수집하는 방법은?
- Thread Dump에서 "Found one Java-level deadlock" 섹션을 어떻게 읽는가?
- 락 소유자와 대기자를 추적해 데드락 사이클을 찾는 방법은?
- 데드락을 예방하는 코드 패턴은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

데드락은 간헐적으로 발생하여 재현이 어렵고, 발생 시 서비스가 완전히 멈춰버린다. Thread Dump를 읽는 능력은 온콜 엔지니어의 필수 스킬이다. 데드락을 코드에서 예방하는 패턴과 발생 후 신속히 진단하는 방법 모두를 알아야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 락 획득 순서를 일관되게 유지하지 않음
  // Thread A
  synchronized (lock1) { synchronized (lock2) { doWork(); } }
  // Thread B
  synchronized (lock2) { synchronized (lock1) { doWork(); } }
  // → 순환 대기 → 데드락

실수 2: Thread Dump를 읽는 방법 모름
  서비스가 멈춤 → CPU 0% → 응답 없음
  "모르겠다, 서버 재시작"
  → 근본 원인 파악 없이 반복 발생
  → Thread Dump에 데드락 정보 그대로 있음

실수 3: tryLock 사용 안 함
  // 무기한 대기 → 데드락 가능
  lock1.lock();
  lock2.lock();  // lock2를 기다리는 동안 다른 스레드가 lock1 대기

  // 안전한 패턴
  if (lock1.tryLock(1, TimeUnit.SECONDS)) {
      try {
          if (lock2.tryLock(1, TimeUnit.SECONDS)) {
              try { doWork(); } finally { lock2.unlock(); }
          }
      } finally { lock1.unlock(); }
  }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 데드락 예방 패턴

// 패턴 1: 락 획득 순서 통일 (시스템 전체에서 일관)
// 모든 곳에서 lock1 → lock2 순서로만 획득
class BankAccount {
    private final int id;
    private final ReentrantLock lock = new ReentrantLock();

    static void transfer(BankAccount from, BankAccount to, int amount) {
        // id 기준으로 항상 낮은 id 먼저 락 획득 → 순환 대기 방지
        BankAccount first  = from.id < to.id ? from : to;
        BankAccount second = from.id < to.id ? to : from;
        first.lock.lock();
        try {
            second.lock.lock();
            try {
                from.balance -= amount;
                to.balance += amount;
            } finally { second.lock.unlock(); }
        } finally { first.lock.unlock(); }
    }
}

// 패턴 2: tryLock + 타임아웃 (데드락 자가 회복)
boolean tryTransfer(BankAccount from, BankAccount to, int amount) {
    if (from.lock.tryLock()) {
        try {
            if (to.lock.tryLock()) {
                try {
                    from.balance -= amount;
                    to.balance += amount;
                    return true;
                } finally { to.lock.unlock(); }
            }
        } finally { from.lock.unlock(); }
    }
    return false;  // 실패 시 재시도 (백오프 추가)
}

// 패턴 3: 단일 글로벌 락 (복잡성 낮출 때)
// 성능보다 안전성 우선 시 단일 락으로 모든 작업 직렬화
synchronized (globalLock) {
    transfer(from, to, amount);
}
```

---

## 🔬 내부 동작 원리

### 1. 데드락 4가지 필요 조건

```
데드락이 발생하기 위한 4가지 조건 (모두 동시에 성립해야 함):

① 상호 배제 (Mutual Exclusion):
  자원(락)은 하나의 스레드만 독점 사용
  → synchronized, ReentrantLock 모두 해당

② 점유 대기 (Hold and Wait):
  스레드가 이미 자원을 보유하면서 추가 자원을 기다림
  Thread A: lock1 보유 + lock2 대기
  Thread B: lock2 보유 + lock1 대기

③ 비선점 (No Preemption):
  자원을 강제로 빼앗을 수 없음
  락을 보유한 스레드가 스스로 해제해야 함
  → JVM은 데드락 감지 후 자동 해제 없음!

④ 순환 대기 (Circular Wait):
  스레드들이 자원을 원형으로 대기
  T1 → lock2(T2가 보유) → T2 → lock1(T1이 보유) → T1

데드락 예방 전략:
  ① 비해당: 자원의 공유 허용 (읽기 전용화)
  ② 비해당: 필요한 모든 락을 한번에 획득 (Lock Ordering)
  ③ 해당 가능: tryLock으로 포기 후 재시도
  ④ 가장 실용적: 락 획득 순서 통일
```

### 2. Thread Dump 수집 방법

```bash
# 방법 1: jstack (별도 프로세스)
jstack <pid> > thread_dump.txt
jstack -l <pid> > thread_dump_with_locks.txt  # 락 정보 포함

# 방법 2: jcmd (더 안전, 권장)
jcmd <pid> Thread.print > thread_dump.txt
jcmd <pid> Thread.print -l > thread_dump_with_locks.txt

# 방법 3: kill -3 (SIGQUIT) - Unix/Linux
kill -3 <pid>
# JVM이 stderr에 Thread Dump 출력

# 방법 4: jvisualvm / jconsole (GUI)
# 원격 JMX 연결 → Threads 탭 → Thread Dump

# 방법 5: Spring Boot Actuator
curl http://localhost:8080/actuator/threaddump

# 방법 6: jmap
# (데드락 감지에는 jstack이 더 적합)

# 방법 7: JFR
jcmd <pid> JFR.start duration=30s filename=dump.jfr
# jfr print --events jdk.JavaMonitorDeadlock dump.jfr
```

### 3. Thread Dump 읽는 방법

```
Thread Dump 구조 예시 (데드락 발생 시):

=======================================================
# 데드락 감지 섹션 (JVM이 자동 분석)
Found one Java-level deadlock:
=============================

"Thread-B":
  waiting to lock monitor 0x00007f1234567890 (object 0x000000070fff1234, a java.lang.Object),
  which is held by "Thread-A"

"Thread-A":
  waiting to lock monitor 0x00007f1234567891 (object 0x000000070fff5678, a java.lang.Object),
  which is held by "Thread-B"

Java stack information for the threads listed above:
===================================================
"Thread-B":
        at com.example.DeadlockDemo.methodB(DeadlockDemo.java:30)
        - waiting to lock <0x000000070fff1234> (a java.lang.Object)
        - locked <0x000000070fff5678> (a java.lang.Object)
        at com.example.DeadlockDemo.run(DeadlockDemo.java:20)

"Thread-A":
        at com.example.DeadlockDemo.methodA(DeadlockDemo.java:15)
        - waiting to lock <0x000000070fff5678> (a java.lang.Object)
        - locked <0x000000070fff1234> (a java.lang.Object)
        at com.example.DeadlockDemo.run(DeadlockDemo.java:10)

Found 1 deadlock.
=======================================================

읽는 방법:
1. "Found N Java-level deadlock" 검색 → 데드락 존재 확인
2. 각 스레드가 어떤 락을 보유(locked)하고 대기(waiting to lock)하는지 확인
3. 순환 사이클 추적:
   Thread-B → waiting for 0x...1234 → held by Thread-A
   Thread-A → waiting for 0x...5678 → held by Thread-B
   → 순환 확인!
4. 스택 트레이스에서 실제 코드 위치 확인
   → DeadlockDemo.java:30, DeadlockDemo.java:15
5. 해당 코드에서 락 획득 순서 분석 → 수정
```

### 4. ThreadMXBean으로 프로그래밍 방식 감지

```java
import java.lang.management.*;

public class DeadlockDetector {
    private final ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();

    // 주기적으로 데드락 체크
    public void startMonitoring() {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
        scheduler.scheduleAtFixedRate(this::checkDeadlock, 10, 10, TimeUnit.SECONDS);
    }

    public void checkDeadlock() {
        long[] deadlockedIds = mxBean.findDeadlockedThreads();
        if (deadlockedIds == null) return;

        ThreadInfo[] infos = mxBean.getThreadInfo(deadlockedIds, true, true);
        StringBuilder sb = new StringBuilder("데드락 감지!\n");
        for (ThreadInfo info : infos) {
            sb.append("스레드: ").append(info.getThreadName())
              .append(", 상태: ").append(info.getThreadState())
              .append("\n");
            // 대기 중인 락
            LockInfo lockInfo = info.getLockInfo();
            if (lockInfo != null) {
                sb.append("  대기 락: ").append(lockInfo.getClassName())
                  .append("@").append(Integer.toHexString(lockInfo.getIdentityHashCode()))
                  .append("\n");
            }
            // 보유 중인 락들
            for (MonitorInfo monitor : info.getLockedMonitors()) {
                sb.append("  보유 락: ").append(monitor.getClassName())
                  .append("@").append(Integer.toHexString(monitor.getIdentityHashCode()))
                  .append(" at ").append(monitor.getLockedStackFrame()).append("\n");
            }
        }
        log.error(sb.toString());
        // 알림 발송, 힙 덤프 생성 등
    }
}
```

---

## 💻 실전 실험

### 실험 1: 데드락 의도적 재현

```java
public class DeadlockDemo {
    static final Object LOCK_A = new Object();
    static final Object LOCK_B = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> {
            synchronized (LOCK_A) {
                System.out.println("Thread-A: LOCK_A 획득");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("Thread-A: LOCK_B 대기 중...");
                synchronized (LOCK_B) {
                    System.out.println("Thread-A: LOCK_B 획득 (데드락 없이)");
                }
            }
        }, "Thread-A");

        Thread threadB = new Thread(() -> {
            synchronized (LOCK_B) {
                System.out.println("Thread-B: LOCK_B 획득");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("Thread-B: LOCK_A 대기 중...");
                synchronized (LOCK_A) {
                    System.out.println("Thread-B: LOCK_A 획득 (데드락 없이)");
                }
            }
        }, "Thread-B");

        threadA.start();
        threadB.start();

        // 데드락 감지
        Thread.sleep(500);
        ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
        long[] deadlocked = mxBean.findDeadlockedThreads();
        if (deadlocked != null) {
            System.out.println("데드락 감지! 스레드 수: " + deadlocked.length);
            // jstack으로 Thread Dump 확인
        }

        threadA.join(3000);
        if (threadA.isAlive()) {
            System.out.println("Thread-A 여전히 대기 중 → 데드락 확인");
        }
    }
}
```

### 실험 2: 락 순서 고정으로 데드락 예방

```java
public class DeadlockPrevention {
    static final Object LOCK_A = new Object();
    static final Object LOCK_B = new Object();

    // 해결: 항상 같은 순서로 락 획득
    static void safeCriticalSection() {
        Object first  = System.identityHashCode(LOCK_A) < System.identityHashCode(LOCK_B) ?
                        LOCK_A : LOCK_B;
        Object second = first == LOCK_A ? LOCK_B : LOCK_A;

        synchronized (first) {
            synchronized (second) {
                System.out.println(Thread.currentThread().getName() + ": 임계 구역");
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(DeadlockPrevention::safeCriticalSection,
                "Thread-" + i);
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("완료 (데드락 없음)");
    }
}
```

---

## 📊 성능/비용 비교

```
데드락 예방 전략별 특성:

전략               | 오버헤드 | 복잡성 | 보장 수준
─────────────────┼─────────┼────────┼───────────────────────
락 순서 통일      | 없음     | 중간   | 순환 대기 제거 → 데드락 없음
tryLock + 백오프  | 낮음     | 높음   | 라이브락 가능 (백오프 필요)
단일 글로벌 락    | 중간     | 낮음   | 완전 예방, 처리량 저하
락 없는 설계      | 없음     | 높음   | 완전 예방 (CAS 기반)
타임아웃 tryLock  | 낮음     | 중간   | 데드락 회복 가능

ThreadMXBean 감지 비용:
  findDeadlockedThreads(): ~수 μs~수 ms (스레드 수에 비례)
  주기적 호출(10초마다): 무시 가능한 오버헤드
```

---

## ⚖️ 트레이드오프

```
데드락 예방 vs 감지 vs 허용:

예방 (Prevention):
  장점: 데드락 자체가 발생하지 않음
  단점: 코드 복잡성 증가, 성능 제약 가능
  방법: 락 순서 통일, tryLock 타임아웃

감지 (Detection):
  장점: 자유로운 코드 작성 가능
  단점: 감지 후 복구 로직 필요 (스레드 종료 등)
  방법: ThreadMXBean.findDeadlockedThreads(), JFR

허용 (Ostrich Algorithm):
  장점: 최대 성능
  단점: 데드락 발생 시 수동 재시작 필요
  실무: 데드락이 매우 드물고 재시작이 허용되는 경우

실무 권장:
  1순위: 락 순서 통일 (복잡성 낮음, 안전)
  2순위: tryLock + 타임아웃 (유연성)
  3순위: ThreadMXBean 감지 + 알림 (방어선)
  운영: Thread Dump 즉시 분석 능력 갖추기
```

---

## 📌 핵심 정리

```
데드락 핵심:

4가지 조건:
  상호 배제 + 점유 대기 + 비선점 + 순환 대기 → 데드락

예방:
  락 순서 통일 (순환 대기 제거) → 가장 실용적
  tryLock 타임아웃 (비선점 극복)
  필요한 락 한번에 획득 (점유 대기 제거, 어려움)

Thread Dump 수집:
  jstack <pid>, jcmd <pid> Thread.print
  kill -3 <pid> (Unix)

Thread Dump 읽기:
  "Found N Java-level deadlock" 섹션 찾기
  waiting to lock (대기) vs locked (보유) 추적
  순환 사이클 확인 → 코드 위치 파악 → 수정

프로그래밍 감지:
  ThreadMXBean.findDeadlockedThreads()
  주기적 호출 + 알림으로 자동화

수정 방법:
  락 획득 순서 통일
  synchronized → tryLock으로 교체
  락 범위 최소화
  Lock Striping으로 락 분리
```

---

## 🤔 생각해볼 문제

**Q1.** `ReentrantLock.tryLock()`으로 데드락을 회피할 때, 두 스레드가 계속 동시에 tryLock 실패하면 라이브락(Livelock)이 되는가?

<details>
<summary>해설 보기</summary>

이론적으로 라이브락이 가능하다. 두 스레드가 동시에 `tryLock()`을 시도하고 동시에 실패하면, 랜덤 백오프 없이 즉시 재시도하는 경우 계속 동시에 시도→실패→재시도 패턴이 반복될 수 있다.

해결: **지수 백오프(Exponential Backoff)**:
```java
long backoff = 1;
while (!acquired) {
    if (lock1.tryLock()) {
        try {
            if (lock2.tryLock()) {
                try { doWork(); return; }
                finally { lock2.unlock(); }
            }
        } finally { lock1.unlock(); }
    }
    Thread.sleep(backoff + (long)(Math.random() * backoff));
    backoff = Math.min(backoff * 2, 1000);  // 최대 1초
}
```

랜덤 요소가 두 스레드의 재시도 타이밍을 서로 다르게 만들어 라이브락을 방지한다.

실제로 CPU 스케줄러의 비결정성 때문에 순수 라이브락은 극히 드물다. 하지만 공정한 설계를 위해 백오프는 권장된다.

</details>

---

**Q2.** `jstack`으로 Thread Dump를 뜰 때 JVM이 잠시 멈추는가? 운영 환경에서 안전한가?

<details>
<summary>해설 보기</summary>

`jstack`은 대상 JVM에 SIGQUIT 신호를 보내거나 JVM의 Attach API를 사용한다. 스레드 정보를 수집하는 동안 짧은 STW(Stop-The-World)가 발생할 수 있다. 대형 애플리케이션(수천 스레드)에서는 수백 ms에서 수 초의 지연이 생길 수 있다.

운영 환경 권장:
1. **`jcmd Thread.print`**: `jstack`보다 안전. JVM과 동일 프로세스에서 실행 가능.
2. **JFR**: 운영 환경에서 가장 안전. 오버헤드 ~1~2%, STW 없음.
3. **`kill -3`**: SIGQUIT 사용, 추가 STW 없이 stderr에 덤프.
4. **Spring Actuator `/actuator/threaddump`**: HTTP 요청으로 안전하게 수집.

`jstack`은 `sudo`나 OS 권한이 필요한 경우도 있어 컨테이너 환경에서 제약이 있다. 운영 환경에서는 JFR을 항상 켜두는 것이 권장된다.

</details>

---

**Q3.** 데이터베이스 수준 데드락(DB deadlock)과 Java 스레드 데드락(JVM deadlock)은 어떻게 구별하는가?

<details>
<summary>해설 보기</summary>

두 가지는 발생 위치와 감지 방법이 다르다.

**Java 스레드 데드락**:
- `synchronized`, `ReentrantLock` 등 JVM 락에서 발생
- `jstack`의 "Found N Java-level deadlock" 섹션에서 감지
- JVM이 감지하지만 자동으로 해제하지 않음 (프로세스 멈춤)
- `ThreadMXBean.findDeadlockedThreads()`로 프로그래밍 감지

**DB 데드락**:
- 두 트랜잭션이 서로의 행(row)이나 테이블 락을 기다릴 때 발생
- DB 서버가 감지하고 하나의 트랜잭션을 강제 롤백
- 애플리케이션은 `SQLException: Deadlock found when trying to get lock`을 받음
- Java 스레드는 `BLOCKED`가 아닌 `RUNNABLE` 또는 `WAITING` 상태 (DB 응답 대기)

구별 방법:
- `jstack`에서 스레드가 `WAITING on condition`이고 스택에 JDBC 호출이 있으면 DB 데드락 의심
- "Found Java-level deadlock"이 없으면 Java 데드락 아님 → DB, 외부 서비스 등 확인
- DB 로그(InnoDB: `SHOW ENGINE INNODB STATUS`)에서 deadlock 섹션 확인

두 가지가 동시에 발생하는 경우도 있다: Java 락이 DB 연결을 보유한 채로 대기하면 두 레벨에서 데드락이 중첩될 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: Ch6 Spring Boot + Virtual Thread](../virtual-threads/06-spring-boot-virtual-thread.md)** | **[홈으로 🏠](../README.md)** | **[다음: 라이브락과 기아 ➡️](./02-livelock-starvation.md)**

</div>
