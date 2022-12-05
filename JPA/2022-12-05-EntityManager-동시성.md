```Java
    @Autowired
    EntityManager em;

    JPAQueryFactory queryFactory = new JPAQueryFactory(em);
```

JPAQueryFactory를 필드로 제공하면 동시성 문제는 어떻게 될까? 

동시성 문제는 JPAQueryFactory를
생성할 때 제공하는 EntityManager(em)에 달려있다. 스프링 프레임워크는 여러 쓰레드에서 동시에 같은
EntityManager에 접근해도, 트랜잭션 마다 별도의 영속성 컨텍스트를 제공하기 때문에, 동시성 문제는
걱정하지 않아도 된다.