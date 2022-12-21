### QuerydslPredicateExecutor<T>
---

QuerydslPredicateExecutor를 확장하면 findAll 같은 기본적인 쿼리를 빠르게 사용할 수 있다. 파라미터로 querydsl의 Predicate를 넘겨주면 된다.

```Java
@Repository
public interface MemberRepository extends JpaRepository<Member, Long>, MemberDynamicRepository, QuerydslPredicateExecutor<Member> {
}
```

<br>

테스트

```Java
@Test
public void QuerydslPredicateExecutorTest() {

    Iterable<Member> res = memberRepository.findAll(
            member.age.goe(10).and(member.age.loe(20))
    );

    assertThat(res)
            .extracting("age")
            .containsExactly(10, 20);
}
```

<br>

그러나 이를 실제 코드에 적용한다면 서비스 레이어가 리포지토리 레이어에만 의존하는 것이 아니고, querydsl에도 의존성을 가지게 된다. 왜냐하면 querydsl의 Predicate를 파라미터로 넘겨주어야 하기 때문이다.

<br>

그리고 묵시적 조인은 가능하나 left join이 불가능하다는 단점도 있다. 따라서 실무에 사용하기에는 한계가 명확하다.