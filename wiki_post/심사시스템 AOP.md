## AOP



AOP란 Aspect Oriented Programming 의 줄임말로, 그 뜻을 파헤쳐보면 관점지향 프로그래밍 이라는 의미다.

기능을 핵심 비즈니스 로직과 공통 모듈로 구분하고, 핵심 로직에 영향을 미치지 않고 사이사이에 공통 모듈을 효과적으로 잘 끼우넣도록 하는 개발 방법으로써, 공통 모듈(보안 인증, 로깅 요소 등)을 만든 후 코드 밖에서 이 모듈을 비즈니스 로직에 삽입하는것이 AOP 적인 개발이다. 

<br>

등급분류 심사시스템에서는 보안적 요소는 스프링시큐리티의 필터에 의해 관리되어지고 있으며, 로깅요소에 대해서는 바로 이 AOP 를 활용하고 있는데, [스프링의 관점 지향 프로그래밍](https://blog.outsider.ne.kr/843) 을 참고하면 좋다.

<br>

<br>



## 시스템 관리 히스토리



등급분류 시스템의 로깅 기능중 하나인 **시스템 관리 히스토리**에서는 시스템 사용에 대한 유저단위의 로깅을 지원한다.


<br><br>

### CRUD 이력관리

시스템 내에서 활용되는 데이터 생성, 삭제, 갱신, 조회에 대한 로깅을 구현하였다.

이를위해 개발간에 사전에 약속된 메소드네이밍을 활용하였고, 포인트컷은 다음과 같이 작성되었다.

<br>



```java
    @Pointcut("@annotation(com.kakaogames.audit.annotation.History)")
    public void select() {}

    @Pointcut("execution(* com..service.impl.*ServiceImpl.create*(..))")
    public void create() {}

    @Pointcut("execution(* com..service.impl.*ServiceImpl.update*(..))")
    public void update(){}

    @Pointcut("execution(* com..service.impl.*ServiceImpl.delete*(..))")
    public void delete(){}
```



<br>



조회 로깅의경우 페이지 단위의 기록을 하기위해 컨트롤러단에, 나머지 create, update, delete 로깅의경우 엔티티 단위의 로깅생성을 위해 서비스레이어에 조인포인트를 설정하였다.

<br>

위의 포인트컷을 보면 select의경우 @History 커스텀어노테이션이 붙은것에 대해서만 로깅을 하겠다. 라는 의미이며, 이는 뷰페이지(html 렌더링이 되어지는)를 반환하는 컨트롤러에만 붙여서 사용자가 실질적으로 조회하는 페이지단위의 로깅을 구현하기 위해 사용되었다. 



<br>



```java
private void makeSystemUseHistroy(Object inputParam, Object outputData, String logType){
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();

        Member member = memberService.readMemberByPrincipal();
        Map requestMap = ServletRequestToMap.getParameterMap(request);

        createUserId = member.getId();
        uri = request.getRequestURI();

        SystemUseHistory systemUseHistory = SystemUseHistory.builder()
                .createUserId(createUserId)
                .uri(uri)
                .detailContents(detailContents)
                .logType(logType)
                .build();

        systemUseHistoryService.makeSystemHistory(systemUseHistory);
}
```

AOP단에서는 서블릿 객체를 활용하여 통신에 사용되는 오브젝트들을 참조할수 있기때문에, 다음과같이 서블릿를 통해 요청 객체 정보들을 활용하여 로깅에 사용할수있다.



<br>



### 로그인 이력관리

 이를 활용하여 다음과같이 **로그인 이력 관리**에 대해서 활용하였다.

스프링시큐리티의 인증관련 핸들러 객체를 잘 활용하면 다음과같이 로그인 인증이 완료되었을때의 시점을 조인포인트로 사용가능하다.

```java
@After("execution(* org.springframework.security.web.authentication..onAuthenticationSuccess(..))")
    public void after(JoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        String ip = request.getRemoteAddr();
        String createUserId = SecurityContextHolder.getContext().getAuthentication().getName();

        SystemUseHistory systemUseHistory = SystemUseHistory.builder()
                .createUserId(createUserId)
                .uri(request.getRequestURI())
                .detailContents(new StringBuilder("시스템 로그인 접속 IP : ").append(request.getRemoteAddr()).toString())
                .logType(login)
                .build();
        systemUseHistoryService.makeSystemHistory(systemUseHistory);
}
```





<br><br><br>

![](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/aop1.png)

<sub>시스템 사용 히스토리 리스트 뷰</sub>