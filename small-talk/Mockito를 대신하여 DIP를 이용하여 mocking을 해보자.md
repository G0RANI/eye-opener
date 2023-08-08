# Mockito를 대신하여 DIP를 이용하여 mocking을 해보자

# 1. 개요

Repositroy에 대한 연동 테스트가 필요 할 시 기존에는 H2 DB를 이용하여 미리 스키마와 데이터를 준비 하고,
@SpringBootTest와 @BeforeAll를 이용하여 데이터를 밀어넣고 테스트 코드를 작성하였다.
이렇게 작성을 하다보니 테스트의 속도도 느리고 Entity의 변화, 트랜잭션에 따른 데이터 정합성 등... 쉽게 깨지는 테스트가 많이 작성이 되었다.
운영 하는 DB는 H2도 아니여서 의미가 있는 테스트인가라는 생각도 많이 들었다. 
그리고 기존에는 service의 일부분의 테스트이니 단위테스트 아냐? 라고 생각을 하였지만 이부분에 대해 알아보니 외부 연동, DB연동 등... 직접 연동하는 테스트는 중형테스트에 맞다는 내용을 보았다.
이에 따라 외부 연동하는 부분을 기존에도 사용하고 있던 Mockito를 이용하여 외부 요인에 대한 의존성을 제거(?) 하여 테스트 하는 방식으로 바꿨다.
하지만 이에도 문제가 있었으니... Mockito를 이용하다보니 해당 라이브러리에 대한 의존도 늘어나고 오히려 테스트가 보내주는 안좋은 신호를 무시하고 테스트를 작성하는 경향이 생김을 인지 했다.
따라서 Mockito를 사용하는 방식에서 DIP를 이용하여 테스트 코드를 작성하는 방식으로 넘어가게 되었다.



# 2. 기존 테스트

```java
@Slf4j
@ActiveProfiles("test")
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@SpringBootTest(classes = CouponApplication.class)
public class CouponServiceTest {

    @Autowired private DataSource dataSource;
  	@Autowired private CouponService couponService;
		@Autowired private CouponRepository couponRepository;
  	// 중략...
    @BeforeAll
    void beforeAll() {
        try (Connection conn = dataSource.getConnection()) {
            ScriptUtils.executeSqlScript(conn, new ClassPathResource("/db/h2/init_campaign.sql"));
            ScriptUtils.executeSqlScript(conn, new ClassPathResource("/db/h2/init_coupon_policy.sql"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @AfterAll
    void afterAll() {
        couponRepository.deleteAllInBatch();
        // 중략...
    }

    @AfterEach
    void afterEach() {
        couponRepository.deleteAllInBatch();
      	// 중략...
    }

    @Order(1)
    @Test
    void 생일_쿠폰이_정상_생성된다() throws Exception {
        // given
        CouponCreateRequest request = CouponCreateRequest.builder()
                // 중략...
                .build();

        // when
      	CouponCreateResponse result = couponService.create(request);

        // then
        assertThat(result.getCouponStatus()).isEqualTo(CouponStatus.PUBLISHED);
      	// 중략...
    }
}
```

해당 테스트는 기존의 H2 DB를 이용한 테스트이다.
이 테스트 구조에서 Mockito를 사용하는 테스트로 변경을 하였다.

```java
@ExtendWith(MockitoExtension.class)
public class CouponServiceTest {
    
		@InjectMocks
    private CouponService couponService;
  	@Mock
		private CouponRepository couponRepository;
  	// 중략...

    @Test
    void 생일_쿠폰이_정상_생성된다() throws Exception {
        // given
        CouponEntity entity = CouponEntity.builder()
                // 중략...
                .build();
        given(couponRepository.save(any())).willReturn(entity);
        CouponCreateRequest request = CouponCreateRequest.builder()
                // 중략...
                .build();

        // when
      	CouponCreateResponse result = couponService.create(request);

        // then
        assertThat(result.getCouponStatus()).isEqualTo(CouponStatus.PUBLISHED);
      	// 중략...
    }
}
```

테스트를 변경 후 속도와 쉽게 깨지지 않는 테스트가 되었다.
하지만 Mockito에 의존하고 이에 따라 좋은 설계를 놓치는 부분에 대해 고려를 하였다.

# 3. 한번 더 진화

