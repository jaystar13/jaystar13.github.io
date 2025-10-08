---
layout: post
title: Dynamic Form Search with Spring Boot + Thymeleaf
date: 2024-01-09 16:40:16
description: 스프링 부트와 타임리프를 이용한 동적인 폼 요소 조회
tags: java spring-boot thymeleaf
categories: develop
giscus_comments: true
featured: true
---

## Issue

Frontend를 개발하다 보면 form형식을 굉장히 자주 마주칩니다. 한 두개의 입력 항목만 기입하는 단순한 폼 형식부터
경력사항 입력과 같이 여러 폼 형식을 입력하고 제출하는 리스트 형식의 폼까지 다양한 UI의 폼을 볼 수 있습니다.  
React/VueJS와 같은 CSR 방식이 아닌 thymeleaf를 이용한 SSR 방식으로 다양한 폼에 대한 효율적인 구현을 고민해보고자 합니다.

![예상화면](/assets/img/2024-01-09-Figure-1.jpg)

## Condition

구현해야할 페이지는 다음과 같습니다.

- 페이지는 하나 또는 둘 이상의 그룹으로 나뉘어져 있습니다.(이 그룹을 섹션이라고 하겠습니다.)
- 섹션은 하나 또는 둘 이상의 폼으로 구성되어 있습니다.
- 섹션은 여러개가 존재할 수 있으며 페이지를 구성하는 섹션은 설정으로 관리하고 있습니다.

## Plan

- 반복되는 섹션과 폼을 위해서 상속과 인터페이스를 사용해야 할 것 같습니다.

- Factory Pattern을 사용해서 원하는 서비스를 호출하도록 합니다.

## Spec

java17, spring boot 3.x, spring-mvc, thymeleaf

## Implementation

