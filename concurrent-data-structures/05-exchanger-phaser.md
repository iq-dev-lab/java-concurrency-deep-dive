# Exchanger와 Phaser — 데이터 교환과 다단계 동기화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Exchanger`가 두 스레드 간 데이터를 직접 교환하는 내부 메커니즘은?
- `Exchanger`를 파이프라인 패턴에서 어떻게 활용하는가?
- `Phaser`가 `CyclicBarrier`/`CountDownLatch`를 어떻게 대체하고 확장하는가?
- `arriveAndAwaitAdvance()`, `arriveAndDeregister()`, `arrive()`의 차이는?
- 동적 참가자 추가/제거가 필요한 다단계 병렬 처리에서 `Phaser`를 어떻게 사용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`Exchanger`와 `Phaser`는 표준 동기화 도구(CountDownLatch, CyclicBarrier)로 표현하기 어려운 패턴을 간결하게 구현할 수 있게 해준다. `Exchanger`는 파이프라인 스테이지 간 버퍼 교환을, `Phaser`는 단계별 병렬 계산을 자연스럽게 표현한다. 이 도구들을 이해하면 올바른 동기화 도구를 선택하는 안목이 생긴다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: CyclicBarrier로 동적 참가자를 관리하려는 시도
  // 참가자가 중간에 빠질 수 있는 경우
  CyclicBarrier barrier = new CyclicBarrier(5);  // 고정 5명
  // 3명만 도착 → 나머지 2명은 영원히 대기
  // → barrier.reset() + 재생성 필요 → 번거롭고 에러 발생 쉬움
  // → Phaser 사용

실수 2: Exchanger를 두 스레드 이상으로 사용 시도
  Exchanger<List<Item>> ex = new Exchanger<>();
  // 스레드 3개가 같은 exchanger 사용
  // → Exchanger는 정확히 2개 스레드를 위해 설계
  // → 3번째 스레드는 파트너가 없어 영원히 대기 또는 타임아웃

실수 3: Phaser.arriveAndDeregister() 후 재등록 시도
  phaser.arriveAndDeregister();  // 참가 종료
  phaser.register();             // 다시 등록 가능? → 가능하지만 주의 필요
  // 등록 해제 후 phaser가 종료됐을 수 있음
  // isTerminated() 확인 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Exchanger 올바른 사용 패턴

// 패턴 1: 더블 버퍼 파이프라인 (생산자-소비자)
Exchanger<List<Data>> exchanger = new Exchanger<>();

// 생산자 스레드
Thread producer = new Thread(() -> {
    List<Data> buffer = new ArrayList<>();
    while (running) {
        buffer.add(produce());
        if (buffer.size() >= BATCH_SIZE) {
            try {
                buffer = exchanger.exchange(buffer);  // 소비자와 교환
                buffer.clear();  // 소비자가 준 빈 버퍼 재사용
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); break;
            }
        }
    }
});

// 소비자 스레드
Thread consumer = new Thread(() -> {
    List<Data> buffer = new ArrayList<>();  // 초기 빈 버퍼
    while (running) {
        try {
            buffer = exchanger.exchange(buffer);  // 생산자와 교환
            consume(buffer);  // 가득 찬 버퍼 처리
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); break;
        }
    }
});

// Phaser 올바른 사용 패턴

// 패턴 2: 다단계 병렬 계산 (3단계 파이프라인)
Phaser phaser = new Phaser(NUM_WORKERS);

for (int i = 0; i < NUM_WORKERS; i++) {
    final int workerId = i;
    new Thread(() -> {
        // 단계 0: 데이터 로드
        loadData(workerId);
        phaser.arriveAndAwaitAdvance();  // 모두 로드 완료 대기

        // 단계 1: 변환 처리
        transformData(workerId);
        phaser.arriveAndAwaitAdvance();  // 모두 변환 완료 대기

        // 단계 2: 결과 저장
        saveResult(workerId);
        phaser.arriveAndDeregister();    // 완료 후 등록 해제
    }).start();
}

