### QuerydslRepositorySupport
---

JPA에서 지원하는 이 지원 클래스는 스프링 데이터가 제공하는 페이징을 Querydsl로 편리하게 변환하는 메서드를 제공한다.

```Java
getQuerydsl().applyPagination()
```

그러나 이를 이용하면 메서드 체인이 끊겨서 코드가 길어지는 단점이 있고, querydsl 3.X을 기준으로 만들어져서 select가 아닌 from 문으로 쿼리문을 시작해야 한다는 단점이 있다.

또한 sort도 안되고 queryFactory를 제공하지 않는다는 단점도 있다.

따라서 이러한 단점을 극복하고 querydsl 4.X 이상 버전에 적합한 지원 클래스를 만들어보자. applyPagination()의 경우 5.X 이상 버전에 맞게 다시 수정했다.

```Java
// import ...

/**
 * Querydsl 4.x 버전에 맞춘 Querydsl 지원 라이브러리
 *
 * @author Younghan Kim
 * @see
org.springframework.data.jpa.repository.support.QuerydslRepositorySupport
 */
@Repository
public abstract class Querydsl4RepositorySupport {
    private final Class domainClass;
    private Querydsl querydsl;
    private EntityManager entityManager;
    private JPAQueryFactory queryFactory;

    public Querydsl4RepositorySupport(Class<?> domainClass) {
        System.out.println("construct() 수행");
        Assert.notNull(domainClass, "Domain class must not be null!");
        this.domainClass = domainClass;
    }

    @Autowired
    public void setEntityManager(EntityManager entityManager) {
        System.out.println("@Autowired setEntityManager() 수행");
        Assert.notNull(entityManager, "EntityManager must not be null!");

        JpaEntityInformation entityInformation = JpaEntityInformationSupport.getEntityInformation(domainClass, entityManager);
        SimpleEntityPathResolver resolver = SimpleEntityPathResolver.INSTANCE;
        EntityPath path = resolver.createPath(entityInformation.getJavaType());

        this.entityManager = entityManager;
        this.querydsl = new Querydsl(entityManager, new PathBuilder<>(path.getType(), path.getMetadata()));
        this.queryFactory = new JPAQueryFactory(entityManager);
    }

    @PostConstruct
    public void validate() {
        System.out.println("@PostConstruct validate() 수행");
        Assert.notNull(entityManager, "EntityManager must not be null!");
        Assert.notNull(querydsl, "Querydsl must not be null!");
        Assert.notNull(queryFactory, "QueryFactory must not be null!");
    }

    protected JPAQueryFactory getQueryFactory() {
        return queryFactory;
    }

    protected Querydsl getQuerydsl() {
        return querydsl;
    }

    protected EntityManager getEntityManager() {
        return entityManager;
    }

    protected <T> JPAQuery<T> select(Expression<T> expr) { // 기본 select 쿼리 지원
        return getQueryFactory().select(expr);
    }

    protected <T> JPAQuery<T> selectFrom(EntityPath<T> from) { // 기본 selectFrom 쿼리 지원
        return getQueryFactory().selectFrom(from);
    }

    protected <T> Page<T> applyPagination(Pageable pageable, Function<JPAQueryFactory, JPAQuery<T>> contentQuery, Function<JPAQueryFactory, JPAQuery<Long>> countQuery) {

        JPAQuery<T> jpaContentQuery = contentQuery.apply(getQueryFactory());
        List<T> content = getQuerydsl().applyPagination(pageable, jpaContentQuery).fetch();

        JPAQuery<Long> jpaCountQuery = countQuery.apply(getQueryFactory()); // (1)

        return PageableExecutionUtils.getPage(content, pageable, jpaCountQuery::fetchOne);
    }
}
```

(1): querydsl 5.X 이상 버전에는 count 쿼리를 단독으로 수행하도록 권장하고 있다. 그래서 countQuery가 구현된 람다를 전달받아 queryFactory를 통해 수행한다. 이때 파라미터에 JPAQuery의 타입 파라미터를 Long으로 명시해서 컴파일 오류를 잡아주었다. 

원래 코드를 수행하면 어떤 오류가 발생하는지 살펴보자. 

```Java
protected <T> Page<T> applyPagination(..., Function<JPAQueryFactory, JPAQuery> countQuery) {

        ...

        JPAQuery jpaCountQuery = countQuery.apply(getQueryFactory());

        return PageableExecutionUtils.getPage(content, pageable, jpaCountQuery::fetchOne); // error -- Bad return type in method reference: cannot convert java.lang.Object to long
    }
```

이는 사실 반환타입이 long인 fetchCount를 이용하면 해결되지만 fetchCount가 deprecated 되었기 때문에 fetchOne을 이용하는 것이 좋다.

그런데 fetchOne이 현재 Object 타입을 반환하는 이유는 제네릭의 타입 파라미터를 명시하지 않았기 때문이다. 

fetchOne()을 살펴보면 반환 타입을 JPAQuery의 타입 파라미터로 추론하는 것을 알 수 있다.

```Java

// public class JPAQuery<T> extends AbstractJPAQuery<T, JPAQuery<T>>

// public abstract class AbstractJPAQuery<T, Q extends AbstractJPAQuery<T, Q>> extends JPAQueryBase<T, Q>

@Override
public T fetchOne() throws NonUniqueResultException {
    ...
}
```

