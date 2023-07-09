## AWS Lambda를 이용하여 DLQ에 있는 메세지를 SQS로 redrive 해보자

## 1. 개요

내가 맡고 있는 시스템에서는 AWS SQS + SNS 를 통하여 발생하는 이벤트 대하여 비동기 처리를 하고 있다.

이벤트를 수신 받아 처리 하는 과정에 외부 연동에 문제가 있어 수신 되지 못한 메세지는 DLQ (Dead Letter Queue)에 적재되는 일이 있었다.

DLQ에 적재되는 기준은 10번 초과하여 메세지를 처리 하지 못하면 적재되게 세팅이 되어있다.

문제를 해결 후 콘솔에서 redrive를 통해 정상적으로 처리 되지 못한 메세지들을 처리 하였다.

문제를 해결 하고 나니 문제가 있을때마다 일일이 확인이라던지 알 수 없는 이유로 처리하지 못한 메세지를 감지하고 재처리 하기 위해 (사실 수동으로 처리 하기 귀찮음을 해결 하기 위해, 개발자들은 게을러야 발전한다라는 말을 본듯하다...) AWS Lambda를 통해 처리하는 방식을 선택하고 진행해보았다.



## 2. AWS Lmaba 란?

AWS Lambda은 서버를 프로비저닝하거나 관리하지 않고도 코드를 실행할 수 있게 해주는 컴퓨팅 서비스입니다.

Lambda는 고가용성 컴퓨팅 인프라에서 코드를 실행하고 서버와 운영 체제 유지 관리, 용량 프로비저닝 및 자동 조정, 코드 및 보안 패치 배포, 로깅 등 모든 컴퓨팅 리소스 관리를 수행합니다. Lambda를 사용하면 Lambda가 지원하는 언어 런타임 중 하나로 코드를 제공하기만 하면 됩니다.

