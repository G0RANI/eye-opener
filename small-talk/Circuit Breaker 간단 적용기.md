## Circuit Breaker 란?

https://mangkyu.tistory.com/261

## 개요

- 회원 도메인의 문제로 인한 쿠폰 발급, 조회에 대한 문제 발생
- 외부 연동에 대한 유연한 대처와 시스템의 회복력을 위해 도입

## Hystrix vs resilience4j

- Hystirx는 2017년 7월 13일 `Version 1.5.13`과 2018년 11월 17일 `Re-release stable 1.5.11 as 1.5.18` 버전을 이후로 더 이상 새로운 release 버전 X
- **공식** [**github repository**](https://github.com/Netflix/Hystrix)**에도 Hystrix는 더이상 개발 상태가 아닌 유지보수 상태(maintenance mode)라고 공식적으로 명시**
- 따라서 기능도 더 좋고 살아있는 **resilience4j 선택**

## 설정

- yml 설정
  - Sliding Window 알고리즘에 대한 설정
  - 감지해야 할 Exception에 대한 설정 (Record, Ignore)
- 연동 Client
  - `HttpClientErrorException` 을 Ignore로 감지([![img](https://docs.spring.io/favicon.ico)HttpClientErrorException (Spring Framework 6.0.12 API)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/HttpClientErrorException.html) ), 나머지는 Record로 감지
  - fallback을 통하여 공통적인 Response 설정 (**fallback은 ignoreException이 발생해도 실행된다**)
