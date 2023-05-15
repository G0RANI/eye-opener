# localstack을 이용해 SQS, SNS를 테스트 해보자 2편



##  1.요구사항

1. 각 환경 마다 다른 세팅
   - 환경은 test, local, real
2. test 환경은 의존성 주입을 통해 여러 환경을 테스트 할 수 있어야 한다
3. local 환경에서는 localstack을 이용해 연결
4. prod 환경에서는 aws에 연결



## 2. 구현



### 2.1 방법 선택

각 환경마다 다른 세팅을 하기 위해 찾아본 방법들 

##### 1. junit + localstack-utils

junit에서 제공해주는 `localstack-utils` 와 같은 라이브러리를 통해 쉽게 localstack을 실행 시키는 방법이 있다 한다. 

하지만 localstack 하나에 의존하는 방법이라 제외하기로 결정

##### 2. Testcontainers

여러 도커컨테이너를 실행 시켜 테스트코드와 연동할 수 있는 방법을 제공해 준다 한다.

localstack말고도 다른 도커컨테이너에 대한 테스트로 사용할 여지가 보여 선택하려 하였으나...

##### 3. yml 파일을 통한 환경 분리

yml 파일을 통해 요구 사항의 환경을 나눠 테스트와 배포를 할 수 있게 구현을 하도록 하였다.

##### 4. integration test

 yml의 설정을 none으로 설정 후 Testcontainers를 통해 주입을 받을 수 있게 구성하였다.

local pc에 직접 도커컨테이너를 띄워서 테스트 하는 방법도 있겠지만, 개인의 개발 환경에 의존하는 test는 좋지 않다고 판단하여 Testcontainers를 사용하기로 결정하였다.

 아뿔사... 아무리 찾아도 sns의 arn을 설정 할 수 있는 내용을 찾을 수 없어 개인 pc의 localstack을 올려 sqs, sns에 대한 설정을 하기로 하였다. 🥲

##### 5. local application 구동

yml의 설정을 mock으로 설정 후  개인의 pc의 도커컨테이너에 localstack을 올려 사용하기로 결정하였다.

##### 6. real application 구동

yml의 설정을 real로 설정 후  aws로 연결



### 2.2 yml 설정

```yaml
cloud:
  mode: real
...중략
---
spring.config.activate.on-profile: test
cloud:
  mode : none
...중략
---
spring.config.activate.on-profile: local
cloud:
  mode: mock
...중략
---
spring.config.activate.on-profile: real
cloud:
...중략
```

mode로 하여 실행하게끔 할지 결정을 하게 하였다.

- `mode: real  ` : aws 환경으로 연결
- `mode: mock  ` : localstack 환경으로 연결
- `mode: none  ` : 아무것도 주입 받지 않는다   



### 2.3 config 설정

```java
@Component
@Configuration
public class EventConfig {

    private static final String CLOUD_MODE = "cloud.mode";

    @Configuration
    @Import({EventRestConfig.class})
    @ConditionalOnProperty(value = CLOUD_MODE, havingValue = "real")
    @EnableConfigurationProperties({AwsCredentialProperties.class})
    static class EventRealConfig {

    }

    @Configuration
    @Import({EventMockConfig.class})
    @ConditionalOnProperty(value = CLOUD_MODE, havingValue = "mock")
    @EnableConfigurationProperties({AwsCredentialProperties.class})
    static class EventMockConfig {

    }

    @Configuration
    @ConditionalOnProperty(value = CLOUD_MODE, havingValue = "none")
    static class EventNoneConfig {
    }

}
```

2.2 에서 설정한 mode에 따라 다른 환경의 config를 나눠 각각 다른 환경에 따라 bean을 주입 받을 수 있게 구성하였다.



### 2.4 localstack 설정

먼저 docker-compose.yml을 사용하여 컨테이너를 실행한다.

```dockerfile
version: '3.3'

services:
  localstack:
    image: localstack/localstack:0.11.3
    privileged: true
    ports:
      - "4566-4597:4566-4597"
    environment:
      - AWS_DEFAULT_REGION=ap-northeast-2
      - EDGE_PORT=4566
      - SERVICES=sqs,sns
    volumes:
      - "${TMPDIR:-/tmp/localstack}:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

aws cli를 통해 Sns, Sqs를 생성한다.

* SQS 생성

```
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name event
```

* SNS 생성

```
aws --endpoint-url=http://localhost:4566 sns create-topic --name notify
```



### 2.5 integration test

##### 1. Sqs 메세지 전송

test를 위한 config class를 작성해 주었다.

```java
@TestConfiguration
public class LocalStackConfig {

