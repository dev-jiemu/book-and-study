## How to access container in fargate
#### fargate에 배포된 컨테이너에 접속하는 방법

<br>

### issue
- fargate는 serverless 기반
- 컨테이너에 직접적으로 접근할 수 있는 ec2가 없음
- ec2 기반의 ecs를 운영중이면 docker exec로 들어갈수 있는데 얘는 그게 아님...

### AWS ECS Exec
Ref. https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/ecs-exec.html

~~마저읽어용..~~