먼저 Mockito에 의존하고 있는 Repository에 대한 부분을 DIP를 적용하였다.
`CouponRepository` 에 대한 명칭을 `CouponJPARepository` 로 변경하고 `CouponRepository` 인터페이스를 새롭게 만든 후 `CouponRepositoryImpl`을 만들어 `CouponJPARepository` 를 주입 받아 사용 하는 방식으로 변경 하였다.

```java
public interface CouponRepository {
    CouponEntity save(CouponCreateRequest couponCreateRequest);
    // 중략...
}
```

```java
@Repository
@RequiredArgsConstructor
public class CouponRepositoryImpl implements CouponRepository {
    private final CouponJPARepository couponJPARepository;
    @Override
    public CouponEntity save(CouponCreateRequest couponCreateRequest) {
        // 중략...
        CounponEntity newCouponEntity = couponJPARepository.save(couponEntity)
        return newCouponEntity;
    }
}
```

이후 테스트 코드에 주입할 구현체 `CouponTestRepositoryImpl` 를 작성하였다.

```java
public class CouponTestRepositoryImpl implements CouponJPARepository {
    @Override
    public CouponEntity save(CouponCreateRequest couponCreateRequest) {
        // 중략...
        return CouponEntity.builder()
                // 중략...
                .build();
    }
}
```

그 후 테스트를 다시 작성하였다.

```java
public class CouponServiceTest {
    
    private CouponService couponService = new CouponService(new CouponTestRepositoryImpl(),
                                                           // 중략...);  	
  	// 중략...

    @Test
    void 생일_쿠폰이_정상_생성된다() throws Exception {
        // given
        CouponEntity entity = CouponEntity.builder()
                // 중략...
                .build();
        CouponCreateRequest request = CouponCreateRequest.builder()
                // 중략...
                .build();

        // when
      	CouponCreateResponse result = couponService.create(request);

        // then
        assertThat(result.getCouponStatus()).isEqualTo(CouponStatus.PUBLISHED);
      	// 중략...
    }
}
```



# 4. 이점이 무엇이지?

내가 생각해보고 찾아 정리한 이점이다.

1. **테스트 격리**: 외부 의존성을 제거하거나 모킹함으로써 테스트는 자신의 목적에 집중할 수 있게 되고, 테스트 실패의 원인을 더욱 명확하게 파악할 수 있게 된다.
2. **테스트 속도 향상**: 메모리 데이터베이스 사용이나 외부 연동을 모킹함으로써, 실제 DB 연동보다 훨씬 빠른 테스트 실행이 가능하다. 이로 인해 개발자는 자주 테스트를 실행할 수 있게 되고, CI/CD 파이프라인에서의 테스트 실행 시간도 단축된다. 실제로 배포 전에 항상 clean build 과정을 거치고 코드를 merge 하는데 이 과정에 걸리는 시간이 상당히 줄었다.
3. **유연성**: DIP (의존 역전 원칙)을 사용하면, 테스트를 위한 구현체 교체나 다른 구성 변경이 용이해진다. 만약에 조회하는 부분이 엘라스틱서치 같은 검색 엔진으로 변경이 되었다면 기존의 repositroy는 커버할 수 없는 문제들을 테스트를 통해 유연한 설계를 얻고 빠르게 대응할 수 있는 부분을 얻었다.
4. **테스트의 안정성**: 외부 의존성이나 환경 변화에 의해 테스트가 실패하는 경우를 줄일 수 있다. Mockito가 아니라 순수한 java로 작성하였기 때문에 해당 라이브러리가 언제 변경이 되어도 걱정이 없는 안정성을 얻었다.
5. **진짜 문제에 집중**: Mockito와 같은 도구에 지나치게 의존할 때 발생하는 가짜 통과 문제를 줄여, 실제 코드의 문제점을 더 정확하게 포착할 수 있게 된다. 알람 이벤트에 대한 배치 리펙토링 할때 실제로 가짜 통과 문제를 발견하여 수정을 할 수 있었다.



# 5. 마무리

글에서는 DB에 대한 부분만 DIP를 적용하여 테스트 코드 리펙토링을 하였는데 외부 연동에 대한 부분에도 적용이 가능한 내용이다.
나 같은 경우는 RestTemplate를 활용하는 테스트는 환경에 따라 실제로 연결하는 RestTemplate를 주입받아 테스트를 하는 부분과 모킹 하는 부분을 나눠 테스트를 작성 하는 등 유연하게 테스트를 할 수 있는 환경을 얻었다. 아무런 의존성 없이 순수히 java로 테스트 코드를 작성 하였다는게 굉장히 마음에 든다.