// 패턴 3: 동적 참가자 (중간에 스레드 추가/제거)
Phaser dynamicPhaser = new Phaser(1);  // 조율자 1명 먼저 등록

// 나중에 참가자 추가
dynamicPhaser.register();   // +1명
dynamicPhaser.bulkRegister(5);  // +5명

// 조율자가 단계 진행 제어
dynamicPhaser.arriveAndAwaitAdvance();  // 조율자도 이 단계에 참여
```

---

## 🔬 내부 동작 원리

### 1. Exchanger 내부 메커니즘

```java
// Exchanger<V> 핵심 구조 (단순화)
public class Exchanger<V> {
    // 슬롯: 교환 대기 중인 첫 번째 스레드의 데이터
    private volatile Node slot;

    static final class Node {
        volatile Object item;    // 이 스레드가 제공하는 데이터
        volatile Object match;   // 교환 상대가 제공한 데이터 (교환 완료 후 설정)
        volatile Thread parked;  // park된 스레드
    }

    public V exchange(V x) throws InterruptedException {
        Node item = new Node();
        item.item = x;

        // 1단계: 슬롯이 비었는가?
        if (slot == null) {
            // 첫 번째 도착자: 슬롯에 놓고 대기
            if (CAS(slot, null, item)) {
                park(item);  // 상대방을 기다림
                return (V) item.match;  // 상대방이 넣어준 데이터 반환
            }
        }

        // 2단계: 슬롯에 대기 중인 상대방이 있음
        Node partner = slot;
        if (CAS(slot, partner, null)) {
            // 교환 실행
            partner.match = x;          // 상대방에게 내 데이터 전달
            unpark(partner.parked);     // 상대방 깨우기
            return (V) partner.item;    // 상대방 데이터 반환
        }
        // 실패 시 재시도...
    }
}

실제 구현은 더 복잡:
  고경쟁 환경에서 CPU 코어 수에 맞게 슬롯 배열 사용 (Arena)
  → 여러 페어가 동시에 교환 가능
  → 경합 없을 때: 단일 슬롯 사용 (빠름)
  → 경합 시: Arena로 전환 (분산)

타임아웃 지원:
  exchange(x, timeout, unit): timeout 후 TimeoutException
  → 데드락 방지에 유용
```

### 2. Phaser 내부 — state 필드 설계

```java
// Phaser 핵심 구조 (단순화)
public class Phaser {
    // state: 단일 long 필드에 여러 카운터 pack
    private volatile long state;

    // state 비트 레이아웃:
    // [63]: terminated 플래그
    // [62..32]: phase (현재 단계 번호, 0~MAX_PHASE)
    // [31..16]: unarrived (아직 도착 안 한 참가자 수)
    // [15..0]:  parties (총 등록 참가자 수)

    // state 비트 마스크
    static final int MAX_PARTIES = 0xffff;  // 16비트 = 65535 최대 참가자
    static final int MAX_PHASE   = Integer.MAX_VALUE;
    static final int PARTIES_SHIFT = 0;
    static final int UNARRIVED_SHIFT = 16;
    static final int PHASE_SHIFT = 32;

    // arrive 시 unarrived 1 감소
    // unarrived == 0이면 다음 단계로 전진: phase++, unarrived = parties

    public int arriveAndAwaitAdvance() {
        long s = state;
        int phase = (int)(s >>> PHASE_SHIFT);
        int unarrived = (int)((s >>> UNARRIVED_SHIFT) & MAX_PARTIES);
        // CAS로 unarrived 감소
        // unarrived == 1 (마지막 도착자)?
        //   → 다음 단계로 전진: state를 원자적으로 업데이트
        //   → 대기 중인 모든 스레드 unpark
        // 아니면: park하며 다음 단계까지 대기
        return doArrive(ONE_ARRIVAL);
    }

