---
layout: post
title: Dynamic Form Submit
date: 2024-01-23 07:20:00
description: 스프링 부트와 타임리프를 이용한 동적인 폼 요소 제출
tags: java spring-boot thymeleaf
categories: develop
giscus_comments: true
featured: true
---

## 1. Opening

스프링 부트와 타임리프를 이용한 동적인 폼 요소 제출 방식을 알아보도록 합니다.  
이 글을 읽기 전에 아래 글을 참고해주세요.

- [Dynamic Form Search with Spring Boot + Thymeleaf](https://jaystar13.github.io/blog/2024/dynamic-form/)
- [Dynamic Form Search - Refactoring](https://jaystar13.github.io/blog/2024/dynamic-form-refactoring/)

### 조건 및 요구사항

- 페이지에 다양한 레이아웃의 폼이 다수 구성되어 있습니다.
- 폼 제출(`submit`)시 페이지의 모든 폼은 같은 트랙잭션으로 처리됩니다.

### 해결계획

- 입력한 폼 값을 전달받을 수 있도록 submit 전용 DTO를 만들고 Controller는 해당 DTO에 값을 매핑합니다.
- html의 name속성을 이용하여 각각 폼 요소의 값을 DTO에 매핑하도록 합니다.
- 기존 `MyFormService`를 활용하여 저장 메서드를 추가합니다.

### Spec

java17, spring boot 3.x, spring-mvc, thymeleaf

## 2. Implementation

본격적인 구현전에 기존 소스를 조금 리팩터링 하도록 하겠습니다.  
조회와 제출을 같이 하려다 보니 섹션에 대한 정의(또는 설정)가 필요하게 되어 Enum 객체를 만들 필요게 되었습니다.

### 2-1. SectionItem 생성

해당 Enum은 각 섹션에 대한 key-name 과 bean-name을 관리하도록 합니다.

```java
@Getter
public enum SectionItem {
    PROFILE("profile", "myProfileFormService"),
    ORDER("order", "myOrderFormService");

    private final String key;
    private final String beanName;

    SectionItem(String key, String beanName) {
        this.key = key;
        this.beanName = beanName;
    }
}
```

해당 클래스는 어플리케이션 전반적으로 사용되기 때문에 사용법은 구현 코드를 참조해주세요.

#### 2-2. submit 전용 DTO 생성

폼 제출시 입력 값 매핑을 담당할 DTO 객체를 생성합니다.

```java
@Getter
@Setter
public class MySectionsSubmit {

    private List<MyProfileForm> profiles;

    private List<MyOrderForm> orders;

}
```

저희는 현재 profile과 order 두가지의 폼 양식이 있기 때문에 이 두가지만 정의하였습니다.  
만일 폼 양식이 더 늘어나게 된다면 여기 DTO에 추가를 해야하는 구조입니다.

---

spring-mvc는 클라이언트에서 입력한 input 값을 전달받기 위해 다양한 방식의 기능을 제공하고 있습니다.  
html의 name속성에서 정의한 값을 기준으로 우리가 만든 객체에 매핑을 자동으로 도와주는 기능도 있습니다.  
java bean property를 사용하기 때문에 name속성 값과 DTO에서 정의한 필드 이름을 맞춰주어야 합니다.  
(사실 정확히 말하자면 필드 이름이 아닌 getter/setter 메서드 이름이지만 통상적으로 필드이름과 getter/setter이름을 맞추기 때문에 필드 이름이라고 말씀드렸습니다.)

#### 2-3. html name속성 정의

기존 단순 문자출력 항목을 input 타입으로 변경하고 name속성을 정의하도록 합니다.

```html
(...)
<table th:fragment="section(profile)">
  <tr th:each="form, formStat : ${profile}">
    <td>
      <table>
        <tr>
          <td>
            <span>Name:</span>
            <input type="text" th:value="${form.name}" th:name="|profiles[${formStat.index}].name|" />
          </td>
        </tr>
        <tr>
          <td>
            <span>Phone Number:</span>
            <input type="text" th:value="${form.phoneNumber}" th:name="|profiles[${formStat.index}].phoneNumber|" />
          </td>
        </tr>
      </table>
    </td>
  </tr>
</table>
(...)
```

타임리프의 `th:name`을 이용하여 값을 정의하였습니다.

`order.html`도 위와 동일하게 변경하도록 합니다.

그리고 우리는 폼을 제출해야 하기 때문에 `dynamicForm.html` 파일도 수정을 해야합니다.

```html
(...)
<form name="dynamicForm" action="/forms" method="post">
  <table>
    <tr th:each="section, sectionStat : ${sections.sections}">
      <td>
        <span th:text="|Section-${section.sectionItem.key}"></span>
        <table th:replace="~{|fragment/${section.sectionItem.key}| :: section(${section.myForms.myForms})}"></table>
      </td>
    </tr>
  </table>
  <button type="submit">Submit</button>
</form>
(...)
```

#### 2-4. Controller 수정

이제 폼 제출을 Controller에서 받을 수 있도록 해봅시다.

```java
    (...)
    @PostMapping("/forms")
    public String submitDynamicForms(@ModelAttribute("sections") MySectionsSubmit submit) {

        service.saveForms(getSectionItems(), submit);

        return "redirect:/forms";
    }

    private List<SectionItem> getSectionItems() {
        return Arrays.asList(SectionItem.PROFILE, SectionItem.ORDER);
    }
```

POST방식의 요청이 왔을 때 처리하도록 메서드를 추가하였습니다.  
위에서 말씀드렸던데로 spring-mvc의 `@ModelAttribute`를 사용하여 `MySectionsSubmit` DTO와 매핑하도록 하였습니다.  
`@ModelAttribute`의 name속성을 `sections`로 정의한 것은 양식 제출전 해당 페이지의 값 전달에 `sections` 이름이 사용되었기 때문입니다.

그리고 `Service`에도 저장을 할 수 있도록 메서드를 추가 해야겠지요?

### 2-5. Service 수정

**DynamicFormSearchService**

```java
    (...)
    public void saveForms(List<SectionItem> sectionItems, MySectionsSubmit mySections) {

        for (SectionItem sectionItem : sectionItems) {
            MyFormService myFormService = myFormFactory.getInstance(sectionItem.getBeanName());
            myFormService.saveMySection(mySections);
        }

    }

```

우리 예제에서는 data관련 라이브러리 의존을 사용하고 있지 않아 위 메서드는 Transaction이 명시되어 있지 않지만  
`@Transaction`을 사용하면 전파기능 동작으로 단일 트랙잭션으로 묶을 수 있을 것 같습니다.

**MyFormService**

```java
@Service
public interface MyFormService {
    MySection getMySection();

    Integer saveMySection(MySectionsSubmit mySection);
}
```

**MyProfileFormService**

```java
@Service
    (...)
    @Override
    public Integer saveMySection(MySectionsSubmit mySection) {
        List<MyProfileForm> profiles = mySection.getProfiles();
        //저장로직 구현

        return null;
    }
```

이렇게 수정을 해서 동적인 폼에 대한 제출 방법 구현도 마무리 하도록 하겠습니다.

## 3. Closing

동적인 폼 양식 처리를 위한 조회 및 제출 방법에 대해  
몇개의 글을 통하여 알아보았습니다.

사실 동적인 폼 양식이라면 Frontend가 이것보다는 훨씬 더 복잡할 것입니다.  
하지만 어떤 복잡한 화면이라도 클라이언트의 입력 값을 표준 방식으로 Backend에 전달하도록 규약을 정의하고  
Backend에서는 이 전달 받은 값을 효율적으로 처리할 수 있는 구조로 설계를 한다면  
**성능에 이점을 가져올 수 있고 운영과 유지보수 또한 비교적 손쉽게 가능할 것**으로 생각됩니다.

제가 해당글을 통해서 전달하고자 했던 내용이 사실 이 부분이기도 합니다.

다른 좋은 아이디어가 있다면 논의해 보는것도 즐거울 것 같습니다.

위 소스는 [깃헙](https://github.com/jaystar13/blog-code/tree/master/_2024-01-21-dynamic-form-submit)에서 확인할 수 있습니다.
