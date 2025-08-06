## RabbitMQ for Golang

---


### Setting (macOS)
```bash 

brew install rabbitmq

#서비스 시작
brew services start rabbitmq

# 서비스 중지
brew services stop rabbitmq
```

### Go 의존성 설치
```go
// RabbitMQ Go 클라이언트 설치
go get github.com/streadway/amqp

// 근데 설치해보니 얘를 더 권장하는데?
go get github.com/rabbitmq/amqp091-go
```

---

### 개념정리

##### Connection, Channel
- Connection: 물리적 네트워크 연결
- Channel: 논리적 연결, 실제 AMQP 명령이 전송되는 통로
- 멀티스레드 환경에서는 각 고루틴마다 별도의 Channel 사용 권장


##### Queue
- 메세지가 저장되는 버퍼
- **Producer** : 보내는 애, **Consumer** : 받는 애
- **Queue 속성**:
  - **Name**: 큐 이름 (빈 문자열 시 자동 생성)
  - **Durable**: `true`면 서버 재시작 시에도 큐가 유지됨
  - **Exclusive**: `true`면 해당 연결에서만 사용 가능
  - **Auto-delete**: 마지막 Consumer가 연결 해제되면 자동 삭제
  - **Arguments**: 추가 설정 (TTL, 최대 길이 등)


##### Exchange, Routing
- 메세지를 어디로 보낼지 결정하는 역할
- **Exchange 타입**:
  - **Direct**: Routing key가 정확히 일치하는 큐로 전송
  - **Fanout**: 바인딩된 모든 큐로 브로드캐스트
  - **Topic**: 패턴 매칭으로 라우팅 (`*`, `#` 와일드카드 사용)
  - **Headers**: 메시지 헤더 기반 라우팅


---
### Exception & Recovery

##### Connection Recovery

```go
// 자동 재연결을 위한 구조체
type RabbitMQ struct {
    conn    *amqp.Connection
    channel *amqp.Channel
    done    chan bool
}

// 연결 복구 로직 구현 필요
func (r *RabbitMQ) handleReconnect() {
    // 연결 끊김 감지 및 재연결 로직
}
```

##### Message Acknowledgment

**Auto ACK**: 메시지 전송 즉시 큐에서 제거 (빠르지만 메시지 손실 위험) <br>
**Manual ACK**: 처리 완료 후 수동 확인 (안전하지만 복잡)

```go
// Manual ACK 사용
msgs, err := ch.Consume(queueName, "", false, false, false, false, nil)

for msg := range msgs {
    // 메시지 처리
    processMessage(msg.Body)
    
    // 성공 시 ACK
    msg.Ack(false)
    
    // 실패 시 NACK (재큐잉)
    // msg.Nack(false, true)
}
```

##### Dead Letter Exchange

**설정 방법**:
```go
args := amqp.Table{
    "x-dead-letter-exchange": "dlx",
    "x-dead-letter-routing-key": "failed",
}
ch.QueueDeclare("main_queue", true, false, false, false, args)
```

---

### 성능 관련

##### Prefetch Count

```go
// 한 번에 최대 10개 메시지만 받기
ch.Qos(10, 0, false)
```

##### gracefun shutdown for Docker
```go
var wg sync.WaitGroup
wg.Add(1)  // 고루틴 하나만!

go func() {
    defer wg.Done()
    
    // RabbitMQ 메시지 처리 루프
    for msg := range msgs {
        // 메시지 처리...
    }
}()

// 시그널 처리 (도커에서 중요!)
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

<-sigCh  // 종료 시그널 대기
fmt.Println("Shutting down...")

// 연결 정리
ch.Close()
conn.Close()

wg.Wait()  // 고루틴이 안전하게 종료될 때까지 대기
fmt.Println("Gracefully stopped")
```
**도커 컨테이너 종료** : `docker stop` -> `SIGTERM` -> `SIGKILL` <br>
메세지 처리가 완료될 때 까지 기다림 (+DB 연결, 핸들 정리 포함)


---

### ✨ Round Robin
1. 큐 이름이 같고 
2. 같은 RabbitMQ 서버일때
3. Consumer 가 여러개일 경우 **자동으로 로드밸런싱 처리**가 됨