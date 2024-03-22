## gRPC

#### gRPC 특징

- 프로토콜 버퍼 사용
- .proto 파일로 데이터 구조 설계 및 컴파일 해서 사용
- stub을 통해서 데이터 직렬화/역직렬화 (.pb 파일)
- 서버는 HTTP/2 기반으로 데이터를 받아서 처리

<br>

서버에서 데이터를 받는 기본적인 코드 Ref. Golang
```go
func main() {
	lis, err := net.Listen("tcp", ":8088")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	
	s := grpc.NewServer() // HTTP/2 기반
	pb.RegisterConfigStoreServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

정의한 .proto 파일 컴파일 할때
```shell
protoc -I . sample.proto --go_out=.
```
** GOPATH 안맞으면 패스 관련 에러 나오니 참고

<br>


+) 일반 Socket 통신과 gRPC의 차이점

```
일반적인 소켓 통신과 gRPC 사이의 가장 큰 차이점: 데이터 전송방식, 통신 프로토콜

통신 프로토콜
- 일반적인 소켓 통신: TCP, UDP와 같은 표준 API로 데이터를 주고받음. 개발자가 직접 데이터의 전송 및 수신을 관리해야함
- gRPC: HTTP/2를 기반으로 하는 프로토콜 버퍼(Protocol Buffers)를 사용하여 통신. RPC(Remote Procedure Call) 프레임워크를 이용하고 서비스 정의와 메시지 형식을 .proto 파일에 정의하고 컴파일해서 사용

데이터 직렬화
- 일반적인 소켓 통신: 직렬화 및 역직렬화를 수동으로 구현
= gRPC: 프로토콜 버퍼를 사용하여 데이터 직렬화/역직렬화 처리

코드 생성
- 일반적인 소켓 통신: 개발자가 직접 데이터를 주고받는 코드를 작성
- gRPC: .proto 파일을 정의하고, gRPC 도구를 사용하여 클라이언트 및 서버 코드를 자동으로 생성
```


##### Ref.
- https://cloud.google.com/api-gateway/docs/grpc-overview?hl=ko
- https://github.com/GoogleCloudPlatform/golang-samples/tree/main/endpoints/getting-started-grpc
- https://lejewk.github.io/protocol-buffer/