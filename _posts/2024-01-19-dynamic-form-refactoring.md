---
layout: post
title:  Dynamic Form Search - Refactoring
date:   2024-01-19 15:30:16
description: 스프링 부트와 타임리프를 이용한 동적인 폼 요소 조회 - 리팩토링
tags: java spring-boot thymeleaf refactoring
categories: develop
giscus_comments: true
featured: true
---
## Issue
[이전글](https://jaystar13.github.io/blog/2024/dynamic-form/)에서 구현했던 코드를 리팩터링 해보도록 하겠습니다.

## Condition
구현을 위한 특별한 조건은 없습니다.  

다만, [일급 컬렉션](https://jojoldu.tistory.com/412)을 사용하기 때문에 이에 대한 지식이 필요할 수 있습니다.  

## Plan
- 컨트롤러의 forms타입(`List<List<MyForm>>`)을 일급 컬렉션으로 변경하도록 합니다.  
- html의 타입별 if문 분기를 파일로 따로 뺄 수 있도록 수정합니다.  

## Spec
java17, spring boot 3.x, spring-mvc, thymeleaf

## Implementation
컨트롤러의 `List<List<MyForm>> forms = service.getForms(formNames)` 이 부분을 보면  
리스트 안에 다시 리스트가 들어있는 조금은 복잡한 구조를 가집니다.  

먼저 `List<MyForm>`을 DTO클래스로 변경하는 작업을 하겠습니다.  

### 1. MySection 구현
처음엔 일급 컬렉션 객체로 만들려 했지만 후술하게될 html분기문 리팩터링을 위해서  
Section의 고유 이름이 필요하여 멤버 변수가 두개인 DTO객체로 구현하게 되었습니다.  

```java
@Getter
@Setter
public class MySection {
    private String sectionName;

    private List<MyForm> mySection;

    public MySection(String sectionName, List<MyForm> mySection) {
        this.sectionName = sectionName;
        this.mySection = mySection;
    }
}
```

`sectionName`은 해당 섹션에 고유 이름을 부여하였고  
`mySection`으로 섹션을 구성하는 form을 참고하도록 하였습니다.  

그리고 기존에 `List<MyForm>`을 사용하는 곳을 이 `MySection`로 변경을 합니다.  

#### 1-1. MyFormService 인터페이스 및 구현체
```java
@Service
public interface MyFormService {
    MySection getMySection();
}  
```
메서드 이름도 하는일이 좀 더 명확하도록 변경하였습니다.  

```java
@Service
public class MyOrderFormService implements MyFormService {
    @Override
    public MySection getMySection() {
        return new MySection("order", Arrays.asList(new MyOrderForm("clean code", 2),
                new MyOrderForm("candy", 10)));
    }
}
```  
주문 서비스에서는 `order`라는 이름으로 `MySection`을 리턴하였습니다.  

```java
@Service
public class MyProfileFormService implements MyFormService {
    @Override
    public MySection getMySection() {
        return new MySection("profile", List.of(new MyProfileForm("홍길동", "012-3456-7890")));
    }
}
```  
프로필 서비스에서는 `profile`라는 이름으로 `MySection`을 리턴하였습니다.  

#### 1-2. MySections DTO 생성
서비스에서 조회한 `Section`을 리스트 형태로 가지고 있는 `MySections`를 생성하겠습니다.  
이 `MySections`는 화면으로 전달되어 값을 표현하는데 사용할 수 있습니다.

```java
@Getter
@Setter
public class MySections {
    private List<MySection> sections;

    public MySections() {
        this.sections = new ArrayList<>();
    }

    public boolean add(MySection mySection) {
        return sections.add(mySection);
    }
}
```

`MySections`는 일급 컬렉션객체 입니다.

인터페이스의 메서드 명과 리턴타입을 변경하였으니  
여기저기 컴파일 에러가 발생합니다.  

이제 관련된 부분을 수정해 보겠습니다.

#### DynamicFormSearchService
```java
@RequiredArgsConstructor
@Service
public class DynamicFormSearchService {

    private final MyFormFactory myFormFactory;

    public MySections getForms(String[] serviceNames) {
        MySections sections = new MySections();
        for (String serviceName : serviceNames) {
            MyFormService myFormService = myFormFactory.getInstance(serviceName);
            MySection mySection = myFormService.getMySection();
            sections.add(mySection);
        }

        return sections;
    }
}
```

#### DynamicFormSearchController
```java
@RequiredArgsConstructor
@Controller
public class DynamicFormSearchController {

    private final DynamicFormSearchService service;

    @GetMapping("/forms")
    public String getDynamicForms(Model model) {
        String[] formNames = {"myProfileFormService", "myOrderFormService"};

        MySections sections = service.getForms(formNames);

        model.addAttribute("sections", sections);

        return "dynamicForm";
    }
}
```  
모델에 전달하는 key 명칭을 `sections`로 변경하였습니다.

#### dynamicForm
구조가 변경되었기 때문에 html도 수정을 해줘야 합니다.  
```html
(...)
<table>
    <tr th:each="forms, formsStat : ${sections.sections}">
        <td>
            <span th:text="|Form-${formsStat.index}"></span>
            <table>
                <tr th:each="form, formStat : ${forms.mySection}">
                    <td th:if="${form instanceof T(com.jaystar.dto.MyProfileForm)}">
                        <table>
                            <tr>
                                <td>
                                    <span>Name:</span>
                                    <span th:text="${form.name}"></span>
                                </td>
                            </tr>
                            <tr>
                                <td>
                                    <span>Phone Number:</span>
                                    <span th:text="${form.phoneNumber}"></span>
                                </td>
                            </tr>
                        </table>
                    </td>

                    <td th:if="${form instanceof T(com.jaystar.dto.MyOrderForm)}">
                        <table>
                            <tr>
                                <td>
                                    <span>Product Name:</span>
                                    <span th:text="${form.productName}"></span>
                                </td>
                            </tr>
                            <tr>
                                <td>
                                    <span>Quantity:</span>
                                    <span th:text="${form.quantity}"></span>
                                </td>
                            </tr>
                        </table>
                    </td>
                </tr>
            </table>
        </td>
    </tr>
</table>
(...)
```   
코드가 너무 기네요😭  

얼릉 리팩터링을 해봐야겠습니다.

### html분기문 리팩터링
dynamicForm.html 코드를 보면 `<td th:if="${form instanceof T(com.jaystar.dto.MyProfileForm)}">`와 같이 타입을 검사하여 분기처리를 하고 있습니다.  
이 부분은 thymeleaf가 제공하는 fragment를 이용하여 각각의 파일로 만들어 처리를 할 수 있을거 같습니다.  

thymeleaf의 fragment는 웹 페이지에서 공통으로 사용하는 파일의 재사용을 위한 기능을 제공합니다.  
이를 이용하여 우리는 각각 Section별로 별도의 html을 만들고 이를 호출하여 사용하도록 변경을 하면 좋을 것 같습니다.  

이것을 위해서 우리는 위에 `MySection`를 만들어 놓았습니다.   

#### fragment/profile.html 생성
프로필을 표현하기 위한 profile파일을 생성하였습니다.  
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<table th:fragment="section(profile)">
    <tr th:each="form, formStat : ${profile}">
        <td>
            <table>
                <tr>
                    <td>
                        <span>Name:</span>
                        <span th:text="${form.name}"></span>
                    </td>
                </tr>
                <tr>
                    <td>
                        <span>Phone Number:</span>
                        <span th:text="${form.phoneNumber}"></span>
                    </td>
                </tr>
            </table>
        </td>
    </tr>
</table>
</body>
</html>
```

#### fragment/order.html 생성
주문내역을 표현하기 위한 order파일도 생성하였습니다.  
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<table th:fragment="section(order)">
    <tr th:each="form, formStat : ${order}">
        <td>
            <table>
                <tr>
                    <td>
                        <span>Product Name:</span>
                        <span th:text="${form.productName}"></span>
                    </td>
                </tr>
                <tr>
                    <td>
                        <span>Quantity:</span>
                        <span th:text="${form.quantity}"></span>
                    </td>
                </tr>
            </table>
        </td>
    </tr>
</table>
</body>
</html>
```

#### dynamicForm.html 수정
그리고 각각 섹션파일을 불러오기 위해 `dynamicForm` 파일을 아래와 같이 수정하였습니다.
```html
(...)
<table>
    <tr th:each="forms, formsStat : ${sections.sections}">
        <td>
            <span th:text="|Section-${forms.sectionName}"></span>
            <table th:replace="~{|fragment/${forms.sectionName}| :: section(${forms.mySection})}"></table>
        </td>
    </tr>
</table>
(...)
```
처음에 비해 훨씬 깔끔해진 코드입니다.

## Conclusion
이제 우리는 테스트로 넣어 놓은 프로필, 주문 두개의 양식 이외에 다른 양식이 필요한 경우에도  
인터페이스를 구현한 서비스와 DTO, html정도만 추가하면  
기존 코드의 수정 없이 쉽게 처리할 수 있는 구조를 만들었습니다.  

다음엔 해당 양식을 제출하여 저장소까지 처리될 수 있는 구조를 알아보도록 하겠습니다.  

위 소스는 [깃헙](https://github.com/jaystar13/blog-code/tree/master/_2024-01-19-dynamic-form-refactor)에서 확인할 수 있습니다.