    public int arriveAndDeregister() {
        // unarrived-- AND parties-- 동시에
        return doArrive(ONE_ARRIVAL | ONE_PARTY);
    }

    public int arrive() {
        // unarrived-- (대기 없이)
        // 다음 단계 전진 조건 달성해도 알리기만 하고 대기하지 않음
        return doArrive(ONE_ARRIVAL);
    }
}

단계 전진 시 onAdvance() 호출:
  protected boolean onAdvance(int phase, int registeredParties) {
      return registeredParties == 0;  // 참가자 없으면 종료
  }
  → 오버라이드로 단계별 커스텀 로직 삽입 가능
  → true 반환 시 Phaser 종료 (isTerminated() = true)
```

### 3. CountDownLatch vs CyclicBarrier vs Phaser 비교

```
CountDownLatch:
  단방향 카운트다운 (N → 0)
  재사용 불가
  사용 패턴: 하나의 이벤트 또는 N개 이벤트 완료 대기
  제한: 단계가 1개, 참가자 수 고정

  적합: 서비스 초기화 완료 대기, 테스트에서 여러 작업 완료 대기
  예: latch.countDown() × N → latch.await()

CyclicBarrier:
  N개 스레드가 모두 도달할 때까지 대기 (재사용 가능)
  단계 수 고정 (매번 N개 참가자 필요)
  barrierAction: 마지막 도착자가 실행할 작업 설정 가능
  제한: 참가자 수 생성 시 고정, 도중 변경 불가
       한 스레드 예외 → barrier broken → 전체 실패

  적합: 고정 인원 반복 동기화 (게임 라운드, 반복 병렬 계산)
  예: barrier.await() × 5스레드 → 모두 모이면 다음 단계

Phaser:
  동적 참가자 관리 (register/deregister)
  다단계 지원 (phase 무한 증가)
  계층적 Phaser (parent 설정) → 대규모 분산 동기화
  onAdvance() 오버라이드 → 단계 전진 시 커스텀 로직

  적합: 참가자 수가 동적으로 변하는 다단계 처리
  예: 단계별 작업자 수가 다른 경우, 일부 작업이 중간에 완료

세 도구 결정 트리:
  참가자 수 고정 + 단계 1개 + 재사용 없음 → CountDownLatch
  참가자 수 고정 + 단계 반복 → CyclicBarrier
  참가자 수 동적 + 단계 다수 → Phaser
```

### 4. Phaser 계층 구조 (Tiered Phaser)

```
대규모 동기화에서의 한계:
  단일 Phaser에 수천 개 참가자:
    모든 arrive()가 동일한 state를 CAS로 수정
    → 고경쟁 → CAS 실패율 증가

계층적 Phaser:
  루트 Phaser: 전체 상위 조율
  리프 Phaser: 소수의 참가자 (예: 8~16개)
  리프 Phaser가 단계 전진 시 → 자동으로 루트 Phaser에 arrive()

  Phaser root = new Phaser();
  // 4개 그룹, 각 그룹 8명 = 총 32명
  for (int g = 0; g < 4; g++) {
      Phaser sub = new Phaser(root, 8);  // parent = root
      for (int i = 0; i < 8; i++) {
          new Thread(() -> {
              doWork();
              sub.arriveAndDeregister();
              // 8명 모두 도착하면 sub가 root에 자동 arrive
          }).start();
      }
  }
  // root: 4개 리프가 모두 도착하면 단계 전진
  root.awaitAdvance(0);  // 모든 32명 완료 대기

이점:
  CAS 경합이 각 sub Phaser로 분산 (최대 8개씩)
  전체 동기화는 root Phaser만 담당
  → O(log N) 확장성
```

---

## 💻 실전 실험

### 실험 1: Exchanger 더블 버퍼 파이프라인

```java
import java.util.concurrent.*;
import java.util.*;

