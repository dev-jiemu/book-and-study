# Go vs Java 동시성 처리 비교

> 정리일: 2026-08-23

## 개요

Go와 Java의 언어 차이를 비교하면서, 대표적인 차이점으로 동시성(concurrency) 처리 방식을
살펴봤다. Go는 언어 설계 단계부터 goroutine/channel을 1급 시민으로 넣은 반면, Java는
기존 무거운 OS 스레드 모델 위에 가상 스레드(Virtual Threads, Project Loom)를 최근에
얹은 구조라 접근 방식과 그로 인한 이슈의 성격이 다르다.

## 핵심 내용

### 기본 모델

- **Go**: goroutine + channel이 언어 문법에 내장. GMP(Goroutine-Machine-Processor)
  스케줄러가 work-stealing 방식으로 OS 스레드에 goroutine을 분배. goroutine은 2KB
  스택으로 시작해서 필요시 확장.
- **Java**: 전통적으로 OS 스레드를 그대로 사용(스레드 하나당 약 1MB)해서 대규모 동시
  요청 처리에 한계가 있었음. Java 21에서 가상 스레드(JEP 444)가 도입되어 ForkJoinPool
  기반 캐리어 스레드 위에서 경량 스레드를 흉내냄. 다만 25년치 플랫폼 스레드 가정 위에
  얹은 구조라 Go에는 없는 종류의 함정이 존재.

### 대표적인 동시성 이슈

- **Pinning (Java 21~23 한정)**: 가상 스레드가 `synchronized` 블록 안에서 블로킹
  I/O를 만나면 캐리어 스레드에 고정(pin)되어 가상 스레드의 장점이 사라짐. JDK 24의
  JEP 491에서 완전히 해결됨. 그 전까지는 `synchronized` 대신 `ReentrantLock`을 쓰는
  게 우회책. Go는 스케줄러 설계상 이런 pinning 문제 자체가 없음.
- **구조화된 동시성(structured concurrency)**: Go는 언어 차원의 지원이 없어서
  `WaitGroup`, `context.Context`, `errgroup` 같은 라이브러리로 직접 생명주기를
  관리해야 함 — 소유권을 명시하지 않으면 goroutine leak이 흔함. Java는
  `StructuredTaskScope` API로 스코프 종료 시 하위 작업을 자동 취소하며, Java 25에서
  JEP 506으로 정식 기능이 됨.
- **메모리·성능**: I/O 바운드 시나리오에서 Go가 Java보다 대략 40~60% 처리량이 높고,
  대규모 동시성에서 전체 메모리 사용량도 4~5배 적다는 벤치마크가 있음(다만 이런
  비교는 보통 Java의 JIT 워밍업 전 상태를 측정해 다소 불리하게 나온다는 지적도 있음).
- **통신 방식**: Go의 channel + `select`는 언어 문법에 내장되어 팬인/타임아웃 패턴을
  우아하게 표현. Java는 executor/future 기반이라 상대적으로 저수준이고 명시적.

### 두 언어에서 공통적으로 반복되는 실패 패턴

언어와 무관하게 실제 프로덕션 장애에서 반복되는 패턴들:

- **Fire-and-forget**: 동시 작업을 누가 소유하고 누가 멈출지 정하지 않고 띄우는
  패턴. Go의 방치된 goroutine, Java의 소유자 없는 executor 제출 모두 리소스 누수와
  에러 처리 누락으로 이어짐.
- **백프레셔 부재**: 요청마다 무제한으로 goroutine/task를 만들면 트래픽 급증 시
  시스템이 무너짐.
- **취소(cancellation) 무시**: Go의 `context.Context`는 전달만 되고 체크 안 되는
  경우가 많고, Java의 `InterruptedException`도 자주 무시됨.
- **공유 상태 혼란**: Go는 channel과 동기화를 혼용하다 꼬이고, Java는 `synchronized`
  과용으로 데드락에 빠지는 경우가 흔함.

