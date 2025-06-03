### REST API, 이벤트 기반 아키텍처 비교

.. 면접 준비 하는데 이벤트 기반 처리에 대한 개념이 햇갈려서 정리함;;; (feat. chatGPT)

---

### ✅ 상황 가정: 회원가입 처리

**회원가입 시 필요한 작업 예시**
1. 계정정보 DB 저장
2. 사용자가 업로드한 이미지 CDN 서버에 업로드
3. 이미지 URL 생성 및 저장
4. 클라이언트에 응답 반환

---

### ✅ 전통적인 REST API 방식

- 클라이언트 → `POST /signup`
- 서버는 위의 1~4번 **모든 작업을 순차적으로 수행**
- 하나라도 실패하면 전체 실패 (`500` 또는 `400` 응답)

### 흐름 예시:

```text
요청
 ↓
1. DB 저장
 ↓
2. 이미지 업로드
 ↓
3. 이미지 URL 생성
 ↓
4. 응답 반환
```

- ✅ 장점: 구현이 직관적이고 이해하기 쉬움
- ❌ 단점:
    - 중간에 실패하면 전체 실패 처리됨
    - 응답이 느릴 수 있음
    - 작업 간 결합도가 높음 (tightly coupled)

---

## ✅ 이벤트 기반 아키텍처 방식

- 클라이언트 → `POST /signup`
- 서버는 1번(계정정보 저장)만 수행하고 **즉시 응답 반환**
- `user.created`라는 이벤트를 발행(publish)함
- 이후 작업은 해당 이벤트를 구독(subscribe)하는 모듈들이 처리

### 흐름 예시:
```text
요청
↓
DB 저장
↓
응답 반환 (빠름)
↓
'User Created' 이벤트 발행
↓
[이미지 업로더 워커]
[URL 생성기 워커]
[이메일 발송기 워커]
```



- ✅ 장점:
    - 빠른 응답
    - 작업 간 결합도 낮음 (loosely coupled)
    - 나중에 작업 추가 시 API 수정 없이 이벤트 구독자만 추가
- ⚠️ 단점:
    - 설계가 상대적으로 복잡할 수 있음
    - 분산 환경에 익숙하지 않으면 디버깅 어려움

---

## ✅ 핵심 비교 요약

| 항목 | REST API 방식 | 이벤트 기반 아키텍처 |
|------|----------------|------------------------|
| 처리 방식 | 모든 작업을 API 안에서 처리 | 핵심 작업만 처리 후 이벤트 발행 |
| 실패 시 영향 | 중간 실패 → 전체 실패 | 개별 실패는 전체와 무관 |
| 응답 속도 | 상대적으로 느림 | 빠름 |
| 확장성 | 낮음 (API 코드 수정 필요) | 높음 (리스너만 추가하면 됨) |
| 결합도 | 높음 (tightly coupled) | 낮음 (loosely coupled) |
| 유연성 | 작업 순서 고정 | 병렬 처리 및 재사용 가능 |

---

## ✅ 요약

> **REST API 방식**은 모든 처리가 한 요청 안에서 이루어지고, 하나라도 실패하면 전체 실패가 됩니다.  
> 반면, **이벤트 기반 아키텍처**는 핵심 로직만 처리하고 나머지는 이벤트에 반응하는 구조로 분리되어 유연하고 확장성이 뛰어납니다.

> 당신이 말한:
>
> > “REST API는 전체 작업이 다 끝나야 성공,  
> > 이벤트 기반은 핵심만 하고 나머진 따로 처리”
>
> → 이건 이벤트 기반 시스템을 직관적으로 완벽하게 설명한 표현입니다 🎯

---

#### ❓ Q: 핵심만 처리하고 나머진 따로 처리한다가 이벤트면 Java Spring 에서 별도 스레드 처리해도 이벤트 기반임?

> 예: `/signup` API 요청이 들어왔을 때
> 1. 계정 생성은 `UserService.createUser()`에서 처리
> 2. 이미지 업로드나 이메일 발송 같은 부가 작업은 별도의 스레드로 처리 (`new Thread`, `CompletableFuture.runAsync()` 등)
>
> 이 구조도 이벤트 기반 처리라고 봐야함?

---

### ✅ 결론부터 말하면:

> ❌ **이건 이벤트 기반 아키텍처라고 보긴 어렵습니다.**  
> ✅ **단지 비동기 처리(asynchronous processing)일 뿐입니다.**

---

### 🔍 비교: 비동기 처리 vs 이벤트 기반 처리

| 항목 | 비동기 처리 | 이벤트 기반 처리 |
|------|-------------|------------------|
| 예시 | `new Thread`, `CompletableFuture.runAsync()` | `@EventListener`, Kafka, RabbitMQ |
| 동작 방식 | 특정 로직을 비동기로 실행 | 이벤트 객체를 발행하고, 여러 리스너가 반응 |
| 결합도 | 상대적으로 높음 (함수 호출 명시됨) | 낮음 (발행자와 수신자 분리됨) |
| 목적 | 응답 속도 향상, 처리 분리 | 확장성과 느슨한 결합, 분산 처리 |
| 아키텍처 특성 | 코드 구조만 비동기 | 구조적 아키텍처 패턴 (Publish / Subscribe) |

---

### ✅ 예시 1: 단순 비동기 처리 (이벤트 기반 아님)

```java
@PostMapping("/signup")
public ResponseEntity<?> signup(@RequestBody UserDto userDto) {
    userService.createUser(userDto); // 1. 계정 저장

    // 2. 비동기 처리 (하지만 이벤트는 아님)
    CompletableFuture.runAsync(() -> {
        imageService.uploadToCDN(userDto.getProfileImage());
        emailService.sendWelcomeEmail(userDto.getEmail());
    });

    return ResponseEntity.ok("회원가입 완료");
}
```

### ✅ 예시 2: Spring의 이벤트 기반 구조

```java
// 1. 이벤트 정의
public class UserCreatedEvent extends ApplicationEvent {
    private final User user;
    public UserCreatedEvent(User user) {
        super(user);
        this.user = user;
    }
}

// 2. 이벤트 발행
userService.createUser(userDto);
applicationEventPublisher.publishEvent(new UserCreatedEvent(user));

// 3. 이벤트 리스너
@EventListener
public void handleUserCreated(UserCreatedEvent event) {
    imageService.uploadToCDN(event.getUser().getProfileImage());
    emailService.sendWelcomeEmail(event.getUser().getEmail());
}
```