public class DoubleBufferPipeline {
    static final Exchanger<List<Integer>> exchanger = new Exchanger<>();
    static final int BATCH = 100;
    static volatile boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread producer = new Thread(() -> {
            List<Integer> buf = new ArrayList<>(BATCH);
            int val = 0;
            while (running) {
                buf.add(val++);
                if (buf.size() >= BATCH) {
                    try {
                        buf = exchanger.exchange(buf);
                        buf.clear();
                    } catch (InterruptedException e) { break; }
                }
            }
        }, "Producer");

        Thread consumer = new Thread(() -> {
            List<Integer> buf = new ArrayList<>(BATCH);
            long total = 0;
            while (running || !buf.isEmpty()) {
                try {
                    buf = exchanger.exchange(buf);
                    total += buf.stream().mapToLong(Integer::longValue).sum();
                } catch (InterruptedException e) { break; }
            }
            System.out.println("소비 총합: " + total);
        }, "Consumer");

        producer.start(); consumer.start();
        Thread.sleep(1000);
        running = false;
        producer.interrupt(); consumer.interrupt();
        producer.join(); consumer.join();
    }
}
```

### 실험 2: Phaser로 다단계 병렬 처리

```java
import java.util.concurrent.*;

public class PhasedParallelPipeline {
    static final int WORKERS = 4;

    public static void main(String[] args) throws InterruptedException {
        // 3단계 처리 파이프라인
        Phaser phaser = new Phaser(WORKERS) {
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.printf("[Phase %d 완료] 참가자=%d%n", phase, registeredParties);
                return phase >= 2 || registeredParties == 0;  // 3단계(0,1,2) 후 종료
            }
        };

        for (int w = 0; w < WORKERS; w++) {
            final int id = w;
            new Thread(() -> {
                // 단계 0: 데이터 로드
                System.out.printf("Worker %d: 단계 0 (로드) 시작%n", id);
                try { Thread.sleep(100 + id * 20); } catch (InterruptedException e) {}
                phaser.arriveAndAwaitAdvance();  // 모두 완료 대기

                // 단계 1: 처리
                System.out.printf("Worker %d: 단계 1 (처리) 시작%n", id);
                try { Thread.sleep(150 + id * 10); } catch (InterruptedException e) {}
                phaser.arriveAndAwaitAdvance();

                // 단계 2: 저장 (일부 워커는 일찍 완료)
                System.out.printf("Worker %d: 단계 2 (저장) 시작%n", id);
                try { Thread.sleep(50); } catch (InterruptedException e) {}
                phaser.arriveAndDeregister();  // 완료 후 등록 해제
                System.out.printf("Worker %d: 완료%n", id);
            }, "Worker-" + w).start();
        }

        while (!phaser.isTerminated()) {
            System.out.printf("[Main] 현재 phase=%d, 미도착=%d%n",
                phaser.getPhase(), phaser.getUnarrivedParties());
            Thread.sleep(200);
        }
        System.out.println("파이프라인 완료");
    }
}
```

### 실험 3: 동적 참가자 Phaser

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class DynamicPhaserDemo {
    static Phaser phaser = new Phaser(1);  // 조율자만 등록
    static AtomicInteger resultSum = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        System.out.println("초기 참가자: " + phaser.getRegisteredParties());

        // 동적으로 워커 추가
        for (int i = 0; i < 5; i++) {
            phaser.register();  // 워커 등록 (+1)
            final int id = i;
            new Thread(() -> {
                int result = compute(id);
                resultSum.addAndGet(result);
                phaser.arriveAndDeregister();  // 완료 후 해제
                System.out.printf("Worker %d 완료 (result=%d)%n", id, result);
            }).start();
        }

        System.out.println("워커 추가 후 참가자: " + phaser.getRegisteredParties());

        // 조율자가 모두 완료될 때까지 대기
        phaser.arriveAndAwaitAdvance();  // 조율자도 도착
        System.out.println("모든 워커 완료. 합산 결과: " + resultSum.get());

        // Phaser 종료
        phaser.arriveAndDeregister();
        System.out.println("종료: " + phaser.isTerminated());
    }

    static int compute(int id) {
        try { Thread.sleep(100 * (id + 1)); } catch (InterruptedException e) {}
        return id * id;
    }
}
```