결론적으로 Java는 "레거시 위에 경량 동시성을 얹으면서 생기는 호환성 이슈(pinning 등)"가
특징이고, Go는 "언어가 동시성을 쉽게 만들어주는 대신 구조화·소유권 관리를 개발자가
직접 챙겨야 해서 생기는 누수·백프레셔 문제"가 특징. 결국 "누가 이 작업을 소유하고,
언제 취소하고, 얼마나 많이 동시에 띄울지"를 설계 단계에서 명확히 하지 않으면 두
언어 모두 비슷한 장애로 귀결된다.

## 예시 / 코드

### 1. 기본: 동시 작업 실행하고 결과 모으기

Go — goroutine + channel:
```go
func fetchAll(urls []string) []string {
    results := make(chan string, len(urls))

    for _, url := range urls {
        go func(u string) {
            results <- fetch(u) // 결과를 channel로 전달
        }(url)
    }

    out := make([]string, 0, len(urls))
    for i := 0; i < len(urls); i++ {
        out = append(out, <-results)
    }
    return out
}
```

Java — 가상 스레드(Java 21+):
```java
List<String> fetchAll(List<String> urls) throws InterruptedException {
    List<Future<String>> futures = new ArrayList<>();

    try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
        for (String url : urls) {
            futures.add(executor.submit(() -> fetch(url)));
        }
    } // executor.close()가 자동으로 모든 작업 완료를 기다림

    return futures.stream()
        .map(f -> { try { return f.get(); } catch (Exception e) { throw new RuntimeException(e); } })
        .toList();
}
```

Go는 `go` 키워드 하나로 동시 실행이 언어 문법이고 결과 취합도 channel로 자연스러운
반면, Java는 executor라는 "객체"를 통해 스레드를 다룬다. Go가 더 가볍고 직관적으로
보이지만, 그만큼 종료 시점을 개발자가 직접 관리해야 함(channel 개수를 세는 루프가
그 역할).

### 2. Pinning 이슈 (Java 21~23에서만 발생, JDK 24에서 해결)

```java
// 문제가 되는 코드 (Java 21~23)
synchronized void badMethod() {
    // 가상 스레드가 이 블록 안에서 블로킹 I/O를 하면
    // 캐리어 스레드에 "pin" 되어버려서 다른 가상 스레드가 그 캐리어를 못 씀
    someBlockingIO();
}

// 우회책
private final ReentrantLock lock = new ReentrantLock();
void goodMethod() {
    lock.lock();
    try {
        someBlockingIO(); // pinning 없음
    } finally {
        lock.unlock();
    }
}
```

Go는 `sync.Mutex`에 이런 "고정" 문제 자체가 없음 — goroutine은 스케줄러가 언제든
다른 OS 스레드로 옮길 수 있게 설계됨. Java가 "레거시 위에 얹은 모델"이라 생기는
특유의 함정.

### 3. 구조화된 동시성 (여러 작업 중 하나가 실패하면 나머지도 취소)

