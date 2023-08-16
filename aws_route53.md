## AWS Route53

### 1. 퍼블릭 호스팅 영역 생성(도메인 등록)
<p><img src="image/route53/2023-08-16 route53 hosting.png" alt=""/></p>

- NS: 네임서버 레코드(도메인 등록한 곳에 맞춰줘야 함)
- SOA: 권한시작 레코드


### 2. 하위 도메인 레코드 등록
<p><img src="image/route53/2023-08-16 route53 record.png" alt=""/></p>

- 레코드 이름 : 서브 도메인으로 사용할 도메인 이름
- 레코드 유형을 A type으로 지정할때 : ec2의 퍼블릭 Ip주소 또는 할당받은 탄력적 ip 주소 기입
- 레코드 유형을 CNAME type으로 지정할 때 : 로드밸런서의 DNS 이름 기입


<br><br>



Ref. 
- https://tech.cloud.nongshim.co.kr/2018/10/18/%EC%B4%88%EB%B3%B4%EC%9E%90%EB%A5%BC-%EC%9C%84%ED%95%9C-aws-%EC%9B%B9%EA%B5%AC%EC%B6%95-8-%EB%AC%B4%EB%A3%8C-%EB%8F%84%EB%A9%94%EC%9D%B8%EC%9C%BC%EB%A1%9C-route-53-%EB%93%B1%EB%A1%9D-%EB%B0%8F-elb/
- https://velog.io/@ssssujini99/Web-AWS-Route53%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-%EB%8F%84%EB%A9%94%EC%9D%B8-%EC%97%B0%EA%B2%B0%ED%95%98%EA%B8%B0
- https://eunoia3jy.tistory.com/202
- https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/using-domain-names-with-elb.html