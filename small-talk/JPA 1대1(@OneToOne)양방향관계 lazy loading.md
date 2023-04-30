# JPA 1대1(@OneToOne)양방향관계 lazy loading의 문제점

## 1. 문제 상황
4월의 매출을 위해서 급하게 이벤트 쿠폰을 발행 해야는 상황이 되었다. </br>
화요일(25일) 오후에 결정이 났고 목요일(27일) 오후에 적용을 해야 한다.  </br>
오픈 초기라 전에 만들어둔 대량의 쿠폰 발급 batch를 이용하기로 결정 하지만 한번도 사용, 테스트를 안해본 내용이었다. </br>
수요일(26일) 오전까지 결정난 내용으로 수정 개발을 하여 테스트를 진행 하였고, 별 문제 없이 끝나고 당일....  </br>
불안한 예감은 틀리지 않았고 해당 배치는 정상적인 기능은 하였으나 발급에 있어 약 30분  시간을 소요 하였다.  </br>
불행중에 일단 발급은 잘되었고 원인을 파악... </br>
* 발급 되어야 하는 쿠폰 수 : 35000건
* 소요 시간 : 30분
* 기능 동작 : 정상

## 2. 문제 해결
### 2.1 프로세스
쿠폰 발행 프로세스는 조회, 유효성 검사, 발행의 순이다. </br>
entity 단위로 움직이며 조회는 querydsl을 사용, 저장은 entity id 채번 방식에 있어 bulik insert가 불가능했기 때문에 </br> 
JDBC Template를 사용하여 bulk insert를 구현함 </br>
```
1. 쿠폰을 발급 받을 대상 조회
2. 발급을 할 쿠폰 켐페인(외부로 보일 내용) 조회
3. 조회 한 켐페인을 대상으로 하는 쿠폰 정책(세부 내용) 조회
4. 대상을 정책으로 유효성 검사
5. 저장
6. view를 담당하는 테이블에 동기화
```
조회와 insert는 이미 구현을 하여 다른 로직에서도 사용하여 이상이 없을것이라고 예상했으나....발생...

### 2.2 문제 원인
batch log를 확인한 결과 조회 한적 없는 entity를 반복하여 조회 하고 있는것을 확인하였다. </br>
분명 조회하는 부분을 모두 찾아 보았지만 해당 entity를 사용하고 있는 부분을 찾지 못하였다. </br>
따라서 조회를 하는 entity를 추적하니 OneToOne 관계를 가진 다른 entity를 확인 할 수 있었다 </br> 
```java
public class CouponEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long couponId;
    .
    .
    .

    @OneToOne(mappedBy = "couponEntity", fetch = FetchType.LAZY, cascade = {CascadeType.ALL}, orphanRemoval = true)
    private CouponUsedEntity couponUsedEntity;
}
```
분명히 코드에서는 조회 전략이 lazy로 설정이 되어있었으나 설정이 적용이 안되었다 </br>
해당 관계는 양방향 @OneToOne 관계로 되어있다.

### 2.3 원인 파악
**양방향 관계에서는 연관관계의 주인이 호출할 때는 지연 로딩이 정상적으로 동작하지만, 연관관계의 주인이 아닌 곳에서 호출한다면 지연 로딩이 아닌 즉시 로딩으로 동작한다라는 내용을 확인 할 수 있었다.** </br>
지연 로딩은 로딩되는 시점에 Fetch 전략이 Lazy로 설정되어있는 엔티티를 프록시 객체로 가져온다. </br>
지연 로딩으로 설정이 되어있는 엔티티를 조회할 때는 프록시로 감싸서 동작하게 되는데, 프록시는 null을 감쌀 수 없기 때문에 이와 같은 문제점이 발생하게 된다. </br>
즉, 프록시의 한계로 인해 발생하는 문제이다.</br>
CouponEntity 조회할때 CouponUsedEntity 프록시 객체로 가져오게 된다. </br>
이후 실제로 CouponEntity 객체를 사용하는 시점에 초기화 되면서 쿼리가 실행된다. </br>

### 2.4 해결 방안
찾아본 해결 방안으로는 

* 구조 변경하기 </br>
  * 양방향 매핑이 반드시 필요한 상황인지 다시한번 생각해본다. </br>
  * OneToOne -> OneToMany 또는 ManyToOne 관계로 변경이 가능한지 생각해본다. </br>
* 구조를 유지한채 해결하기 </br>
  * CouponEntity 조회할때 CouponUsedEntity 함께 조회한다. (Fetch Join) </br>
  * batch fetch size를 사용한다. </br>

1. 구조를 변경하는 방법은 키값의 위치에 따라 양방향의 관계를 유지 해야하기 때문에 불가 </br>
2. 구조를 유지한채 해결하는 방법에서 기존의 코드를 최대한 변경 안하고 할 수 있는 방법중에 조회를 할때 같이 하는 방향 </br>

2번 방법을 통해 해결하였다.
```java
public List<CouponEntity> findNotExistLiveCoupon(LocalDate publishDate) {
        return from(couponEntity)
                .leftJoin(couponUsedEntity).on(couponEntiy.eq(couponUsedEntity.couponEntiy))
                .where(...중략))
                .fetch();
}
```

### 2.5 결과
로컬에 운영과 같은 데이터를 내려 테스트
* 발급 되어야 하는 쿠폰 수 : 35000건
* 소요 시간 : 10초 이내
* 기능 동작 : 정상

## 3 정리
기존에 잘 동작하던 내용이더라도 잘 테스트를 하여 진행하자... </br>
핑계 아닌 핑계이지만 개발 기간이 너무 짧았다... </br>
하지만 해당 내용은 모르고 있던 내용이라 정리할 기회가 있어서 나쁘지는 않은듯... </br>
