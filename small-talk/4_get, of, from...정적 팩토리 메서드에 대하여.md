## get, of, from...정적 팩토리 메서드에 대하여

## 1. 개요

업무를 하게 되면 여러 객체를 생성하기 위한 factory란 class를 만들어 정적 method로 처리하는 경우가 많았다.

requestDto -> dto

dto -> entity

dto -> clientRequest 

... 등등 아무 생각없이 작성하던 static factory method(정적 팩토리 메서드)에 대해 알아보자.



## 2. 팩토리란 무엇인가?

디자인 패턴에서 말하는 팩토리는 `객체 생성을 캡슐화`하는 것.



## 3. 정적 팩토리 메서드란? 

위에서 설명한 팩토리패턴에서 객체 생성에서 용어를 따와 만든 객체 메서드라 정의 한다.



## 4. 정적 팩토리 예시

해당 예시는 내가 진행하였던 쿠폰 시스템의 정책에 대한 생성 부분이다.

```java
public class CouponPolicyFactory {

    private CouponPolicyFactory() {
    }

    public static CouponPolicyEntity of(CouponPolicyCreateRequest couponPolicyCreateRequest,
                                          Supplier<String> numberSupplier) {
        return CouponPolicyEntity.createBuilder()
                ... 중략
                .build();
    }
}
```

1. constructor(생성자)를 private으로 정의하여 객체를 생성 할 수 없게 하였다. 
2. 네이밍 컨벤션에 맞게 entity를 생성하였다.
3. 정적으로 작성 하였기 때문에 전역 변수는 사용 할 수 없다.



## 5. 생정자와 어떠한 차이가 있지...?

"이펙티브 자바"라는 책을 누구나 한번 쯤은 들어봤을 것이다. 이 책에서 소개된 많은 아이템들 중에서 가장 첫 번째로 나오는 것이 바로 **"생성자 대신 정적 팩토리 메서드를 고려하라"** 이다.



### 5.1 이름

객체 생성을 하는것에 대한 이름에 의존을 하여 어떤 상황에 맞게 객체가 생성되었는지 파악이 쉽다.

```java
public class OrderFactory {

    private OrderFactory() {
    }

    public static OrderEntity from(OrderCreateRequest orderCreateRequest) {
        return OrderEntity.createBuilder()
                ... 중략
                .build();
    }
      
    public static OrderEntity regularOrderFrom(RegularOrderCreateRequest regularOrderCreateRequest) {
        return OrderEntity.createBuilder()
                ... 중략
                .build();
    }
}
```



### 5.2 정적 매서드

정적 매서드로 구현 되어 있기 때문에 매번 새로운 객체 생성이 필요 없다.



### 5.3 하위 자료형 객체 or 인터페이스 구현체

상속을 받은 하위 객체를 반환 가능하다.

또한 인터페이스의 구현체 또한 반환이 가능하다.

```java
public class CouponEventFactory {

    public Optional<CouponEvent> of(List<CouponEvent> couponEvents) {
        return couponEvents.stream()
                .filter(event -> event.isSupport(eventType))
                .findFirst();
    }
}
```

여러 이벤트로 구현된 객체를 찾아 생성하는 예제인데...사실 이부분의 네이밍은 잘 되었는지...모르겠다.

컨벤션또한 Optional로 return하고 있어 Optional.of()를 따왔는데 잘 한지.... 🥲



### 5.4 캡슐화

만약 정적 팩토리 패턴을 사용하지 않는다면 5.3에서 작성한 내용이 그대로 들어날것이다.

객체를 생성하는 입장에서 안의 내용은 알 필요가 없다.


### 5.5 정리
이처럼 정적 팩토리 메서드는 단순히 생성자의 역할을 대신하는 것 뿐만 아니라, 우리가 좀 더 가독성 좋은 코드를 작성하고 객체지향적으로 프로그래밍할 수 있도록 도와 준다. 객체간 형 변환이 필요하거나, 여러 번의 객체 생성이 필요한 경우라면 생성자보다는 정적 팩토리 메서드를 사용해보자.



## 6. 네이밍 컨벤션

- from: 하나의 매개 변수를 받아서 인스턴스를 생성해서 반환합니다.
- of: 여러 개의 매개 변수를 받아서 인스턴스를 생성해서 반환합니다.
- getInstance | instance: 인스턴스를 생성해서 반환합니다. 하지만 항상 동일한 인스턴스를 반환한다는 보장을 하지 않습니다. 주로 싱글톤(singleton)을 구현할 때 많이 사용되는 규칙이지만, 싱글톤과는 무관하며 동일한 인스턴스 반환을 보장하지 않습니다.
- newInstance | create: 항상 새로운 인스턴스를 생성해서 반환합니다. getInstace | instance와 유사한 개념이지만 항상 다른 인스턴스를 반환한다는 개념이 다릅니다.
- get[OtherType]: 특정한 타입의 인스턴스를 생성해서 반환합니다. getInstance처럼 인스턴스를 반환하지만, 전혀 다른 유형의 인스턴스[OtherType]를 반환한다는 차이점이 있습니다.
- new[OtherType]: 특정한 타입의 새로운 인스턴스를 생성해서 반환한다. newInstance처럼 인스턴스를 반환하지만, 전혀 다른 유형의 인스턴스[OtherType]를 반환한다는 차이점이 있습니다.



## 7. 출처

https://velog.io/@nisroeld/GoF-%EB%94%94%EC%9E%90%EC%9D%B8%ED%8C%A8%ED%84%B4-Factory-Method-Pattern

https://velog.io/@ljinsk3/%EC%A0%95%EC%A0%81-%ED%8C%A9%ED%86%A0%EB%A6%AC-%EB%A9%94%EC%84%9C%EB%93%9C%EB%8A%94-%EC%99%9C-%EC%82%AC%EC%9A%A9%ED%95%A0%EA%B9%8C

https://7942yongdae.tistory.com/147
