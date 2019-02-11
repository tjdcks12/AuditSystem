

## Pagination

페이징을 구현하기위해 Thymeleaf Spring data dialect를 사용했다. 해당 dialect는 페이징 구성 버튼 UI제공한다.



해당 dialect를 적용하기위해 maven dependency를 추가하고

```xml
<dependency>
	<groupId>io.github.jpenren</groupId>
	<artifactId>thymeleaf-spring-data-dialect</artifactId>
	<version>3.3.1</version>
</dependency>
```



SpringDataDialect객체를 빈주입한다.

```java
    @Bean
    public SpringDataDialect springDataDialect() {
        return new SpringDataDialect();
    }
```

<br><br>




### Thymeleaf

```html
<div class="ibox-content">
  <div class="ibox-tools">
    <a class="btn btn-primary btn-s new_ceremony" href="/admin/notice/write">+ 글쓰기</a>
  </div>
  <br>
  <table class="footable table table-stripped toggle-arrow-tiny text-center" id="notice_list_footable">
    <colgroup>
      <col width="15%" />
      <col width="*" />
      <col width="20%" />
      <col width="15%" />
      <col width="15%" />
    </colgroup>
    <tbody>
      <tr th:each="notice, iterStat : ${noticeList}" th:data="${notice.id}">
        <td th:text="${notice.id}"></td>
        <td class="text-left"><span th:text="${notice.title}" th:onclick="'viewNotice(\''+${notice.id}+'\')'"></span></td>
        <td th:text="${#dates.format(notice.updatedAt, 'yyyy-MM-dd HH:mm:ss')}"></td>
        <td>
          <th:block th:text="${notice.createUserInfo.getDecryptedName()}"></th:block>
          (
          <th:block th:text="${notice.createUserInfo.companyInfo.name}"></th:block>)
        </td>
        <td>
          <button class="btn-outline btn-default btn btn-xs" th:onclick="'viewNotice(\''+${notice.id}+'\')'">관리</button>
        </td>
      </tr>
    </tbody>
  </table>
    
  <!-- paging -->
  <div class="row">
    <div class="col-sm-5 col-sm-offset-7 ">
      <nav class="pull-right">
        <ul class="pagination" sd:pagination="full">
          <li class="disabled"><a href="#" aria-label="Previous"><span aria-hidden="true"></span></a></li>
          <li class="active"><a href="#">1 <span class="sr-only">(current)</span></a></li>
        </ul>
      </nav>
    </div>
  </div>
</div>
```

<sub>페이징이 적용된 html 소스</sub>

<br>

<br>



위 뷰는 다음과같은 구성을 통해 구현된다.





### Controller

```java
//공지 리스트
@RequestMapping(method = RequestMethod.GET)
@History
public String readNoticeList(Model model,   @RequestParam(required = false) String query, @PageableDefault(size = 10,  sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable){
    model.addAttribute("noticeList", noticeService.readNoticeByTitle(query, pageable));
    return "admin/notice";
}
```

<br><br>



RequestParam으로 Pageable 객체를 전달받는다. 해당 객체는  다음과 같은 필드값을 가진다.

```java

public interface Pageable {
    int getPageNumber();

    int getPageSize();

    int getOffset();

    Sort getSort();

    Pageable next();

    Pageable previousOrFirst();

    Pageable first();

    boolean hasPrevious();
}

```



또한 뷰에 전달하는 객체인 noticeList의 경우 Page객체인데 spring-data-dialect는 내부적으로 이 Page객체를 이용해서  페이징을위한 UI를 구성한다

```java
public Page<Notice> readNoticeByTitle(String query, Pageable pageable) {
    if(query != null && query.length() >0){
        return noticeRepository.findByStatusAndTitleContaining(noticeExposedStatus,query, pageable);
    }
    else{
        return noticeRepository.findNoticeByStatus(noticeExposedStatus, pageable);
    }
}
```

<sub>DAO단(Repository 객체)에서 Page를 자료형으로 반환을 받으며, 서비스단에서도 해당객체를 그대로 반환하고있다</sub>





```java
public interface Page<T> extends Slice<T> {
    int getTotalPages();

    long getTotalElements();

    <S> Page<S> map(Converter<? super T, ? extends S> var1);
}

```

<sub>페이징 UI 구성에 사용하기위한 totalPages, totalElement등을 필드값으로 갖고있다.</sub>



<br><br>



![img](https://raw.githubusercontent.com/tjdcks12/AuditSystem/master/images/pagination1.png)



<br>

기술 참고 : https://github.com/jpenren/thymeleaf-spring-data-dialect
