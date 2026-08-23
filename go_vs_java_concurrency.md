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