Go — `errgroup` 라이브러리로:
```go
func fetchAllOrFail(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            r, err := fetchWithContext(ctx, url) // ctx 취소를 체크해야 함
            if err != nil {
                return err // 하나 실패하면 다른 goroutine들의 ctx도 취소됨
            }
            results[i] = r
            return nil
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

Java — `StructuredTaskScope`(Java 25 정식):
```java
List<String> fetchAllOrFail(List<String> urls) throws Exception {
    try (var scope = StructuredTaskScope.open(Joiner.<String>allSuccessfulOrThrow())) {
        List<Subtask<String>> tasks = urls.stream()
            .map(url -> scope.fork(() -> fetch(url)))
            .toList();

        scope.join(); // 하나 실패하면 나머지 자동 취소
        return tasks.stream().map(Subtask::get).toList();
    }
}
```

Go는 `errgroup`이 표준 라이브러리가 아니라 별도 패키지고, 취소 전파도 `ctx`를 각
함수에 손으로 넘겨서 체크해야 함(안 체크하면 무시됨 — "cancellation negligence"
문제). Java의 `StructuredTaskScope`는 스코프 종료 시 런타임이 자동으로 하위 작업을
취소해줘서 개발자가 잊어버릴 여지가 적음.

### 4. 누수(leak) 나는 패턴 예시

```go
// Go: 흔한 goroutine leak — 아무도 이 goroutine을 기다리거나 취소하지 않음
func handler(w http.ResponseWriter, r *http.Request) {
    go doSomethingSlow() // fire-and-forget, 서버 종료돼도 안 끝날 수 있음
    w.Write([]byte("OK"))
}
```
```java
// Java: 동일한 패턴 — executor가 shutdown 안 되면 스레드가 계속 살아있음
void handler() {
    executor.submit(() -> doSomethingSlow()); // 마찬가지로 소유자 불명
}
```

둘 다 "이 작업을 누가 끝까지 책임지는가"를 명시하지 않으면 동일하게 문제가 생김.
Go는 문법이 가볍다 보니 오히려 이런 실수를 더 쉽게 저지르는 경향이 있다는 지적도
있음.

## 질문했던 것들

- **Go랑 Java 언어 차이를 비교하려는데 대표적으로 동시성 처리 이슈가 있나?**
  → 위 "핵심 내용" 참고. Java는 pinning처럼 레거시 호환 이슈, Go는 소유권/취소를
  직접 관리해야 해서 생기는 누수 이슈가 대표적.
- **코드로 비교해볼 수 있나? 감이 잘 안 오네** → 위 "예시 / 코드" 참고. 기본 동시
  실행, pinning, 구조화된 동시성, leak 패턴 네 가지를 나란히 비교.

## 참고

- [Go's Concurrency Model vs. Java Virtual Threads: A Practical Comparison - Java Code Geeks](https://www.javacodegeeks.com/2026/05/gos-concurrency-model-vs-java-virtual-threads-a-practical-comparison.html)
- [Concurrency in Go vs Java - Keep It Simple, Stupid!](https://blog.skopow.ski/concurrency-in-go-vs-java)

## 2026-08-30 추가 학습: `synchronized` vs `ReentrantLock`

### 개요

은행 계좌 이체 시뮬레이터 토이 프로젝트(`bank-transfer-concurrency`)에서
`SynchronizedTransferService`와 `LockTransferService`를 나란히 구현해보면서,
`synchronized`와 `ReentrantLock`의 차이를 실제 코드로 비교했다.

### 핵심 내용

- **락 해제 방식**: `synchronized`는 언어 문법이라 블록을 벗어나면(정상 종료든
  예외든) JVM이 자동으로 락을 풀어준다. `ReentrantLock`은 그냥 객체라서
  `lock()`/`unlock()`을 직접 호출해야 하고, `unlock()`을 `finally`에 안 넣으면
  예외 발생 시 락이 영원히 안 풀리는 버그가 생긴다.
- **`tryLock()` / `tryLock(timeout, unit)`**: 락을 못 얻으면 무한정 기다리지 않고
  바로 포기하거나 일정 시간만 기다릴 수 있다. 데드락을 사후 감지하는 게 아니라
  애초에 회피/재시도 전략을 짤 수 있다는 뜻. `synchronized`는 얻을 때까지 무조건
  블로킹이라 이런 선택지가 없다.
- **`lockInterruptibly()`**: 락을 기다리는 스레드를 인터럽트로 깨울 수 있다.
  `synchronized`로 대기 중인 스레드는 인터럽트가 먹히지 않는다.
- **공정성(fairness)**: `new ReentrantLock(true)`로 만들면 가장 오래 기다린
  스레드부터 락을 주도록 강제할 수 있다. `synchronized`는 이런 설정 자체가 없다
  (구현체가 알아서 함, 대개 불공정).
- **조건 변수**: `ReentrantLock.newCondition()`으로 조건 변수를 여러 개 만들 수
  있다. `synchronized`의 `wait()/notify()`는 모니터당 대기 집합이 하나뿐이라
  용도별로 나눠 깨울 수 없다.
- **상태 조회**: `ReentrantLock`은 `isLocked()`, `getQueueLength()` 같은 API로
  락 상태를 프로그램에서 조회할 수 있다. `synchronized`는 이런 introspection이
  없다.
- **성능**: 경합이 적을 땐 현대 JVM의 최적화(적응형 스피닝, 락 코어스닝 등) 덕에
  둘이 비등비등하고, 경합이 아주 심해질 때 `ReentrantLock`의 이점(공정성,
  tryLock)이 드러나는 편.
- **Go와의 대응 관계**: Go에는 `synchronized` 같은 언어 키워드가 없다. `sync.Mutex`는
  자바의 `ReentrantLock`처럼 그냥 객체이고 `Lock()`/`Unlock()`을 직접 호출하는
  방식(보통 `defer mu.Unlock()`으로 `finally`를 흉내냄)이라, 사실 자바의
  `synchronized`보다 `ReentrantLock`이 Go의 `sync.Mutex`에 더 가까운 대응 관계다.
  `TryLock()`도 Go 1.18부터 추가됨.

### 예시 / 코드

두 계좌(`from`, `to`)를 모두 잠가야 하는데, 스레드마다 잠그는 순서가 다르면
데드락이 날 수 있어서 둘 다 계좌 id 기준으로 잠금 순서를 고정했다.

`synchronized` 버전 — 계좌 객체 자체를 모니터로 사용:
```java
public class SynchronizedTransferService implements TransferService {