따라서 타입 파라미터를 Long으로 명시하면 모든 문제가 해결된다.

```Java
protected <T> Page<T> applyPagination(..., Function<JPAQueryFactory, JPAQuery<Long>> countQuery) {

    ...

    JPAQuery<Long> jpaCountQuery = countQuery.apply(getQueryFactory());

    return PageableExecutionUtils.getPage(content, pageable, jpaCountQuery::fetchOne);
}
```

굳이 함수 파라미터 부분에는 명시하지 않아도 되겠지만 런타임 시점에 문제가 발생할 수 있으므로 함께 명시해주도록 한다.

```Java
protected <T> Page<T> applyPagination(..., Function<JPAQueryFactory, JPAQuery<Long>> countQuery)
```

런타임에 무슨 문제?? 라고 한다면 아래 코드를 살펴보자

```Java
@Repository
public class MemberTestRepository extends Querydsl4RepositorySupport {

    public MemberTestRepository() {
        super(Member.class);
    }

    public List<Member> basicSelect() {
        return select(member).from(member).fetch();
    }

    public Page<MemberTeamDto> applyPagination(MemberSearchCond cond, Pageable pageable) {

        return applyPagination(
                pageable,
                queryFactory -> queryFactory
                        .select(Projections.constructor(
                                MemberTeamDto.class,
                                member.id,
                                member.username,
                                member.age,
                                team.id,
                                team.name
                        ))
                        .from(member)
                        .join(member.team, team)
                        .where(
                                usernameEq(cond.getUsername()),
                                teamNameEq(cond.getTeamName()),
                                ageGoe(cond.getAgeGoe()),
                                ageLoe(cond.getAgeLoe())),
                queryFactory -> queryFactory
                        .select(member.count()) // count()를 빼도 모를 수 있음
                        .from(member)
                        .join(member.team, team)
                        .where(
                                usernameEq(cond.getUsername()),
                                teamNameEq(cond.getTeamName()),
                                ageGoe(cond.getAgeGoe()),
                                ageLoe(cond.getAgeLoe())
                        )
        );
    }

    private Boolean isEmpty(String cond) {
        return !hasText(cond);
    }

    private Boolean isEmpty(Integer cond) {
        return cond == null;
    }

    private BooleanExpression usernameEq(String username) {
        return isEmpty(username) ? null : member.username.eq(username);
    }

    private BooleanExpression teamNameEq(String teamName) {
        return isEmpty(teamName) ? null : team.name.eq(teamName);
    }

    private BooleanExpression ageGoe(Integer age) {
        return isEmpty(age) ? null : member.age.goe(age);
    }

    private BooleanExpression ageLoe(Integer age) {
        return isEmpty(age) ? null : member.age.loe(age);
    }
}
```

주석에 써둔대로 .count() 부분을 빼먹을 수 있다. 이건 컴파일 시점에도 잡을 수 없고 실행하면 count 쿼리가 아닌 select 쿼리가 나가게 된다. 분명히 실수할 수 있는 여지를 남겨두는 것이기 때문에 그냥 타입을 명시하는 것이 좋다.

타입을 명시하면 .count()를 빼먹었을 때 컴파일 시점에 다음과 같은 에러가 발생한다.

```Java
Bad return type in lambda expression: JPAQuery<Member> cannot be converted to JPAQuery<Long>
```

<br>

### 테스트
---

테스트는 이전 테스트 코드와 비슷하다.

```Java

/**
 * packageName    : study.querydsl.repository
 * fileName       : MemberTestRepositoryTest
 * author         : Kim
 * date           : 2022-12-23
 */
@SpringBootTest
@Transactional
class MemberTestRepositoryTest {

    @Autowired
    MemberTestRepository memberRepository;

    @Autowired
    EntityManager em;

    @BeforeEach
    public void injection() {

        Team teamA = new Team("teamA");
        Team teamB = new Team("teamB");
        em.persist(teamA);
        em.persist(teamB);
        Member member1 = new Member("member1", 10, teamA);
        Member member2 = new Member("member2", 20, teamA);
        Member member3 = new Member("member3", 30, teamB);
        Member member4 = new Member("member4", 40, teamB);
        em.persist(member1);
        em.persist(member2);
        em.persist(member3);
        em.persist(member4);
    }

    @Test
    void applyPaginationTest() {

        MemberSearchCond cond = new MemberSearchCond();
        PageRequest pageRequest = PageRequest.of(0, 2);

        Page<MemberTeamDto> memberTeamDtos = memberRepository.applyPagination(cond, pageRequest);

        assertThat(memberTeamDtos.getContent())
                .extracting("username")
                .containsExactly("member1", "member2");

        assertThat(memberTeamDtos.getTotalElements()).isEqualTo(4);
    }
}

// 발생 쿼리

select
    member0_.member_id as col_0_0_,
    member0_.username as col_1_0_,
    member0_.age as col_2_0_,
    team1_.team_id as col_3_0_,
    team1_.name as col_4_0_ 
from
    member member0_ 
inner join
    team team1_ 
        on member0_.team_id=team1_.team_id limit ?

//

select
    count(member0_.member_id) as col_0_0_ 
from
    member member0_ 
inner join
    team team1_ 
        on member0_.team_id=team1_.team_id
```


## 끝~ 🎉🎊🎇🎆🥳