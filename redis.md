# Redis

### redis-cli commend
```redis
-- localhost:6379
> redis-cli

-- 다른 서버의 redis를 접속하고 싶으면
> redis-cli -h [호스트명] -p [포트번호]

-- information
> redis-cli info

-- monitoring
> redis-cli monitor

-- 모든 key 조회 (단, Refs 문서 참고했을땐 해당 명령어는 O(N) 명령어라 초반부에만 사용할 것
> redis-cli keys *
```

### Redis data types

- String (key:value)
- List
- Set
- Hashes
- Sorted sets
- Streams

### 데이터 타입 별 사용방법
- String
  - Key:value 형태
  - Insert
    - Single: ```Set <key> <value>```
      - ex) ```Set user_id#1234 object```
    - Multi: ```Set <key1> <value1> <key2> <value2> ... ```
      - ex ) ```mset user_id#1234 object1 user_email rebianne03@naver.com```
  - Select
    - Single: ```Get <key>```
      - ex) ```Get user_id#1234```
    - Multi: ```mget <key1> <key2>```
      - ex) ```mget user_id#1234 user_email```
  - Prefix 방식, suffix 방식에 따라 저장되는 형태가 다름

- List
  - 우리가 평소 알고있는 리스트
  - 앞, 뒤로 넣는건 빠른데 중간 삽입은 좀 느린편임
    - insert
      - ```Lpush <key> <A> -> Key:(A)```
      - ```Rpush <key> <B> -> Key:(A, B)```
      - ```Lpush h <key> <C> -> Key:(C, A, B)```
      - ```Rpush <key> <D, A> -> Key:(C, A, B, D, A)```
    - pop
      - ```Key:(C, A, B, D, A)```
      - ```LPOP <key> -> Pop C, Key:(A, B, D, A)```
      - ```RPOP <key> -> Pop A, Key:(A, B, D)```
    - lpop, blpop, rpop, brpop
      - ```Key: ()```
      - ```LPOP <key> -> No data```
      - ```BLPOP <key>```
  - ~~중간에 삽입할일이 없는 경우라면 써도 좋을지도?~~


### 주의사항
- 하나의 컬렉션에 넣는 데이터는 10000개 이하로 유지 (특정 시간 지나면 지우게끔 구현)
- Expire는 전체 컬렉션에 대한 제약사항이므로 여러개의 컬렉션을 동시에 사용중이라면 주의
- O(N) 명령어 주의 : redis는 Single Thread라서 동시에 한개의 명령어만 수행한다

<hr>

## Redis with Spring Boot

#### Repository
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### application.properties
```properties
# server host, password (패스워드는 따로 설정 안했으면 빼도 됨)
spring.redis.host=localhost
spring.redis.password=
 
# port 및 pool 관련 설정
spring.redis.port=6379
spring.redis.pool.max-idle=8
spring.redis.pool.min-idle=0
spring.redis.pool.max-active=8
spring.redis.pool.max-wait=-1
```

### Ref.
- [Redis는 무엇이고, 어떻게 사용하는 것이 좋은가](https://github.com/binghe819/TIL/blob/master/DB/Redis/Redis%EB%8A%94%20%EB%AC%B4%EC%97%87%EC%9D%B4%EA%B3%A0,%20%EC%96%B4%EB%96%BB%EA%B2%8C%20%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94%20%EA%B2%83%EC%9D%B4%20%EC%A2%8B%EC%9D%80%EA%B0%80/Redis%EB%8A%94%20%EB%AC%B4%EC%97%87%EC%9D%B4%EA%B3%A0,%20%EC%96%B4%EB%96%BB%EA%B2%8C%20%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94%20%EA%B2%83%EC%9D%B4%20%EC%A2%8B%EC%9D%80%EA%B0%80.md)
- [Spring Boot 에서 Redis를 연동하는 방법](https://oingdaddy.tistory.com/310)