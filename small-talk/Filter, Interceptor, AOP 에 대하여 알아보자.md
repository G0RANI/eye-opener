# Filter, Interceptor, AOP 에 대하여 알아보자

## 1. 개요

ISMS 심사 대비, url에 따른 권한관리를 하기 위해 Interceptor에 권한을 관리 하는 로직을 추가 하였다.

Interceptor 말고도 요청의 처리 과정에서 공통적으로 적용되어야 하는 로직을 분리하여 관리하는 데 도움을 주는 기능에 Filter, AOP 가 있다.

그래서 각각이 어떠한 위치에서 요청을 처리하고 어떠한 사용 시점, 처리 범위, 그리고 적용 가능성에 대해 알아 보았다.



## 2. 흐름

![img](https://t1.daumcdn.net/cfile/tistory/9983FB455BB4E5D30C)

요청이 들어오면 먼저 Filter를 통과한 뒤 DispatcherServlet을 거쳐 Interceptor를 만나고, 그 후 Controller와 AOP를 만나는 순서로 처리된다. 
이렇게 각 단계에서 공통 관심사를 분리하여 처리하면 코드의 중복을 줄이고, 유지 보수성을 향상시킬 수 있다.



## 3. Filter

Java의 서블릿 필터(Filter)는 클라이언트로부터 들어오는 요청을 서블릿이나 JSP 등의 리소스로 보내기 전에 특정 작업을 처리하고, 리소스에서 나가는 응답을 클라이언트로 보내기 전에도 특정 작업을 처리하는 기능을 제공한다. 필터는 보통 요청의 인코딩 처리, 보안 검사, 로깅 및 감사 로그 기록, 이미지 변환, XSL/T 부착, MIME-T 유형 체인 변경 등과 같은 일반적인 작업에 사용된다.

Filter는 Servlet 2.3 버전에서 도입되었으며, Java에서 제공하는 javax.servlet.Filter 인터페이스를 구현하여 사용한다다. 
Filter 인터페이스는 주로 다음 세 가지 메서드를 정의하는 데 사용됩니다.

1. `init(FilterConfig filterConfig)`: 필터의 초기화 파라미터를 설정합니다. 필터 인스턴스가 처음 생성될 때 한 번만 호출된다.
2. `doFilter(ServletRequest request, ServletResponse response, FilterChain chain)`: 클라이언트의 요청이 들어오고 응답을 반환할 때 실질적으로 수행되는 필터링 작업을 정의한다. 
   `doFilter` 메서드는 FilterChain을 매개변수로 받아서 다음 필터를 호출하거나, 만약 마지막 필터라면 타겟 서블릿 또는 JSP 페이지를 호출한다.
3. `destroy()`: 필터 인스턴스가 메모리에서 제거되기 전에 호출되는 메서드로, 필터의 리소스를 해제하거나 설정을 정리하는 데 사용된다.

필터의 실행 순서는 web.xml 또는 자바 어노테이션에 선언한 순서대로 처리된다. 필터들은 체인으로 연결되어 순차적으로 동작하며, 하나의 필터가 작업을 마치면 다음 필터로 넘어간다. 마지막 필터는 서블릿이나 JSP 페이지를 호출하게 된다. 이런 방식을 필터 체인(Filter Chain)이라고 부른다다.



## 4. Interceptor

Interceptor는 Controller에 도달하기 전, 후 그리고 완료 시점에 사용자 요청을 가로채는 역할을 한다. 이를 통해 각 Controller 메소드가 실행되기 전, 후에 공통적으로 처리해야 하는 로직을 구현할 수 있다. 주로 인증, 권한 검사, 로깅, 트랜잭션 처리 등에 사용된다.

Interceptor를 사용하려면 먼저 `HandlerInterceptor` 인터페이스를 구현하거나 `HandlerInterceptorAdapter` 클래스를 상속 받아야 한다. `HandlerInterceptor` 인터페이스는 다음과 같이 세 가지 메소드를 제공한다.

1. `preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)`: Controller 메소드가 실행되기 전에 호출된다. 이 메소드에서 false를 반환하면 Controller를 실행하지 않고, 요청 처리를 중단한다. 따라서, 권한 검사 등의 로직을 이 메소드에 구현하면 된다.
2. `postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)`: Controller 메소드가 정상적으로 종료된 후에 호출된다. 그러나 View가 렌더링되기 전이므로, 이 메소드에서 ModelAndView를 조작하면 View에 영향을 줄 수 있다.
3. `afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)`: 요청 처리가 완전히 끝난 후, 즉 View가 렌더링된 후에 호출된다. 이 메소드는 요청 처리 중에 예외가 발생하더라도 항상 호출된다.

이러한 Interceptor를 Spring MVC에 등록하려면 `WebMvcConfigurer` 인터페이스의 `addInterceptors(InterceptorRegistry registry)` 메소드를 오버라이드하여 구현하면 된다.

```java
javaCopy code
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new MyInterceptor())
            .addPathPatterns("/path/**");  // 적용할 URL 패턴 지정
    }
}
```

이렇게 구현한 Interceptor는 `/path/**`로 시작하는 URL 패턴에 대한 요청에 대해 동작하게 된다.



## 5. AOP (Aspect-Oriented Programming)

AOP (Aspect-Oriented Programming)는 관심사의 분리 (Separation of Concerns)를 구현하기 위한 패러다임이다. AOP는 'Aspect'라는 모듈을 사용하여 공통적으로 필요한 기능을 분리하고, 이를 어디에 적용할지를 정의함으로써, 코드의 재사용성을 높이고, 가독성을 개선하는데 도움을 준다.

AOP는 주로 로깅, 트랜잭션 관리, 보안 등의 공통적인 기능을 처리하는 데에 사용되며, 이를 통해 각 메서드나 클래스가 실제 비즈니스 로직에만 집중할 수 있도록 돕는다.

AOP의 주요 용어

1. Aspect: 공통 기능을 모듈화한 것을 'Aspect'라고 한다. Aspect는 공통 기능과 어디에 적용할지에 대한 정보를 포함하고 있다.
2. Join Point: Aspect를 적용 가능한 위치를 'Join Point'라고 한다. 메서드 호출, 필드 값 변경 등과 같은 프로그램의 실행 흐름에서 특정 지점이 될 수 있다.
3. Advice: Aspect의 공통 기능을 구현한 코드를 'Advice'라고 한다. Advice는 특정 Join Point에서 실행되어야 하는 코드를 의미하며, 언제 실행할지를 결정하는 방법에 따라 Before, After, Around 등으로 나눌 수 있다.
4. Pointcut: 어떤 Join Point에서 Advice가 실행될지를 결정하는 것을 'Pointcut'이라고 한다. Pointcut 표현식을 사용하여 구체적인 Join Point를 지정할 수 있다.
5. Weaving: Aspect를 Target Object에 적용하는 과정을 'Weaving'이라고 한다. 이 과정은 컴파일 시점, 클래스 로드 시점, 또는 실행 시점에 수행될 수 있다.

Spring Framework에서는 Spring AOP와 AspectJ 두 가지 방식을 지원하며, 이를 통해 쉽게 AOP를 구현할 수 있다. 주로 @Aspect 어노테이션을 사용하여 Aspect를 정의하고, @Pointcut, @Before, @After, @Around 등의 어노테이션을 사용하여 Pointcut과 Advice를 정의한다.



## 6. 그렇다면 왜 권한 관리를 interceptor에서 했지?

권한 관리는 주로 사용자의 인증 상태와 그에 따른 요청 처리 권한을 확인하고 관리하는 작업을 포함한다. 이러한 로직을 적용하는 위치는 다양한 요인에 따라 달라질 수 있다. 

1. **Spring Context 접근**: Interceptor는 Spring Context에 접근할 수 있다. 즉, Spring에서 관리하는 다른 Bean들을 이용할 수 있다는 뜻이다. 이는 서비스나 데이터베이스 레이어에 접근하여 인증 또는 권한 검사에 필요한 정보를 가져오는 등의 복잡한 로직을 처리하는 데 유용하다다. 반면에 **Filter는 Servlet Container가 관리하기 때문에 이러한 기능에 제한**이 있다.
2. **Controller 단위로 적용 가능**: Interceptor는 특정 Controller나 특정 URL 패턴에 대해 적용할 수 있다. 즉, 각 Controller나 메소드마다 다른 권한을 요구하는 등의 세밀한 제어가 가능하다. 하지만 **Filter는 모든 요청에 대해 동일하게 적용되므로 이러한 세밀한 제어가 어렵다**.
3. **AOP와의 차이점**: AOP는 비즈니스 로직을 담고 있는 메소드 실행 전후에 추가적인 동작을 적용하는 경우에 매우 유용하다다. 하지만 **AOP는 메소드 레벨에서 동작하기 때문에, HTTP 요청과 응답에 직접 접근하는 것이 어렵다**. 반면에 Interceptor는 HTTP 요청과 응답에 직접 접근할 수 있으므로, 권한 관리와 같이 HTTP 요청에 따라 동작이 변경되어야 하는 경우에 더 적합하다.

## 7. 마무리

"Interceptor", "Filter", 그리고 "Aspect-Oriented Programming (AOP)"은 주로 웹 애플리케이션 개발에서 사용되는 기술이며, 이 세 가지 기술 모두 요청의 처리 과정에서 공통적으로 적용되어야 하는 로직을 분리하여 관리하는 데 도움을 준다. 
그러나 그들의 사용 시점, 처리 범위, 그리고 적용 가능성에는 명확한 차이가 있다.
이 세 가지 기술은 기본적으로 로직의 분리를 돕는 방법론이지만, Filter와 Interceptor는 주로 웹 애플리케이션의 요청/응답 처리 과정에서 사용되며, AOP는 애플리케이션의 모든 곳에서 사용될 수 있다. 따라서 특정 상황에 따라 적절한 기술을 선택하여 사용하는 것이 중요하다.