Lambda 함수에 코드를 구성합니다. Lambda 서비스는 필요할 때만 함수를 실행하고 자동으로 확장됩니다. 사용한 컴퓨팅 시간만큼만 비용을 지불하고, 코드가 실행되지 않을 때는 요금이 부과되지 않습니다. 자세한 내용은 [AWS Lambda 요금](http://aws.amazon.com/lambda/pricing/)을 참조하세요.

#### Lambda를 사용해야 하는 경우

Lambda는 빠르게 스케일 업해야 하고 수요가 없을 때는 0으로 스케일 다운해야 하는 애플리케이션 시나리오에 이상적인 컴퓨팅 서비스입니다. 예를 들어 Lambda를 다음에 사용할 수 있습니다.

- **파일 처리:** 업로드 후 Amazon Simple Storage Service(S3)를 사용하여 Lambda 데이터 처리를 실시간으로 트리거합니다.
- **스트림 처리:** Lambda 및 Amazon Kinesis를 사용하여 애플리케이션 작업 추적, 거래 주문 처리, 클릭스트림 분석, 데이터 정리, 로그 필터링, 인덱싱, 소셜 미디어 분석, 사물 인터넷(IoT) 디바이스 데이터 텔레메트리 및 계측을 위한 실시간 스트리밍 데이터를 처리합니다.
- **웹 애플리케이션:** Lambda를 다른 AWS 서비스와 결합하여 여러 데이터 센터에서 고가용성 구성으로 자동으로 스케일 업/스케일 다운되고 실행되는 강력한 웹 애플리케이션을 빌드합니다.
- **IoT 백엔드:** Lambda를 사용하여 서버리스 백엔드를 구축함으로써 웹, 모바일, IoT 및 서드 파티 API 요청을 처리합니다.
- **모바일 백엔드:** Lambda 및 Amazon API Gateway를 사용하여 백엔드를 구축함으로써 API 요청을 인증하고 처리합니다. AWS Amplify를 사용하여 iOS, Android, 웹 및 React Native 프론트엔드와 손쉽게 통합합니다.

Lambda를 사용하면 사용자는 자신의 코드에 대해서만 책임을 갖습니다. Lambda는 메모리, CPU, 네트워크 및 기타 리소스의 균형을 제공하는 컴퓨팅 플릿을 관리하여 코드를 실행합니다. Lambda가 이러한 리소스를 관리하므로 컴퓨팅 인스턴스에 로그인하거나 제공된 런타임에 운영 체제를 사용자 지정할 수 없습니다.

Lambda는 사용자를 대신하여 용량 관리, 모니터링 및 Lambda 함수 로깅을 비롯한 운영 및 관리 활동을 수행합니다.

컴퓨팅 리소스를 관리해야 하는 경우 이외에도 다음과 같은 AWS 컴퓨팅 서비스를 고려할 수 있습니다.

- AWS App Runner는 컨테이너식 웹 애플리케이션을 자동으로 빌드 및 배포하고, 암호화를 통해 트래픽 로드 밸런싱을 수행하고, 트래픽 요구 사항에 맞게 확장하고, 프라이빗 Amazon VPC에서 서비스에 액세스하고 다른 AWS 애플리케이션과 통신하는 방법을 구성할 수 있게 합니다.
- Amazon ECS의 AWS Fargate는 가상 머신의 클러스터를 프로비저닝, 구성 또는 확장할 필요 없이 컨테이너를 실행합니다.
- Amazon EC2를 사용하면 운영 체제, 네트워크 및 보안 설정, 전체 소프트웨어 스택을 사용자 지정할 수 있습니다. 용량을 프로비저닝하고, 플릿 상태 및 성능을 모니터링하고, 내결함성을 위해 가용 영역을 사용할 책임은 사용자에게 있습니다.

출처 : https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/welcome.html



역시 공식문서가 제일 내용이 좋다 🙂



## 3. AWS Lambda 구성하기

dlq는 수신이 10번 초과로 실패한 메세지를 2일 동안 보관하고 있다.

메세지 형식은 해당 이벤트를 일으킨 도메인에서 사용하는 키값과 날짜가 포함되어있다.

따라서 처리 방식은 다음과 같이 결정하기로 하였다.

- 매일 새벽 6시 기준
- 처리 되지 못한 모든 dlq를 다시 sqs로 적재
- 대상은 전날에 발생하여 dlq로 적재된 메세지



### 3.1 Lambda 생성

![1.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/1.png?raw=true)

Python으로 코드를 작성하기로 한다. 이름과 런타임을 지정하고 role은 따로 지정안하고 자동생성으로 진행 하였다.



### 3.2 Lambda 코드 작성

![2.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/2.png?raw=true)

소스 코드를 작성 하였다. boto3를 통하여 sqs에 접근, dlq와 sqs url을 통해 메세지를 받는다.

일단 메세지 전송하기 전에 전일 메세지를 모두 받아온다.

`receive_messsage`에 MaxNumberOfMessages 는 한번에 받아올 메세지 수 인데 최대가 10이다.

따라서 메세지를 받아 온 후 삭제 하고 다시 받는 형식으로 만들 예정이다.

해당 코드를 테스트를 하였더니,



### 3.3 lambda 권한 할당 및 세팅

![3.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/3.png?raw=true)

해당 lambda에서 sql 접근 권한이 없어 AccessDenied가 되었다.

IAM에서 해당 lambda에 할당된 role에 sqs에 대한 정책을 주었다.

![4.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/4.png?raw=true)

이번에는 Task timed out 이란 내용을 받았다.

따라서 lambda 구성의 제한 시간을 30초로 늘려주었다.

![5.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/5.png?raw=true)



### 3.4 Event Bridge를 통한 트리거

매일 새벽 6시에 해당 lambda가 실행 될 수 있도록 트리거를 설정하였다.

![6.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/6.png?raw=true)

![7.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/7.png?raw=true)

Event Bridge를 이용하여 cron시간을 추가 해 주었다.



### 3.5 최종 구성

![8.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/8.png?raw=true)

코드는 한번 리펙토링을 하였다.

sqs와 연동되는 부분을 sqs_client.py로 나누고 한 기능만 할 수 있게 함수들을 나누었다.

![9.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/9.png?raw=true)

![10.png](https://github.com/G0RANI/eye-opener/blob/main/image/9-718ed6b5/10.png?raw=true)



## 4. 결과

테스트를 위해 dlq에 대량의 메세지를 생성하였고, 전일자로 생성된 메세지들이 sqs로 다시 적재되는것을 확인 하였다.

외부 연동 하는 서비스가 요즘 문제가 많아 dlq에 메세지가 생기고 다시 redrive하는 일이 있었는데,

이 작업으로 인해 내가 신경 못쓰는 상황에서도 재 기능을 할수 있길 바라본다.

다음에는 해당 lambda 소스코드에 대한 CI/CD를 위해 CodeCommit과 CodePipeline을 통하여 자동 배포되는 아키텍쳐를 만들어 볼 예정이다.

## 5. 추가
2023년 6월 11일 이후로 AWS에서 Dlq를 redrive 시켜주는 Api를 제공해주는듯하다...
https://aws.amazon.com/ko/blogs/korea/a-new-set-of-apis-for-amazon-sqs-dead-letter-queue-redrive/
