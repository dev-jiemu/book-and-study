## Server-Sent Event (SSE)
#### Socket 과 SSE의 차이점
```
Socket
- 대부분 브라우저에서 지원
- 양방향 통신
- 서버 셋업에 따라 브라우저 연결 한도가 정해짐
- 프로토콜: webSocket
```
```
Server-Sent Event
- 대부분 모던 브라우저에서 지원
- 단방향 (Server to Client)
- 3초마다 자동 재접속 시도
- HTTP : 6개까지 가능, HTTP2 : 100개가 기본
- 프로토콜: HTTP
```
### SSE 사용처
- 알람을 주고싶을 때 (ex. twitter 에서 피드를 받을때)

### use SSE with Node.js
```javascript
// header setting
const headers = {
    'Content-Type': 'text/event-stream',
    'Connection': 'keep-alive', // 커넥션 끊으면 안됨
    'Cache-Control': 'no-cache', // 데이터가 새로 올꺼니까 캐시 ㄴㄴ
}
```
```javascript
res.writeHead(200, headers)

const clientId = req.query.clientId
clients[clientId] = {
    res,
}
req.on('close', () => {
    delete clients[clientId] // close 일때 삭제
})
```
연결 후 클라이언트에게 데이터를 보낼 때
```javascript
// ex. 알람 전송
const notifyUser = (req, res) => {
    const payload = {...}
    clients.forEach(client => client.res.write(`data: ${JSON.stringify(payload)}\n\n`))
}
```
클라이언트에서 받을때
```javascript
useEffect(() => {
    const sseEvents = new EventSource('url')
    
    sseEvents.onopen = () => {
        
    }
    
    sseEvents.onerror = () => {
        
    }
    
    sseEvents.onmessage = (result) => {
        const parsedData = JSON.parse(result.data)
    }
}, [])
```