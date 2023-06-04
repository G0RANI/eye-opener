## 상속보단 컴포지션을 사용하라(Prefer Composition Over Inheritance)

## 1. 개요

이펙티브 자바의 책에서 보면 이런 구절이 있다.

"상속보다는 컴포지션을 사용하라(Prefer Composition Over Inheritance)"

 이는 상속보다는 객체 간의 컴포지션(구성)을 활용하라는 내용으로, 상속의 사용을 자제하고 객체 간의 관계를 위임과 조합을 통해 구현하라는 주장을 하고 있다.

이 내용을 통해 내가 리펙토링한 내용과 다른 예시들을 공유 한다.



## 2. 왜 컴포지션을 이용해야 할까?

- 유연한 디자인

컴포지션은 상속에 비해 더 유연한 디자인을 제공한다. 상속은 부모 클래스와 자식 클래스 간의 강한 결합을 가져오며, 부모 클래스의 변경이 자식 클래스에 영향을 줄 수 있다. 이는 시스템의 유지보수와 확장을 어렵게 만들 수 있다. 반면에 컴포지션은 객체 간의 느슨한 결합을 제공하여, 객체의 변경이 다른 객체에 영향을 덜 주고 수정이 용이한 구조를 유지할 수 있다.

- 코드 재사용

상속은 부모 클래스의 코드를 자식 클래스로 상속받아 재사용하는 방식이지만, 컴포지션은 객체 간의 협력을 통해 필요한 기능을 조합하여 재사용한다.

- 계층 구조의 제한

상속을 남용하면 클래스 간에 긴밀한 계층 구조가 형성될 수 있다. 이는 복잡한 상속 체인과 강한 의존성을 초래할 수 있으며, 수정과 확장이 어려워질 수 있다. 컴포지션은 계층 구조에 구속받지 않고 객체를 조합하여 유연한 구조를 형성할 수 있다.

- 인터페이스 분리 원칙 (Interface Segregation Principle, ISP)

컴포지션은 인터페이스를 통한 협력을 강조하며, 객체 간에 필요한 기능을 명확하게 정의할 수 있도록 한다. 이는 ISP를 준수하여 인터페이스의 응집성을 높이고 클라이언트 코드의 종속성을 최소화하는데 도움을 준다.



상속은 적절한 상황에서 유용한 도구이지만, 상속 계층의 디자인을 잘못할 경우 유연성과 확장성의 문제를 야기할 수 있다. 따라서, "상속보다 컴포지션"이라는 원칙은 객체 지향 설계에서 유지보수 가능하고 유연한 코드를 작성하기 위해 권장되는 방법이다.



## 3. 나의 경우

기존에 결제 시스템을 구축할때 각 결제 수단별로 비슷한 하나 조금씩 다른 부분이 있었다.

템플릿 메소드 패턴 (Template Method Pattern)을 이용하였다.

```java
@Slf4j
@RequiredArgsConstructor
public abstract class PayTemplate {
		
    private final PgClients pgClients;
    private final PaymentValidator PaymentValidator;
    private final PaymentCommandService paymentCommandService;

    public final void pay(PayRequest payRequest) {
        PaymentValidator.valid(payRequest);
      	Pgreasponse pgreasponse = pay(payRequest, pgClients);
      	createPayment(payRequest, pgResponse);
    }

    public abstract Pgreasponse pay(PayRequest payRequest, PgClients pgClients);
  
  	private void createPayment(PayRequest, payRequest, Pgreasponse pgResponse) {
      paymentCommandService.createPayment(payRequest, pgresponse)
    }

}

```

대략 적인 이런 구조에

```java
public class PayCreditCard extends PayTemplate {
  @Override
  public Pgreasponse pay(PayRequest payRequest, PgClients pgClients) {
      return pgClients.pay(payRequest);
  }
}
```

결제 수단별로 상속을 받아 사용 하였다.

하지만 결제의 추가 내용이 생기고 결제 수단별로 필요한 부분이 달라지고 추가 해야 하는 부분이 생김에 따라 없어도 될 부분이 생겼다.

```java
@Slf4j
@RequiredArgsConstructor
public abstract class PayTemplate {
		
    private final PgClients pgClients;
    private final PaymentValidator PaymentValidator;
    private final PaymentCommandService paymentCommandService;

    public final void pay(PayRequest payRequest) {
        PaymentValidator.valid(payRequest);
      	Pgreasponse pgreasponse = pay(payRequest, pgClients);
      	CashReceipts cashReceipts = issueReceipt(payRequest, pgClients);
      	createPayment(payRequest, pgResponse);
    }

    public abstract Pgreasponse pay(PayRequest payRequest, PgClients pgClients);
  
  	public abstract issueReceipt issueReceipt(PayRequest payRequest, PgClients pgClients);
  
  	private void createPayment(PayRequest, payRequest, Pgreasponse pgResponse) {
      paymentCommandService.createPayment(payRequest, pgresponse)
    }

}
```

가상계좌 결제 수단이 생겨났고 이에 따라서 현금영수증을 따로 발급해줘야 하는 경우가 생겼다.

가상계좌에만 추가를 원하였지만,

