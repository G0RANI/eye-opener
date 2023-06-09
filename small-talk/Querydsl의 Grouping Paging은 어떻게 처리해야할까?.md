## Querydsl의 Group by Paging은 어떻게 처리해야할까?

## 1. 개요

Querydsl의 Group by 를 하였을 때 발생하는 문제가 있다.

Query에 GroupBy 절이 포함된다면 fetchCount(), fetchResults() 메서드를 사용할 수 없다.

해당 문제에 대하여 어떻게 회피를 해야 할지 작성해 본다.

## 2.  Group by Paging에서 발생할 수 있는 문제들

Group by가 포함된 결과에 대한 페이징에 대한 문제이다.

1. 정렬 순서 문제: GroupBy를 사용하면 결과를 그룹화하기 때문에 정렬 순서가 일반적인 경우와 다를 수 있다. Paging은 일반적으로 정렬된 결과를 기반으로 작동하므로, GroupBy와 함께 Paging을 사용하면 예상치 못한 결과를 얻을 수 있다. 특히, 같은 페이지에 속한 결과가 다른 그룹에 속할 수 있다.
2. 페이지 크기 문제: Paging은 보통 페이지 단위로 결과를 반환하는 것을 의미합니다. 그러나 GroupBy를 사용하면 각 그룹당 하나의 결과가 반환되기 때문에 페이지 크기의 개념이 모호해진다. 예를 들어, 페이지 크기를 10으로 설정하면 10개의 그룹이 아닌 10개의 결과를 반환할 수 있습니다.
3. 효율성 문제: GroupBy는 데이터를 그룹화하는 작업을 수행하므로 처리 시간이 더 오래 걸릴 수 있다. 특히 대량의 데이터에 대해 GroupBy를 수행하는 경우에는 성능 저하가 발생할 수 있다. 이로 인해 Paging 작업이 더 오래 걸리거나 응답 시간이 느려질 수 있다.
4. 데이터의 불완전성 문제: GroupBy를 사용하면 결과 집합에서 그룹의 개수가 줄어들기 때문에 페이지 크기를 채우기 위해 일부 결과가 생략될 수 있다. 이로 인해 전체 결과를 정확하게 보여주지 못할 수 있다.

이로 인한 문제들과 만약 Query에 GroupBy 절이 포함된다면 fetchCount(), fetchResults() 메서드를 사용할 수 없을 때는 어떻게 해결해야 할까?

## 3. 어떻게?

```java
public Page<EntityDto> getEntitiesWithGroupBy(Pageable pageable) {
  
    // 쿼리 작성
    getQuerydsl().createQuery()
      .select(
      	new QEntityDto(entity.field1, entity.field2, ...)
      )
      .from(entity)
      .groupBy(entity.field1, entity.field2, ...);

    // 전체 결과 수 계산
    int totalCount = query.fetch().size();

    // 페이징 처리
    query.offset(pageable.getOffset())
         .limit(pageable.getPageSize());

    // 결과 조회
    List<EntityDto> results = query.fetchResults();

    return new PageImpl<>(results, pageable, totalCount);
}
```

1. `Pageable` 객체를 사용하여 페이지 번호, 페이지 크기 등의 정보를 전달받는다.
2. 원하는 쿼리를 작성한다.
3. 먼저 전체 결과 수를 계산한다.
4. 쿼리에 페이징 처리를 한다.
5. 결과를 조회 한다.
6. 생성한 Page 구현체를 return 한다.



## 4. 문제가 해결 된것인가?

내 생각에는 근본적인 문제는 해결 되지 않았다고 생각한다.

Querydsl에서 서브쿼리에 대한 적용이 어렵다던지 해당 문제와 같은것들은 뭔가 애초에 하지 말라는 뜻 처럼 느껴지기 때문이다.

좋은 설계는 제약을 두는것이라 배웠는데 이에 맞는 내용이 아닐까한다.

따라서 해당 문제가 나왔을시에는 기획이나 업무 방향을 변경 하는 방법이 가장 좋다고 생각한다.
