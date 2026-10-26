# Go sync.Once

> 정리일: 2026-10-20

## 개요
이벤트 루프 기반 세션의 종료 콜백(OnStop) 데드락을 수정하면서 종료 신호를 전용 채널(stopCh)로 분리했고,
그 채널을 한 번만 닫기 위해 sync.Once를 처음 써 봤다.
sync.Once가 무엇이고 언제 쓰는지, 그리고 주의할 점을 정리했다.

## 핵심 내용

### sync.Once란
- 어떤 함수를 그 Once 객체 기준으로 **딱 한 번만** 실행되게 보장한다.
- 여러 고루틴이 동시에 호출해도 실제 실행은 한 번뿐이다.

```go
var once sync.Once
once.Do(f) // 처음 호출에서만 f 실행, 이후 호출은 아무것도 안 함
```

### 언제 쓰나
"동시에 여러 번 불릴 수 있는데 실제 동작은 한 번만 일어나야 할 때" 쓴다.

1. **채널을 한 번만 닫기** — 종료 신호용 채널. 가장 흔한 용도
2. **지연 초기화(lazy init)** — 처음 필요한 순간에 한 번만 만든다
3. **Go 1.21+ 축약형** — `sync.OnceFunc`, `sync.OnceValue`, `sync.OnceValues`

### 플래그(bool)로 직접 막으면 안 되는 이유
- `if !closed { close(ch); closed = true }`처럼 쓰면, 두 고루틴이 동시에 `!closed`를 통과해서
  둘 다 close할 수 있다(race).
- Once는 이 경우까지 안전하게 처리한다.

### lazy init에서의 동작
- 처음 호출이 동시에 여러 개 들어오면, 나머지 호출은 **첫 실행이 끝날 때까지 기다렸다가**
  완성된 값을 받는다.
- 그래서 반쯤 초기화된 값을 보는 일이 없다.

### 주의할 점
- **f가 panic나도 "실행 완료"로 친다.** 재시도되지 않으므로, 초기화가 실패할 수 있다면 에러를
  따로 저장해 둔다. `OnceValues`로 `(값, error)`를 받는 방법이 있다.
- **f 안에서 같은 Once의 `Do`를 다시 호출하면 데드락**이 난다.
- **복사하면 안 된다.** Mutex와 같은 규칙이다. 값으로 넘기지 말고 구조체 필드로 둔다.
- **되돌릴 수 없다.** 한 번 실행되면 끝이라, 리셋이 필요하면 Once가 아닌 다른 방식을 쓴다.

## 예시 / 코드

### 채널을 두 번 close하면 panic
```go
close(ch)
close(ch) // panic: close of closed channel
```

### 적용 사례: 이벤트 루프의 종료 신호
- 문제: 미디어 서버 라이브러리(lal)는 서버 전역 락(ServerManager.mutex)을 쥔 채 `OnStop` 콜백을 호출한다.
  그런데 `OnStop`은 버퍼 채널(`eventCh`, 4칸)에 종료 이벤트를 send하고 있었다.
  평소에는 버퍼에 자리가 있어 send가 바로 끝나지만, **버퍼가 가득 찬 상태**면 루프가 꺼내 줄 때까지
  락을 쥔 채 기다리게 되고, 그때 두 가지 문제가 생겼다.
  - 노드 전체의 publish 시작/종료가 지연된다.
  - 같은 락이 필요한 `kick()`과 서로를 기다리는 데드락이 난다.
- 수정: 종료 신호를 전용 채널로 분리하고, 기다리지 않는 close로 알린다.
- `OnStop`이 두 번 불리면 close를 두 번 하게 되어 panic이 나므로, Once로 막았다.

```go
type Session struct {
	stopCh   chan struct{}
	stopOnce sync.Once
}

func (v *Session) OnStop() {
	v.stopOnce.Do(func() { close(v.stopCh) }) // close 는 블로킹되지 않음 + 중복 close panic 방지
}

// runEventLoop
stopCh := v.stopCh
for {
	select {
	case <-stopCh:
		stopCh = nil // 닫힌 채널은 계속 선택되므로 한 번 처리 후 nil 로 바꿔 select 에서 제외
		// ... 종료 처리
	}
}
```

### lazy init
```go
var (
	clientOnce sync.Once
	client     *http.Client
)

func getClient() *http.Client {
	clientOnce.Do(func() { client = &http.Client{Timeout: 5 * time.Second} })
	return client
}

// Go 1.21+
getClient := sync.OnceValue(func() *http.Client { return &http.Client{} })
```

## 질문했던 것들
- **Q. sync.Once는 뭐 할 때 쓰나?**
  - 동시에 여러 번 호출될 수 있지만 실제 동작은 한 번만 일어나야 할 때 쓴다.
  - 대표적인 예는 채널 한 번만 닫기, 지연 초기화다.
- **Q. `OnStop`이 send에서 기다리는 문제는 항상 생기나?**
  - 아니다. 버퍼 채널은 자리가 있으면 send가 바로 끝난다.
  - 기다리는 건 **버퍼가 가득 찼을 때뿐**이다. 그래서 평소에는 드러나지 않다가, 이벤트가 몰리는
    장애 상황에서만 터지는 문제였다.
