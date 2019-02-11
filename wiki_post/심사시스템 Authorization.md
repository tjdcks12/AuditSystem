## Authorization



자체등급 분류 시스템에서는 권한리스트를 데이터베이스 테이블을 통해 관리한다. 

권한마다 접근가능한 메뉴가 세팅되고, 권한에따라 메뉴 노출 여부가 결정된다. <br>



![img](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/role1.png)

<sub>권한 부여 대상이되는 Member 테이블</sub>



<br><br>





![img](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/role2.png)
<sub>권한을 관리하는 Role 테이블</sub>

<br><br>

![img](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/role4.png)

<sub>메뉴를 관리하는 Menu 테이블</sub>

<br><br>

![img](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/role3.png)

<sub>Menu-Role 관계의 MenuAcl 테이블</sub>





이렇게 구성된 권한설정을 위한 테이블은 스프링 시큐리티의 SecurityMetadataSource를 구현한 객체에 의해 권한관리 대상이 된다. 해당 객체에서는

```java
public Collection<ConfigAttribute> getAttributes(Object o)
```

위의 메소드를 재정의하게되는데, 해당 메소드에서 

```java
Optional<Menu> menu = Optional.ofNullable(menuRepository.findByUriAndGrade(sliceUri, 0));

```

위와같이 Menu 테이블의 URI값과 현재 접근하려는 URI 를 비교 및 menuAcl에 존재하는 권한별 접근가능 메뉴를 탐색하여 최종적으로, 현재 URI에 접근할수있는 권한 리스트를 리턴`(Collection<ConfigAttribute> 자료형을 가지는) `하게된다.

<br>

<br>



```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    ...
    
    http.addFilterBefore(customFilterSecurityInterceptor(), FilterSecurityInterceptor.class); 
    
    ...
}

public FilterSecurityInterceptor customFilterSecurityInterceptor() {
    ...
      	filterSecurityInterceptor.setSecurityMetadataSource(filterInvocationMetadataSource());
    ...
    return filterSecurityInterceptor;
}
```

그리고 이러한 설정을 적용하기위해 **WebSecurityConfigurerAdapter** 구현체를 통해 설정정보를 filter에 추가한다.