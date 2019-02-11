## 게임위 API 연동



```java
@Configuration
@EnableResourceServer
public class CustomResourceServerConfiguration extends ResourceServerConfigurerAdapter {
    @Override
    public void configure(HttpSecurity http) throws Exception {
        // validate api 관련 token 확인
        http.requestMatcher(new AntPathRequestMatcherWrapper("/api/**") {
            @Override
            protected boolean precondition(HttpServletRequest request) {
                return true;
            }
        }).authorizeRequests().anyRequest().authenticated();
    }
}

```

내부 인증수단과 별도로 게임위와의 API 연동을 위한 OAuth 인증수단이 요구되어, 등급심사시스템 자체가 ResourceServer와 Authorization 서버로서 게임위에게 데이터를 제공해준다.



- ### OAuth란 ?

  - 애플리케이션 (구글, 페이스북 등) 의 유저 비밀번호를 3rd party(그외 외부 서비스) 앱에 제공하지 않고도 인증 및 권한 부여를 관리할 수 있도록 하는 프레임워크

  - OAuth 기반 서비스의 API를 호출할 때는 헤더에 access token을 포함하여 요청해야 한다.

    

- ### OAuth 2.0 인증 방식 

  ### ![img](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/api1.jpg)



- Resource Owner

  - 사용자

- Client

  - 사용자가 사용하는 앱(소프트웨어 등)

- AuthorizationServer

  - 인증서버

- ResourceServer

  - REST API 서버

    

- **자체등급분류 서버는 Resource Server와 Authorization Server 역할을 동시에 함**

- Grant_Type 

  - #### Resource Owner Password Credentials Grant

    - 유저가 client에 id/password를 입력하여 access token을 직접 받아오는 방식

    - 신뢰할 수 있는 Client 경우에 주로 사용하는 방식이며, 게임위에서 가이드한 방식

    - 동작방식

      ![img](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/api2.png)



- **Dependency 설정**

- ```xml
  <dependency>     
  	<groupId>org.springframework.security.oauth</groupId>   
      <artifactId>spring-security-oauth2</artifactId>
  </dependency>
  ```

  `<dependency>     <groupId>org.springframework.security.oauth</groupId>     <artifactId>spring-security-oauth2</artifactId> </dependency>`

  

- AuthorizationServerConfiguration

  - 인증서버 관련 config 설정
  - token 정보 저장을 위한 jdbcToken 설정
  - client 정보 저장을 위한 ClientDetailsService 설정

- ResourceServerConfiguration
  - REST API 관련 config 설정
  - header token값 확인

- OAuth2 관련하여 DB사용을 위해서는 아래 스키마를 이용해 테이블 추가
  - 커스텀하여 DB table 사용도 가능함
  - <https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql>



**참고 : https://spring.io/guides/tutorials/spring-boot-oauth2/**