    @Bean
    public AwsMockProperties awsMockProperties() {
        AwsMockProperties properties = new AwsMockProperties();
        properties.setEndPoint("http://localhost:4566");
        return properties;
    }

    @Bean
    public AwsCredentialProperties credentialProperties() {
        AwsCredentialProperties awsCredentialProperties = new AwsCredentialProperties();
        ... 중략
        return awsCredentialProperties;
    }

    @Bean
    public AmazonSQSAsync amazonSQSAsync(AwsCredentialProperties credentialProperties,
                                         AwsMockProperties awsMockProperties) {
        return AmazonSQSAsyncClientBuilder
                .standard()
                .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(awsMockProperties.getEndPoint(), credentialProperties.getRegion()))
                .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(credentialProperties.getAccessKey(), credentialProperties.getSecretKey())))
                .build();
    }

}
```

그 후 테스트 코드 작성

```java
@Slf4j
@Tag("integration")
@ActiveProfiles("test")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, classes = LocalStackConfig.class)
class SqsTest {

    @Autowired
    AmazonSNSClient amazonSNSClient;

    @Test
    void test() throws Exception {
        String ARN = "arn:aws:sns:ap-northeast-2:000000000000:notify";
        PublishResult test = amazonSNSClient.publish(ARN, "test");
        log.info("메세지 Id {}", test.getMessageId());
    }
}
```

결과

```tex
2023-05-15 17:36:31.979  INFO [event,,] 8042 --- [    Test worker] event.LocalStackConfig           : localstack endpoint : http://localhost:4566
2023-05-15 17:36:34.250  INFO [event,,] 8042 --- [    Test worker] event.SqsTest : 메세지 Id adddf105
```



##### 2. Sns 메세지 수신

 Sqslistenr 작성

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class EventListener extends LatchListener {

    private final AwsSqsProperties awsSqsProperties;

    @SqsListener(value = "${cloud.aws.sqs.queue.event}", deletionPolicy = SqsMessageDeletionPolicy.NEVER)
    public void eventListener(@Payload EventMessage message, Acknowledgment ack){
        log.info("=================================== message : {}", message);
        ack.acknowledge();
    }
}
```

메세지 수신 정책이 있는데 그중에 NEVER를 선택하였다.

```
- ALWAYS: 메시지를 수신하는 경우 무조건 삭제한다.
- NEVER: acknowledge()를 통해 삭제 요청을 보내고 해당 요청이 없다면 절대 삭제하지 않는다.
- NO_REDRIVE: Redrive 정책이 설정되어있지 않다면 삭제한다.
- ON_SUCCESS: @SqsListener 어노테이션이 붙은 메소드의 수행이 성공한다면 삭제한다.
```

Sqs message 발행

```tex
aws --endpoint-url=http://localhost:4566 sqs send-message --queue-url http://localhost:4566/000000000000/from-member-to-coupon --message-body '{"id":"1"}'
```

어플리케이션 실행 결과

```tex
2023-05-15 18:08:59.321  INFO [event] 10733 --- [upon-listener-6] event.listener : =================================== message : message(id=1)
```



## 3. 소감

Sns는 test코드를 작성할 수 있었지만 Sqs는 어플리케이션을 직접 구동해서 테스트를 했다는 점이 아쉽다.

업무에서 사용한 코드라 많은 부분이 공유 되지 못한점은 아쉽지만, 

해당 방법을 이용한 결과 테스트와 리펙토링에 있어 더욱 공격적으로 할 수 있어 좋았다.

다른 aws서비스와 사용을 하지 못하였지만 Testcontainers를 이용하여 kafka, redis 등 라던지 다른 서비스를 이용한 테스트도 문제 없이 할 수 있을듯하다. 



출처 : https://techblog.woowahan.com/2638/

​		https://medium.com/@anchan.ashwithabg95/using-localstack-sns-and-sqs-for-devbox-testing-fa09de5e3bbb