    @Override
    public boolean transfer(Account from, Account to, long amount) {
        Account first = from.getId().compareTo(to.getId()) <= 0 ? from : to;
        Account second = (first == from) ? to : from;

        synchronized (first) {
            synchronized (second) {
                if (from.getBalance() < amount) {
                    return false;
                }
                from.adjustBalance(-amount);
                to.adjustBalance(amount);
                return true;
            }
        }
    }
}
```

`ReentrantLock` 버전 — `Account`에는 락 필드가 없어서, 계좌 id → `ReentrantLock`
맵을 구현체가 직접 관리해야 했다(그만큼 코드가 더 장황해짐):
```java
public class LockTransferService implements TransferService {

    private final ConcurrentHashMap<String, ReentrantLock> locks = new ConcurrentHashMap<>();

    @Override
    public boolean transfer(Account from, Account to, long amount) {
        boolean fromFirst = from.getId().compareTo(to.getId()) <= 0;
        ReentrantLock firstLock = lockFor(fromFirst ? from : to);
        ReentrantLock secondLock = lockFor(fromFirst ? to : from);

        firstLock.lock();
        try {
            secondLock.lock();
            try {
                if (from.getBalance() < amount) {
                    return false;
                }
                from.adjustBalance(-amount);
                to.adjustBalance(amount);
                return true;
            } finally {
                secondLock.unlock();
            }
        } finally {
            firstLock.unlock();
        }
    }

    private ReentrantLock lockFor(Account account) {
        return locks.computeIfAbsent(account.getId(), id -> new ReentrantLock());
    }
}
```

동일한 시뮬레이션(계좌 10개, 워커 8개 × 10만 회 이체)으로 돌려본 결과, 이 정도
경합에서는 눈에 띄는 성능 차이는 없었다:

```
SynchronizedTransferService: 54ms, 총액 보존 OK
LockTransferService:         67ms, 총액 보존 OK
```

### 질문했던 것들

- **`synchronized`랑 `ReentrantLock`이 두 개 비교는 뭐가 다른가?**
  → 위 "핵심 내용" 참고. 자동/수동 락 해제가 가장 큰 차이이고, `ReentrantLock`은
  tryLock/인터럽트/공정성/조건변수 등 `synchronized`에 없는 제어권을 제공한다.
