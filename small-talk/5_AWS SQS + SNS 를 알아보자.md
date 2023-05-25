## AWS SQS + SNS 를 알아보자

## 1. 개요

지금 회사에서는 비동기 메세징 처리를 위해 AWS SNS + SQS를 조합하여 사용하고 있다.

하지만 도입하기 이전에 대표적인 메세지 브로커인 kafka나 RabbitMQ 를 두고 고민을 하였다.

도입에 있어 여러 문제가 있었지만..기술의 부채, 리소스, 시간..등등..가장 큰 문제점은 비용이었다....🥲

어찌어찌하여 해당 내용을 도입하였고 SNS와 SQS에 대해 알아보자



## 2. 도입 배경

기존 서비스에서의 도메인간의 연동은 api를 통해 하였으며 그로 인해 각 도메인간의 의존 관계가 높아졌다.

의존 관계를 끊어보고자 상태값을 이용한 처리라던지 (결과적으로는 실패하였다. 쓸때 없는 데이터와 그로 인한 장애가 빈번...) 기획의 수정 등...

여러 방식으로 해결해보자 하였지만 (약 5개월) 근본적으로는 해결하지 못하였고 메세징 시스템을 알아보다 개요에서 설명한 내용으로 SNS + SQS를 조합하여 각 도메인간의 연동을 이벤트를 발행하는 [Event Driven Architecture](https://cloud.google.com/eventarc/docs/event-driven-architectures?hl=ko)로 구성 하게 되었다.



## 3. SNS

Amazon Simple Notification Service(Amazon SNS)는 게시자에서 구독자(*생산자* 및 *소비자*라고도 함)로 메시지를 전송하는 관리형 서비스입니다. 게시자는 논리적 액세스 지점 및 커뮤니케이션 채널인 *주제*에 메시지를 전송하여 구독자와 비동기식으로 통신합니다. 클라이언트는 SNS 주제를 구독하고 Amazon Kinesis Data Firehose, Amazon SQS, AWS Lambda, HTTP, 이메일, 모바일 푸시 알림 및 모바일 문자 메시지(SMS)와 같이 지원되는 엔드포인트 유형을 사용하여 게시된 메시지를 수신할 수 있습니다.

![             게시자가 Amazon SNS 주제에 메시지를 보내고 구독자는 메시지를 받습니다.         ](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/images/sns-delivery-protocols.png)

- **애플리케이션 간 메시징**

  애플리케이션 간 메시징은 Amazon Kinesis Data Firehose 전송 스트림, Lambda 함수, Amazon SQS 대기열, HTTP/S 엔드포인트 및 AWS Event Fork Pipelines와 같은 구독자를 지원합니다. 자세한 내용은 [애플리케이션 간(A2A) 메시징에 Amazon SNS 사용](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-system-to-system-messaging.html) 섹션을 참조하세요.

- **애플리케이션 대 개인 알림**

  애플리케이션 대 개인 알림은 모바일 애플리케이션, 휴대폰 번호 및 이메일 주소와 같은 사용자 알림을 구독자에게 제공합니다. 자세한 내용은 [애플리케이션 대 개인(A2P) 메시징에 Amazon SNS 사용](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-user-notifications.html) 섹션을 참조하세요.

- **표준 및 FIFO 주제**

  FIFO 주제를 사용하여 엄격한 메시지 순서를 보장하고, 메시지 그룹을 정의하고, 메시지 중복을 방지할 수 있습니다. Amazon SQS FIFO 대기열만 FIFO 주제를 구독할 수 있습니다. 자세한 내용은 [메시지 순서 지정 및 중복 제거(FIFO 주제)](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-fifo-topics.html) 섹션을 참조하세요.

  메시지 전송 순서와 잠재적 메시지 중복이 중요하지 않은 경우 표준 주제를 사용합니다. 지원되는 모든 전송 프로토콜은 표준 주제를 구독할 수 있습니다.

- **메시지 내구성**

  Amazon SNS는 메시지 내구성을 제공하기 위해 함께 작동하는 다양한 전략을 사용합니다.

  - 게시된 메시지는 지리적으로 분리된 여러 서버 및 데이터 센터에 저장됩니다.
  - 구독된 엔드포인트를 사용할 수 없는 경우 Amazon SNS는 [전송 재시도 정책](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-message-delivery-retries.html)을 실행합니다.
  - 전송 재시도 정책이 종료되기 전에 전송되지 않은 메시지를 보존하기 위해 [배달 못한 편지 대기열](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-dead-letter-queues.html)을 만들 수 있습니다.

- **메시지 아카이브 및 분석**

  [Kinesis Data Firehose 전송 스트림에서 SNS 주제를 구독](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-firehose-as-subscriber.html)하면 Amazon Simple Storage Service(Amazon S3) 버킷, Amazon Redshift 테이블 등과 같은 아카이브 및 분석 엔드포인트에 알림을 보낼 수 있습니다.

- **메시지 속성**

  메시지 속성을 사용하여 메시지에 대한 임의 메타데이터를 제공할 수 있습니다. [Amazon SNS 메시지 속성](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-message-attributes.html).

- **메시지 필터링**

  기본적으로 각 구독자는 주제에 게시된 모든 메시지를 수신합니다. 메시지의 하위 세트만 수신하려면 구독자는 주제 구독에 필터 정책을 할당해야 합니다. 구독자는 필터 정책 범위를 정의하여 페이로드 기반 또는 속성 기반 필터링을 활성화할 수도 있습니다. 필터 정책 범위의 기본값은 `MessageAttributes` 입니다. 수신 메시지 속성이 필터 정책 속성과 일치하면 메시지가 구독된 엔드포인트로 전송됩니다. 그렇지 않으면 메시지가 필터링되어 제외됩니다. 필터 정책 범위가 `MessageBody` 인 경우 필터 정책 속성이 페이로드와 일치합니다. 자세한 내용은 [Amazon SNS 메시지 필터링](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-message-filtering.html) 섹션을 참조하세요.

- **메시지 보안**

  서버 측 암호화는 AWS KMS에서 제공하는 암호화 키를 사용하여 Amazon SNS 주제에 저장된 메시지 내용을 보호합니다. 자세한 내용은 [저장 시 암호화](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-server-side-encryption.html) 섹션을 참조하세요.

  Amazon SNS와 Virtual Private Cloud(VPC) 간에 프라이빗 연결을 설정할 수도 있습니다. 자세한 정보는 [인터네트워크 트래픽 개인 정보 보호](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-internetwork-traffic-privacy.html)에서 확인하세요.

  

출처 : https://docs.aws.amazon.com/ko_kr/sns/latest/dg/welcome.html

(공식 문서가 너무 잘 되어있어 그대로 읽는것이 좋아보여....머쓱...😅)



## 4. SQS

Amazon Simple Queue Service (Amazon SQS) 는 내구력 있고 가용성이 뛰어난 보안 호스팅 대기열을 제공하며 이를 통해 분산 소프트웨어 시스템과 구성 요소를 통합 및 분리할 수 있습니다. Amazon SQS [데드 레터 대기열](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) 및 [비용 할당 태그와](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-queue-tags.html) 같은 일반적인 구조를 제공합니다. AWSSDK에서 지원하는 모든 프로그래밍 언어를 사용하여 액세스할 수 있는 일반 웹 서비스 API를 제공합니다.



- **보안** — [Amazon SQS 대기열에 메시지를 보내고 받을 수 있는 사용자를 제어할](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-authentication-and-access-control.html) 수 있습니다. 기본 Amazon SQS 관리형 서버 측 암호화 (SSE) 를 사용하거나 () 에서 관리되는 사용자 지정 [SSE](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-server-side-encryption.html) 키를 사용하여 대기열에 있는 메시지 콘텐츠를 보호하여 중요한 데이터를 전송하도록 선택할 수 있습니다.AWS Key Management ServiceAWS KMS
- **내구성** — 메시지를 안전하게 보관하기 위해 Amazon SQS 메시지를 여러 서버에 저장합니다. 표준 대기열은 [at-least-once 메시지 전달을](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html#standard-queues-at-least-once-delivery) 지원하고 FIFO 대기열은 단 [한 번의 메시지 처리](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html) 및 [고처리량](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/high-throughput-fifo.html) 모드를 지원합니다.
- **가용성** — Amazon SQS [중복 인프라를](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-basic-architecture.html) 사용하여 메시지에 대한 고도의 동시 액세스와 메시지 생성 및 사용을 위한 고가용성을 제공합니다.
- **확장성** — Amazon SQS 각 [버퍼링된 요청을](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-client-side-buffering-request-batching.html) 독립적으로 처리하여 프로비저닝 지침 없이 로드 증가 또는 스파이크를 처리하도록 투명하게 확장할 수 있습니다.
- **안정성** — Amazon SQS 처리 중에 메시지를 잠그므로 여러 생산자가 동시에 메시지를 보내고 여러 소비자가 메시지를 받을 수 있습니다.
- **사용자 지정** — 대기열이 똑같을 필요는 없습니다. 예를 들어 [대기열에 기본 지연 시간을 설정할](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html) 수 있습니다. 256KB보다 큰 메시지의 콘텐츠를 [Amazon S3 객체에 대한 포인터를 포함하는 Amazon Simple Storage Service (Amazon S3) 또는 Amazon DynamoDB를 사용하여](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-s3-messages.html) 저장하거나 큰 메시지를 작은 메시지로 분할할 수 있습니다.



출처 : https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html

(이것도 마찬가지...😅)



## 5. 왜 SNS 와 SQS를 조합하는가?

- SNS와 연결된 엔드포인트에 장애 발생시 메세지가 유실 된다.
- 메세지 전송 실패로 인한 유실
- 소비자 에러로 인한 메세지 유실

- 어플리케이션의 확장성이 떨어진다.
  - 예를 들어 회원 가입을 하였을 때 신규가입 쿠폰을 발행하는 이벤트가 있다. 이럴 경우 회원이 가입했다 라는 이벤트를 발생시켜 SQS에 이벤트를 발급 하고 쿠폰 도메인에서는 해당 이벤트를 받아 쿠폰의 정책에 따라 쿠폰을 발급할것이다. 하지만 회원가입 이벤트에 다른 도메인의 이벤트가 필요한 경우가 생기게 된다면 그 해당 도메인이 바라보고 있는 새로운 SQS를 생성 후 그 쪽에도 발급을 해야할 것이다.
  - 하지만 하나의 SNS에 새로 생성되는 도메인에 따라 SQS를 붙이면 확장성을 얻을 수 있다.

- AWS 다른 서비스들과의 확장성이 떨어진다.
  - 이점은 SQS를 조합하는것에 대한 장점은 아니다.
  - SQS를 SNS에 구독하여 사용시에는 SQS에 메세지를 보내는것말고도 Lambda, Kinesis, Redshift, S3 등등... 여러 아키텍쳐와 구성을 할수있다.

생각되어지는 점을 정리하였는데 정확하지는 않다...



## 6. 정리

사용하고 있는 SNS 와 SQS에 대하여 정리를 해보았다.

앓던이 (요새 진짜로 치과를 가고 있다.)를 뺀기분이 이런걸까 진작 진행해볼걸 하는 마음이 컸다.

사실 kafka를 가장 도입하고 싶었지만...비즈니스는 역시 돈이다...

따라서 다음글은 도입하고 싶었던 kafka와 AWS SNS + SQS 그리고 많이 사용한다는 RabbitMQ의  장단점과 차이점에 대한 비교가 될 것 같다.
