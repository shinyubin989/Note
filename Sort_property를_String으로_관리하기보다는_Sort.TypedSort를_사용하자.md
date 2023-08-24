### 목차
1. [해당 프로젝트 주소](#해당-프로젝트)
2. [요구사항](#요구사항)
3. [문제상황](#문제상황)
4. [해결방식](#해결방식)
5. [무슨 효과를 얻었나?](#무슨-효과를-얻었나?)

<hr>

<br><br>

### 해당 프로젝트

https://github.com/Team-BuddyCon/BACKEND_V2

### 요구사항

- 기프티콘을  **정렬(유효기간 순, 생성일 순, 이름 순)** 해서 볼 수 있어야 한다는 기획 요구사항이 존재

### 문제상황
- GifticonEntity
<img src="https://github.com/shinyubin989/note/assets/69676101/95a13429-ae68-4dd7-aec7-d8fe485a06e6" width=75%/>   

- 정렬 타입을 명시해놓은 Enum 클래스
<img src="https://github.com/shinyubin989/note/assets/69676101/e1fd4181-b810-4337-86e6-0c9981eb5e23" width=75% />

- 정렬 타입을 enum클래스안에 명시해놓았고, **Sort 객체를 한곳(enum)에서 관리**하여 데이터들간의 연관관계를 표현함
- 하지만 **property를 String으로 관리**하고 있었고(`Sort.by(”expireDate”)`), GifticonEntity의 **필드명이 수정되었을때 컴파일타임에 에러를 잡을 수 없었음**
    - 예를들어, Entity의 expireDate 필드를 expireTime으로 수정하고, 이후 enum을 수정하지 않은 상태에서 expireTime 기준으로 정렬 요청을 보낸다면, 에러가 뜰 것임
- 따라서 String을 대체할 방식을 찾아야했음

### 해결방식

- 처음에는 String대신 meta model 클래스(querydsl의 Qclass등..)를 집어넣어서 해결할 수 없을까.. 생각했는데 Sort.by()는 String을 매개변수로 받기때문에 사용할 수 없었음
- 어떤 방식이 있을까 찾아보던중 Sort클래스 내에 TypedSort클래스를 발견했고, 이를 통해 해결함

![image](https://github.com/shinyubin989/note/assets/69676101/656ff87f-e982-4da5-ba2c-723ac726e6f0)


- `Sort.sort()` 메서드 인자에 클래스를 명시하면 TypedSort객체가 반환되고, `TypedSort.by()` 메서드 인자에 `Function<T, S> property` 타입 인자를 넣어줌

### 무슨 효과를 얻었나?

- Sort property를 String으로 관리할 경우 **Entity 필드에 암묵적으로 의존**하게되며, 이때 직접적인 연관관계는 없기때문에 엔티티 필드명이 수정될경우 추후 **런타임에 에러가 발생**하게됨
- 이를 TypedSort로 대체하면서, Entity 필드 명이 수정된후 enum 내부를 수정하지 않았다고 한다면 **컴파일 에러로 문제를 잡을 수 있음**

- 또한 Entity내 Embedded클래스의 필드로 정렬을 요청하고자 할 때
    - String을 사용할때 → `Sort.by(”embeddedClass.fieldName”);` 처럼 entity 내의 embedded 클래스의 이름에 `.` 을 붙이고 필드명을 적어줘야함
    - TypedSort를 사용할때 → 단순하게 `Sort.sort(Entity.class).by(Entity::getFieldName)` 와 같이 getter메소드로 embedded클래스의 필드에 접근할 수 있음