- 조회  
  구현 전 Plan 단계에서 구상한 대로 인터페이스를 만들도록 하겠습니다.  
  `FormService`라고 이름지었고 코드는 아래처럼 단순합니다.

  ```java
  @Service
  public interface MyFormService {
      List<MyForm> getMyForms();
  }
  ```

  이 인터페이스를 구현한 객체는 `getMyForms`를 사용해서 자신의 Form 데이터인 `List<MyForm>` 타입을 리턴할 것입니다.  
  Form은 하나 이상 존재할 수 있기 때문에 List타입을 사용합니다.  
  `MyForm`은 모든 Form의 부모객체로 사용합니다.

  ```java
  public class MyForm {
  }
  ```

  이제 `MyFormService`인터페이스 구현체를 준비해 봅시다.  
  `MyProfileFormService` 라는 구현체는 값 전달을 위해 `MyProfileForm` 객체를 사용하는데요,
  이것은 `MyForm`을 상속하고 있습니다.

  코드는 아래와 같이 되겠네요.

  ```java
  @Service
  public class MyProfileFormService implements MyFormService {
      @Override
      public List<MyForm> getMyForms() {
          return List.of(new MyProfileForm("홍길동", "012-3456-7890"));
      }
  }
  ```

  구현체를 하나더 만들어 볼까요?
  `MyOrderFormService`는 나의 주문내역을 관리하는 서비스입니다.  
  여기서 구현한 `getMyForms` 에는 하나 이상의 주문 내역을 가지고 옵니다.

  마찬가지로 코드는 아래와 같습니다.

  ```java
  @Service
  public class MyOrderFormService implements MyFormService {
      @Override
      public List<MyForm> getMyForms() {
          return Arrays.asList(new MyOrderForm("clean code", 2),
                  new MyOrderForm("candy", 10));
      }
  }
  ```

  이제 이 서비스를 효율적으로 사용할 수 있는 방법을 생각해 봐야 합니다.

  `if`문을 사용해서 아래처럼 각 서비스를 호출할 수 있을거 같네요.

  ```java
  if (전달받은 서비스명.equals("profile")) {
      service = new MyProfileFormService();
  } else if (전달받은 서비스명.equals("order")) {
      service = new MyOrderFormService();
  }
  ```

  이렇게도 사용할 수 있겠지만,  
  너무나도 잘 아시다시피 이런 `if...else`구문은 우리를(또는 다른사람을) 너무 힘들게 합니다.

  이걸 해결하기 위해 저는 **Factory Pattern**이 떠올랐습니다!

  ```java
  @RequiredArgsConstructor
  @Component
  public class MyFormFactory {
      private final Map<String, MyFormService> formServiceMap;

      public MyFormService getInstance(String serviceKey) {
          return formServiceMap.get(serviceKey);
      }
  }

  ```

  `@Component`를 선언함으로써 Spring 프레임워크는 `formServiceMap` 안에  
  `MyFormService`를 구현한 `MyProfileFormService` `MyOrderFormService` 를 key, value의 형태로 주입(injection)할 것입니다.  
  `getInstance`메서드는 서비스키(=Bean Name)을 전달받아서 해당 인스턴스를 전달하는 역할을 하지요.

  이 Factory를 `Service`에서 아래처럼 호출하도록 했습니다.

  ```java
  @RequiredArgsConstructor
  @Service
  public class DynamicFormService {

      private final MyFormFactory myFormFactory;

      public List<List<MyForm>> getForms(String[] serviceNames) {
          List<List<MyForm>> sections = new ArrayList<>();
          for (String serviceName : serviceNames) {
              MyFormService myFormService = myFormFactory.getInstance(serviceName);
              List<MyForm> myForms = myFormService.getMyForms();
              sections.add(myForms);
          }

          return sections;
      }
  }
  ```

  이제 이 서비스를 호출하는 `Controller`를 고민해봐야 합니다.

  `Controller`는 `DynamicFormController`라고 명명하겠습니다.

  ```java
  @RequiredArgsConstructor
  @Controller
  public class DynamicFormController {

      private final DynamicFormService service;

      @GetMapping("/forms")
      public String getDynamicForms(Model model) {
          String[] formNames = {"myProfileFormService", "myOrderFormService"};

          List<List<MyForm>> forms = service.getForms(formNames);

          model.addAttribute("section", forms);

          return "dynamicForm";
      }
  }
  ```

  테스트를 위해 `formNames`는 값을 직접 할당하였지만 화면 또는 저장소에서 값을 전달받도록 할 수도 있겠습니다.  
  서비스의 `getForms`메서드를 호출한 결과를 `model`에 세팅하였습니다.

  이제 화면에서 이 값을 꺼내서 표현하기만 하면 될거 같네요.

  타임리프를 사용한 html구현 코드입니다.

  ```html
  <table>
    <tr th:each="forms, formsStat : ${section}">
      <td>
        <span th:text="|Form-${formsStat.index}|"></span>
        <table>
          <tr th:each="form, formStat : ${forms}">
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
  ```

  ![화면](/assets/img/2024-01-09-Figure-2.jpg)

  이렇게 해서 조회기능을 완성하였습니다.

## Conclusion

Frontend와 Backend의 역할이 분리된 CSR방식의 경우,  
각 폼의 데이터를 API형식으로 호출하여 사용할 수 있기 때문에 유연하게 처리가 가능합니다.  
하지만 SSR의 경우,  
화면 표시를 위해 데이터도 함께 가져와야 하기 때문에 여러가지 고민이 생기게 됩니다.  
이번글에서는 데이터 전달 객체를 위해 상속(`MyForm`)을 사용하고  
각 폼의 데이터 조회를 위한 서비스 호출을 위해 인터페이스(`MyFormService`)와 팩토리(`MyFormFactory`)를 사용하였습니다.

만들고 보니 몇가지 리팩터링 할 내용이 보이네요.

- 컨트롤러의 `forms`타입을 일급 컬렉션으로 변경할 수 있을거 같습니다.
- html의 타입별 if문 분기를 파일로 따로 뺄 수 있는 방법이 있을거 같네요.

[다음글](https://jaystar13.github.io/blog/2024/dynamic-form-refactoring/)에서는 리팩터링을 진행해 보도록 하겠습니다.

위 소스는 [깃헙](https://github.com/jaystar13/blog-code/tree/master/_2024-01-09-dynamic-form-search)에서 확인할 수 있습니다.
