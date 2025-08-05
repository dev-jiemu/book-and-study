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
