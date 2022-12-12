### 동적쿼리 - BooleanBuilder
---

BooleanBuilder는 쿼리에 조건절(where)을 동적으로 만드는 도구라고 볼 수 있다.

동적인 파라미터를 이용해서 여러 조건절 조합을 생성할 수 있다.

아래는 username, age가 null이 아닌 경우 검색 조건으로 사용하는 쿼리를 생성하는 예제이다.

```Java
@Test
public void booleanBuilder_test() {

    String username = "member1";
    Integer age = 10;

    List<Member> res = searchMember1(username, age);
}

private List<Member> searchMember1(String username, Integer age) {

    BooleanBuilder builder = new BooleanBuilder();

    if(username != null) {
        builder.and(member.username.eq(username));
    }

    if(age != null) {
        builder.and(member.age.eq(age));
    }
    
    return queryFactory
            .selectFrom(member)
            .where(builder)
            .fetch();
}
```

뭐 어렵진 않다. 근데 SQL Injection에 안전한지 궁금해서 검색해보니 그렇게 설계되어 있다고는 한다. 나중에 자세히 알아보도록 해야겠다.