---

## 📊 성능/비용 비교

```
동기화 도구 선택 비교:

특성                   | CountDownLatch | CyclicBarrier | Phaser
──────────────────────┼───────────────┼───────────────┼─────────────────────
참가자 수              | 고정 (생성 시) | 고정 (생성 시) | 동적 (register/deregister)
재사용성               | ❌ (1회)       | ✅ (반복)      | ✅ (무한 단계)
단계 수                | 1              | 반복 가능     | 무한 (phase 카운터)
단계별 커스텀 로직      | ❌            | ✅ (barrierAction)| ✅ (onAdvance 오버라이드)
일부 참가자 예외 시     | 대기 유지      | Broken 상태   | 해당 참가자만 제거 가능
대규모 참가자 확장성    | 보통           | 보통          | ✅ (계층적 Phaser)
내부 구현              | AQS 공유락    | ReentrantLock + Condition | CAS on long state

Exchanger 특성:
  정확히 2개 스레드만 지원
  고경쟁 시 Arena 확장 (여러 슬롯)
  타임아웃 지원
  비용: park/unpark 1회 + CAS 1~2회

Phaser 비용:
  arrive(): long state에 CAS 1회
  arriveAndAwaitAdvance(): CAS + park/unpark
  단계 전진 시: 모든 대기 스레드 unpark
```

---

## ⚖️ 트레이드오프

```
Exchanger 선택 기준:

적합:
  정확히 2개 스레드 간 데이터 교환
  더블 버퍼 패턴 (생산-소비 버퍼 교환)
  파이프라인 스테이지 간 배치 전달

부적합:
  3개 이상 스레드 → Exchanger 설계 의도 벗어남
  단순 데이터 전달 → SynchronousQueue가 더 적합
  비동기 통신 → CompletableFuture가 더 적합

Phaser 선택 기준:

적합:
  다단계 병렬 계산 (단계별 동기화 필요)
  참가자가 중간에 추가/제거되는 경우
  단계 완료 시 추가 로직 필요 (onAdvance)
  대규모 분산 동기화 (계층적 Phaser)

CyclicBarrier 대신 Phaser:
  참가자 수 동적 변경 필요
  단계 수가 미리 알 수 없는 경우
  일부 참가자 실패 허용 (arriveAndDeregister)

CountDownLatch 대신 Phaser:
  완료 후 재사용 필요
  단계가 여러 개
```

---

## 📌 핵심 정리

```
Exchanger 핵심:
  두 스레드 간 데이터 교환 (정확히 2개)
  단일 슬롯 CAS → 고경쟁 시 Arena 확장
  더블 버퍼 파이프라인 패턴에 최적

Phaser 핵심:
  state 필드: phase + unarrived + parties를 long에 pack
  arrive(): CAS로 unarrived 1 감소
  unarrived = 0 시 다음 단계 전진 + 대기자 전부 깨움

3가지 도착 메서드:
  arriveAndAwaitAdvance(): 도착 + 다음 단계까지 블로킹
  arriveAndDeregister():   도착 + 등록 해제 (이후 참여 없음)
  arrive():               도착만 (대기 없음)

도구 선택:
  이벤트 완료 대기 1회: CountDownLatch
  고정 참가자 반복 동기화: CyclicBarrier
  동적 참가자 다단계: Phaser
  두 스레드 데이터 교환: Exchanger
  두 스레드 직접 전달 (큐): SynchronousQueue
```

---

## 🤔 생각해볼 문제

**Q1.** `Phaser.arrive()`와 `arriveAndAwaitAdvance()`의 차이는 무엇인가? `arrive()`는 언제 유용한가?

