## Spring Security



자체등급 분류 시스템에서는 인증 및 권한관리를 위해 Spring security를 사용한다. <br><br>



스프링시큐리티의 기본 동작방식은 [Spring Security와 보안, 첫번째 이야기](http://www.nextree.co.kr/p1886/) 를 참고하자.



<br><br>

최신버전의 스프링부트에서는 어노테이션기반의 스프링 시큐리티 사용이 가능하다. 

@EnableWebSecurity 어노테이션만 설정해주면 기본 동작방식에서 이야기하는 기존 XML설정에서 포함하고있는springSecurityFilterChain이 자동으로 등록된다.



<br><br>

**인증** 을 위한 메커니즘은 WebSecurityConfigurerAdapter 구현체의 configure 메소드를 오버라이딩함으로써 커스터마이징 가능하다. (WebSecurityConfigurerAdapter는 시큐리티 설정 클래스이다)

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class CustomWebSecurityConfigurerAdapter extends WebSecurityConfigurerAdapter {
    
    ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic()
                .and()
                    .authorizeRequests()
                        .antMatchers("/js/**", "/css/**", "/fonts/**", "/font-awesome/**", "/img/**", "/favicon.ico", "/login", "/signup/**", "/terms/**").permitAll().and()
                        // api path의 경우 ResourceServerConfiguration 에서 처리하도록 처리
                        .requestMatcher(new AntPathRequestMatcherWrapper("/api/**") {
                            @Override
                            protected boolean precondition(HttpServletRequest request) {
                                return false;
                            }
                        }).authorizeRequests()
                    .anyRequest().authenticated()
                .and()
                    .formLogin() // 폼기반 로그인
                    .loginPage("/login") //로그인 커스텀 페이지로 라우팅
                    .permitAll()
                    .defaultSuccessUrl("/main") // 로그인 성공시 라우팅
                    .successHandler(simpleAuthenticationSuccessHandler())
                .and()
                    .logout() // 로그아웃시 동작
                    .invalidateHttpSession(true)
                    .permitAll()
                .and().exceptionHandling().accessDeniedPage("/login?auth"); //권한이 없을시 라우팅
        http.csrf().disable();
        http.headers().defaultsDisabled().cacheControl();
        http.addFilterBefore(customFilterSecurityInterceptor(), FilterSecurityInterceptor.class); // custom filter 등록
    }

    public FilterSecurityInterceptor customFilterSecurityInterceptor() {
        FilterSecurityInterceptor filterSecurityInterceptor = new FilterSecurityInterceptor();
        filterSecurityInterceptor.setAuthenticationManager(customAuthenticationManager());
        filterSecurityInterceptor.setSecurityMetadataSource(filterInvocationMetadataSource());
        filterSecurityInterceptor.setAccessDecisionManager(affirmativeBased());

        return filterSecurityInterceptor;
}
```

<br>

블로그 포스팅과 위의 코드를 비교하여 생각해보면, configure 메소드에서는 securityFilterChain이 동작하는 과정과정의 커스터마이징을 기반하고 있음을 발견할수 있다.<br>



 예를들어 LogoutFilter에서의 동작의 경우

```java
.logout() // 로그아웃시 동작
        .invalidateHttpSession(true)
        .permitAll()
```

과 같은 코드라인을 통해,



<br>

폼기반 로그인을 수행하기 위한 HTML 생성에 관여하는 DefaultLoginPageGeneratingFilter의 동작의 경우 다음과 같은 코드라인을 통해 커스터마이징된다.

```java
.formLogin() // 폼기반 로그인
            .loginPage("/login") //로그인 커스텀 페이지로 라우팅
            .permitAll()
            .defaultSuccessUrl("/main") // 로그인 성공시 라우팅
            .successHandler(simpleAuthenticationSuccessHandler())
```

<br><br>

이렇게 자바기반의 설정에서는 구현체를 통한 **인증 메커니즘**이 설계된다.



<br><br>

하지만 심사시스템에서는 인증뿐만아니라 **권한에 대한 제어**가 필요하다. 이러한 **권한 제어 메커니즘**의 경우 FilterSecurityInterceptor 를 커스터마이징한 customFilterSecurityInterceptor를 통해 설계된다. 

(filterChain이 동작하는 과정에서의 커스터마이징된 인터셉터가 동작하는것으로 생각하면된다)

```java
http.addFilterBefore(customFilterSecurityInterceptor(), FilterSecurityInterceptor.class);
```

<br>

<br>



심사시스템에서 커스터마이징하여 등록한 클래스는 3가지가 있다.

```java
filterSecurityInterceptor.setAuthenticationManager(customAuthenticationManager());
filterSecurityInterceptor.setSecurityMetadataSource(filterInvocationMetadataSource());
filterSecurityInterceptor.setAccessDecisionManager(affirmativeBased());
```



* AuthenticationManager


* InvocationMetadataSource

* AffirmativeBased

<br>

**AuthenticationManager**의 경우 폼기반 로그인을 통해 올바른 접근인지에 대해 판단하는 클래스이다(인증과정에 관여)

```java
@Autowired
    private MemberRepository memberRepository;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        SHAEncoder shaEncoder = new SHAEncoder(256);
        String id = authentication.getName();
        String password = shaEncoder.encode(authentication.getCredentials().toString());


        if(StringUtils.isEmpty(id) || StringUtils.isEmpty(password)){
            throw new BadCredentialsException("Empty Credential");
        }

        List<GrantedAuthority> grantedAuthorities;
        Member member = memberRepository.findByIdAndPassword(id, password);

        //해당 멤버가 없을경우, 삭제된 아이디일경우 null return
        if(member == null || member.getIsAllowed() == -1){
            return null;
        } else{
            //oauth_token 요청 시 roleId 6만 허용
            if(authentication.getDetails().toString().contains("grant_type=")) {
                if(member.getRoleId() != 6) {
                    throw new BadCredentialsException("Invalid Api User Authentication");
                }
            }
            grantedAuthorities = Arrays.asList(new SimpleGrantedAuthority(member.getRoleInfo().getCode()));
            return new UsernamePasswordAuthenticationToken(id, password, grantedAuthorities);
        }
}
```

내부적인 메커니즘은 authenticate 메소드를 오버라이딩하여 구현하고있다.



<br><br>

**InvocationMetadataSource**는 DB를 이용한 URL 패턴별 권한조회를 하여 requestMap을 재구성 할수 있도록 대상정보를 저장한다.

```java
@Override
    @Transactional
    public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {

        FilterInvocation fi = (FilterInvocation) o;
        String uri = fi.getHttpRequest().getRequestURI();   // ex :) /users, /menu
        int slashPos = uri.indexOf("/", 1);

        //대분류 URI
        String sliceUri;
        //소분류 URI
        String subSliceUri;

        if (slashPos > 0) sliceUri = uri.substring(0, slashPos); // ex :) sliceUri = /admin/member => /admin
        else sliceUri = uri;

        log.info("####### URI : {}", uri);

        // 토큰 header 없는 api 요청 시 권한제어
        if(sliceUri.equals("/api")) {
            return SecurityConfig.createList(new String[]{"ROLE_APIUSER"});
        }

        // '/admin/member' 에서 /admin값을 이용해서, uri가 /admin 이고, grade:0 Menu GET
        Optional<Menu> menu = Optional.ofNullable(menuRepository.findByUriAndGrade(sliceUri, 0));

        //TODO 여기서 전부 Anonymous로 하면 login으로 분기되지않고 일단 uri 접근을 하게되서 컨트롤러단까지 타버림
        if (uriList.contains(uri) || menu.orElse(new Menu()).getUri() == null) {
            System.out.println("[URI value : " + sliceUri + "] 는 ROLE_ANONYMOUS의 권한을 가집니다");
            return SecurityConfig.createList(new String[]{"ROLE_ANONYMOUS"}); //menu 테이블에서 관리되는 권한이아닐경우 ROLE_ANONYMOUS 반환
        }

        List<String> roles;
        int subSlashPos = uri.indexOf("/", slashPos+1);

        if (subSlashPos > 0) { // URI slash depth 2 초과일경우
            subSliceUri = uri.substring(slashPos, subSlashPos);
            roles = searchAuthorityByMenu(menu, subSliceUri);
            return SecurityConfig.createList(roles.toArray(new String[roles.size()]));
        } else if(subSlashPos == -1){ // URI slash depth 2 일경우
            subSliceUri = uri.substring(slashPos);
            roles = searchAuthorityByMenu(menu, subSliceUri);
            return SecurityConfig.createList(roles.toArray(new String[roles.size()]));
        } else {
            roles = menu.get().getMenuAcl().stream().map(menuAcl -> menuAcl.getRole().getCode()).collect(Collectors.toList());
            return SecurityConfig.createList(roles.toArray(new String[roles.size()]));
        }

    }
```

위 코드는 심사시스템에서 사용하고있는 메뉴별 권한 대상을 관리하는 로직이다. 심사시스템에서는 2depth의 uri까지 권한 관리를 지원한다.

<br>

이와 관련된 블로그 포스팅은 [Spring Security에서 DB를 이용한 자원 접근 권한 설정 및 판단](https://zgundam.tistory.com/59)을 참고하자. <br>



<br><br>



**AffirmativeBased**의 경우[Spring Security의 자원 접근 판단에 대한 설명](https://zgundam.tistory.com/57) 포스팅을 참고하자.

