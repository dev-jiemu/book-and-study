## Docker and Amazon ECS, ECR(Repository)

### Docker
#### Docker 설치
```bash
sudo -i
yum install -y docker
systemctl start docker
docker images
```
#### container 확인
```
docker ps -a
```
#### container 생성 및 실행, 정지 삭제
```
docker create -i -t --name ecs-nginx-container -p 8080:80 [image id]
docker start [container id]

docker stop [container id]
docker rm [container id]
```
#### dockerfile을 이용한 image build
```
docker build --tag [tag name]:[version] .
```

<br>

#### dockerfile 작성
sample
```bash
vi dockerfile
```
```
FROM nginx:latest
MAINTAINER "rebianne03@naver.com"

COPY ./index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "deamon off;"]
```
만약, golang 을 기반으로 만든 어플리케이션을 dockerfile로 만들려고 할때
1. single stage : image 내에 lib 파일까지 포함되는 경우
```
FROM golang:alpine

ADD . . 

RUN go build main.go
CMD ["./main"]
```
2. multi stage : go build 후 빌드 실행파일만으로 image를 만들고 싶을 때
```
FROM golang:alpine AS builder
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64

WORKDIR /build
COPY go.mod go.sum main.go ./
RUN go mod download
RUN go build -o main .
WORKDIR /dist
RUN cp /build/main .

FROM scratch
COPY --from=builder /dist/main .
ENTRYPOINT ["/main"]
```

<br>

#### aws configure
```
aws configure
```
만약, 별도의 profile로 등록하고 싶다면
```
aws configure --profile [profile name]
```
등록한 config 확인하고 싶으면
```
aws configure list
vi ~/.aws/credentials
```
등록한 profile로 aws cli를 사용하고 싶으면
```
aws [service] ls --profile [profile name]
```
<br>

#### AWS ECR
ECR image push 명령어는 <strong>Amazon ECR 레퍼지토리 페이지</strong>에서 확인

example
<p><img src="image/ecr/2023-06-27 ecr.png"/></p>

ECR에 등록된 docker image를 pull 하고 싶다면
```
docker pull [repository image url].ap-northeast-1.amazonaws.com/ecs-nginx:[tag name]
```

<br>


### ECS
1. 네트워크 구성
- VPC, Subnet, Route table, Internet gateway(subnet public 처리)
- public subnet 로드밸런서 구성
- VPC 내 특정 포트에 대한 대상 그룹 생성 및 로드밸런서 연결
2. Task 구성
- 컨테이너에서 사용할 docker image 지정(ECR URL)
- 한 컨테이너에 cpu 용량 등 지정
- family를 통해 task의 버전을 개정하고 무중단배포를 용이하게 할수 있음(docker image update?)
3. Service 구성
- 한 서비스에 테스크를 몇개로 구성할지 정할 수 있음(n중화?)

<br>

Ref
- https://dev.classmethod.jp/articles/ecs-container-service-establishment/
- https://dev-racoon.tistory.com/23
- https://devlog-wjdrbs96.tistory.com/296
- https://waspro.tistory.com/426
- https://abbo.tistory.com/442