<details>
<summary>해설 보기</summary>

`arrive()`는 현재 단계에 도착했음을 알리지만 다음 단계가 시작될 때까지 **기다리지 않는다**. 반면 `arriveAndAwaitAdvance()`는 모든 참가자가 도착할 때까지 블로킹한다.

`arrive()`가 유용한 경우:
1. **비동기 완료 알림**: 작업을 완료했지만 다음 단계를 기다릴 필요가 없는 스레드
2. **메인 스레드 폴링**: 마스터 스레드가 워커들을 시작시키고 주기적으로 진행 상황을 확인
3. **비블로킹 파이프라인**: 각 스테이지가 독립적으로 진행되고, 전체 완료만 최종 확인

```java
// arrive() 패턴: 완료 알리고 계속 다른 작업
phaser.register();
doAsyncWork(result -> {
    processResult(result);
    phaser.arrive();  // 알리고 대기 안 함
});
doOtherWork();  // 다른 작업 계속
phaser.awaitAdvance(0);  // 나중에 단계 완료 확인
```

</details>

---

**Q2.** `Exchanger.exchange(x, timeout, unit)`에서 타임아웃이 발생하면 교환 상대방은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

타임아웃이 발생하면 `TimeoutException`을 던지고 해당 스레드는 슬롯에서 자신의 항목을 회수하려 시도한다.

구체적으로:
1. Thread A가 슬롯에 항목을 놓고 대기
2. 타임아웃 → Thread A가 CAS로 슬롯에서 자신의 항목 회수 시도
3. 회수 성공 → `TimeoutException` throw, Thread A는 정상적으로 탈출
4. 회수 실패 → Thread B가 이미 슬롯을 가져감 (교환 완료 직전)
   - Thread A는 교환이 완료됐으므로 `TimeoutException` 없이 교환 결과 반환

따라서 타임아웃 직전에 교환이 성사되면 Thread A는 결과를 받는다. 타임아웃이 정확히 "교환 미성사" 또는 "교환 성사" 중 하나로 귀결된다.

Thread B 관점에서: Thread A가 타임아웃으로 항목을 회수했다면 Thread B는 슬롯이 비었음을 발견하고 다시 대기한다. Thread B에게는 Thread A의 타임아웃이 투명하게 처리된다.

</details>

---

**Q3.** `Phaser`의 `onAdvance(phase, registeredParties)`에서 `true`를 반환하면 Phaser가 종료된다. 이후 등록된 참가자들의 `arriveAndAwaitAdvance()`는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

`Phaser`가 종료된 후에는:
- `isTerminated()` = true
- `arrive()`, `arriveAndAwaitAdvance()`, `register()` 등이 즉시 음수 phase 값을 반환
- 종료된 Phaser에서 `arriveAndAwaitAdvance()`는 블로킹하지 않고 즉시 반환

따라서 종료 후에 도착하는 스레드들은 무한 대기에 빠지지 않는다.

`onAdvance()` 오버라이드 패턴:
```java
Phaser phaser = new Phaser(N) {
    protected boolean onAdvance(int phase, int registeredParties) {
        if (phase == 0) {
            System.out.println("1단계 완료, 중간 처리...");
        }
        // 3단계(0,1,2)까지만 실행
        return phase >= 2 || registeredParties == 0;
    }
};
```

실수 주의: `registeredParties == 0`을 기본 종료 조건으로 반드시 포함해야 한다. 그렇지 않으면 모든 참가자가 `arriveAndDeregister()`로 떠나도 Phaser가 종료되지 않는다 (`return false`인 경우).

</details>

---

<div align="center">

**[⬅️ 이전: ConcurrentSkipListMap](./04-concurrent-skip-list-map.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — Virtual Thread와 Continuation ➡️](../virtual-threads/01-virtual-thread-continuation.md)**

</div>
