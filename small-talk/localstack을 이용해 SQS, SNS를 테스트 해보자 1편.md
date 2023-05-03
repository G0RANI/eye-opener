# localstack을 이용해 SQS, SNS를 테스트 해보자 1편

## 1. 개요
운영중인 시스템에서 도메인 밖 관심사 분리를 하기 위해 Messaging을 통한 처리를 하고 있다. </br>
local 환경에서 aws의 의존성을 줄여 개발 및 운영에 편리함을 더하고자 localstack을 도입하고 테스트와 구현하면서 있었던 문제점에 관해 정리를 해보자 남긴다 </br>

## 2. 상황
개요에서 설명 하였듯 분리를 위해 Messaging을 통해 비동기 처리를 하고 있었다. </br>
여러 Messaging 처리중에 aws의 SNS, SQS를 사용하고 있었고, SNS에 message를 발행하면 구독하고 있는 SQS에서 처리하는 방식으로 구현되어있었다. </br>
하지만 빈번하게 SQS listener에서 메세지를 못받는 상황이 이뤄졌고 그 부분에 대해 리펙토링과 테스트를 작성하였다. </br> 
다른 시스템에서는 로컬 환경에서 aws 개발환경에 직접 붙어 테스트를 하는것을 보았고 안좋은 방향으로 흘러가는듯 하여 로컬에서 aws에 의존하지 않고 개발할 수 있는 방법이 없을까 찾아보게 되었다. </br>
그래서 localstack이란 aws환경을 docker container에서 만들어 테스트를 할 수 있다는 것을 확인 하고 진행하였다. </br>


## 3. localstack
### 3.1 loaclstack 이란?
Site : https://localstack.cloud/ </br>
Github : https://github.com/localstack/localstack </br>
</br>
<img src="https://localstack.cloud/images/landing/demo.webp" width="450px" height="450"></img><br/>
</br>
Local Stack은 Application들을 도커 컨테이너로 만들어서 Cloud환경과 동일한 환경을 구축 할 수 있게 도와주는 도구다. </br>
docker-compose를 통해 정의되며 포트를 Mapping시키므로써 local에서 해당 Application(AWS lambda, S3, EC2, etc...)를 실행하고 호출할 수 있게 된다. </br>

### 3.2 loaclstack 사용 이유

* CloudFormation(CFN)을 통해 변경된 Application(Lambda, EC2, IAM, etc...)을 배포할 때, 작업자1 이 변경된 내용을 테스트 하기 위해서 배포된 코드가 있다면, 이 후 작업자2 는 해당 테스트가 끝나기 전에는 테스트를 할 수 없다. (왜냐하면 작업자 1이 수정된 코드를 배포 후 테스트를 진행하고 있는데, 작업자 2가 다른 코드를 배포하게 되면, 작업자 1이 테스트 하고 있는 코드가 master branch 코드로 초기화 되기 때문) </br>
* AWS는 리전별로 같은 CFN을 배포할 수 있지만, 그렇게 되면 새로운 리전에 같은 코드를 만들게 되고, 구축하는데 드는 Work factor가 증가되게 되며 비용도 배로 지불해야 합니다. 또한 모든 리전이 똑같은 Application을 제공하지 않아서 제약이 있을 수 있다. </br>
* 테스트를 진행하기 위해 '코드 수정 - 배포 - 테스트 - 로그 확인' 의 과정 자체가 많은 Delay time을 발생시킨다. </br> 
특히 CFN으로 배포하게 되었을 때, CFN파일이 커지게 되면 그 시간은 배로 증가하게 된다. (경험이 없어 잘 모르겠다...). </br>
</br>
이중에서 가장 큰 장점은 내가 생각했을 때 테스트를 자유롭게 할 수 있으며, Delay time 또한 많이 줄일 수 있다는 점이고, 확실하게 느꼈다. 

### 3.3 loaclstack 지원하는 서비스
LocalStack도 유료 버전과 무료 버전으로 나눠져 있고, 유료 버전에서는 더 많은 서비스를 제공하는 것 같다. 무료로 제공하는 서비스는 아래. </br>
```
Note: localstack 0.11.0 부터는 모든 APIs 단위 포인트(http://localhost:4566)로 연결을 됨
API Gateway at http://localhost:4567
Kinesis at http://localhost:4568
DynamoDB at http://localhost:4569
DynamoDB Streams at http://localhost:4570
S3 at http://localhost:4572
Firehose at http://localhost:4573
Lambda at http://localhost:4574
SNS at http://localhost:4575
SQS at http://localhost:4576
Redshift at http://localhost:4577
Elasticsearch Service at http://localhost:4578
SES at http://localhost:4579
Route53 at http://localhost:4580
CloudFormation at http://localhost:4581
CloudWatch at http://localhost:4582
SSM at http://localhost:4583
SecretsManager at http://localhost:4584
StepFunctions at http://localhost:4585
CloudWatch Logs at http://localhost:4586
EventBridge (CloudWatch Events) at http://localhost:4587
STS at http://localhost:4592
IAM at http://localhost:4593
EC2 at http://localhost:4597
KMS at http://localhost:4599
ACM at http://localhost:4619
```