```java
public class PayCreditCard extends PayTemplate {
  @Override
  public Pgreasponse pay(PayRequest payRequest, PgClients pgClients) {
      return pgClients.pay(payRequest);
  }
  
  @Override
  public CashReceipts issueReceipt(PayRequest payRequest, PgClients pgClients) {
      return null;
  }
}
```

신용 카드에도 이러한 내용이 추가 되버리고 말았다.

따라서 해당 내용에 대한 리펙토링을 이렇게 하였다.

먼저 결제하는 부분의 추상클레스를 인터페이스로 교체 하였다.

```java
public interface PayTemplate {
	public void pay(PayRequest payRequest);
}
```

그리고 결제 흐름을 관리 할 수 있는 클레스를 하나 만들었다.

```java
@Slf4j
@RequiredArgsConstructor
public class PayManager {

  private final PgClients pgClients;
  private final PayTemplate payTemplate;
  private final PaymentValidator PaymentValidator;
  private final PaymentCommandService paymentCommandService;
  
  public final void pay(PayRequest payRequest) {
        PaymentValidator.valid(payRequest);
      	Pgreasponse pgreasponse = payTemplate.pay(payRequest, pgClients);
      	createPayment(payRequest, pgResponse);
  }
  
  private void createPayment(PayRequest, payRequest, Pgreasponse pgResponse) {
      paymentCommandService.createPayment(payRequest, pgresponse)
  }
  
}
```

그 후 신용카드와 가상계좌 부분에 만들었던 template로 구현을 하였다.

```java
public class PayCreditCard implements PayTemplate {
  @Override
  public Pgreasponse pay(PayRequest payRequest, PgClients pgClients) {
      return pgClients.pay(payRequest);
  }
}
```

```java
public class PayVirtualAccount implements PayTemplate {
  @Override
  public Pgreasponse pay(PayRequest payRequest, PgClients pgClients) {
      Pgreasponse pgreasponse =  pgClients.pay(payRequest);
    	issueReceipt(payRequest, pgClients);
      return pgreasponse;
  }
  
  public CashReceipts issueReceipt(PayRequest payRequest, PgClients pgClients) {
      return pgClients.issueReceipt(payRequest);
  }
}
```



## 3. 그렇다면 상속을 사용해야 하는 경우는?

이러한 경우에도 불구하고 상속을 사용해야 하는 경우를 알아보았다.

- 클래스의 일반화

상속은 클래스의 일반화를 표현하는 데 유용하다. 여러 클래스 간에 공통된 특징과 동작이 있을 때, 이를 상속 계층으로 표현하여 공통 코드를 재사용할 수 있다. 예를 들어, "동물" 클래스가 있고 이를 상속받은 "고양이"와 "개" 클래스를 만들어 공통된 기능을 상속할 수 있다.

- 기능 확장

기존 클래스의 기능을 확장해야 할 때 상속을 사용할 수 있다. 부모 클래스의 기능을 상속받고, 새로운 기능을 추가하여 기존 동작을 확장하는 것이 가능하다. 이는 기존 코드를 수정하지 않고 새로운 기능을 추가할 수 있어 유용하다.

- 다형성과 호환성

상속은 다형성을 구현하는 데에도 사용된다. 상속을 통해 부모 클래스 타입으로 자식 클래스의 객체를 다룰 수 있어 다형성을 실현할 수 있다. 이는 코드의 유연성과 확장성을 높여준다. 또한, 이미 사용 중인 코드와의 호환성을 유지하면서 새로운 기능을 추가할 때 상속을 사용할 수 있습니다.

- 프레임워크와 라이브러리 확장

프레임워크나 라이브러리의 확장을 위해 상속을 사용할 수 있다. 기존의 클래스나 인터페이스를 상속받아 새로운 클래스를 작성하여 추가 기능을 구현하거나 특화된 동작을 제공할 수 있다.

위의 경우들은 상속이 유용하게 사용될 수 있는 몇 가지 예시다. 그러나 상속을 사용할 때에는 상속의 함정에 빠지지 않도록 주의해야 한다. 상속 계층 구조가 복잡해지거나 부모 클래스와 자식 클래스 간에 강한 결합도가 발생할 수 있으므로, 상황에 따라 상속보다는 컴포지션과 위임을 고려해야 한다. 내가 생각하기에는 이러한 내용중에 조금만 더 수고스럽더라도 대부분이 컴포지션을 통해서 해결이 가능하고 프레임워크와 라이브러리 확장 정도에만 상속을 쓰는것이 좋다고 생각한다.



## 4. 정리

컴포지션을 사용하면 기존 코드를 변경하지 않고도 새로운 기능을 추가하거나 기존 기능을 수정할 수 있다. 

또한, 객체 간의 결합도가 낮아져 코드의 유지보수성이 향상된다. 

이러한 이점들로 인해 컴포지션은 상속보다 유연하다. 

상속은 약한 결합도와 높은 응집도를 갖는 객체 지향 원칙에 따르지 못할 수도 있다. 

따라서, 위임과 상속을 적절히 혼합하여 사용하거나, 위임을 우선적으로 고려하고 상속을 대안으로 사용하는 것이 좋다. 
