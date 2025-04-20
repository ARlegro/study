# [1편 : @NotNull vs @Column(nullabe=false) ]
> 1편 : 2개의 애노테이션 비교<br>
> 2편 : @Column을 쓰는 이유

JPA를 이용해서 스프링 프로젝트를 할 때,<br>
특정 필드의 null값을 허용하지 않는 방법에서 `@NotNull`과 `@Column(nullabe=false)`가 있다는 것을 알고 있었지만<br>
2개의 큰 차이를 이해하지 못하고 있다는 것을 느껴 좀 더 자세히 알아보기로 했다.

언뜻 보기에는  `@NotNull`과 `@Column(nullabe=false)`가 비슷해 보이고<br>
실제로 JPA 엔티티에 사용될 경우 둘 다 NOT NULL 제약조건을 추가해주기 때문에 뭐가 다르지라고 느낄 수 있다.

이 둘의 차이를 알아볼 것!

### 실험 대상 엔티티

 **기본 엔티티**
```kotlin
@Entity
public class Item {

    @Id
    @GeneratedValue(strategy = UUID)
    private UUID id;

    private String name;
}
```
위의 클래스를 기반으로 테스트를 해볼 것이다.

## `@NotNull` 적용 시

### `@NotNull` 기본 개념

- `@NotNull`은 `Bean Validation` 명세에 정의된 어노테이션이다.
- 엔티티뿐만 아니라, DTO 같은 일반 객체에도 적용된다.
- `@Column(nullabe=false)`와 달리, 엔티티뿐만 아니라 어떠한 빈이나, 클래스(ex. DTO)에도 사용될 수 있다.

### 엔티티 세팅 및 테스트

```kotlin
@Entity
public class Item {
    @Id
    @GeneratedValue(strategy = UUID)
    private UUID id;

    @NotNull
    private String name;
}
```

```kotlin
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
public class ItemNullTest {

  @Autowired
  private ItemRepository itemRepository;

  @DisplayName("@NotNull 테스트")
  @Test
  void notNullTest() {
    itemRepository.save(new Item());
  }
}
```

### **결과 확인**

**✅ 테이블 생성 쿼리**

```kotlin
    create table item (
        id uuid not null,
        name varchar(255) not null,
        primary key (id)
    )
Hibernate: 
    create table item (
        id uuid not null,
        name varchar(255) not null,
        primary key (id)
    )
```

테이블 생성시에 하이버네이트가 자동으로 `@NotNull`이 붙은 필드를 대상으로 `not null` 제약조건을 넣어준다<br>
그 이유는, Hiberanate가 내부적으로 아래의 설정을 통해 Bean Validation 어노테이션을 **DDL 메타데이터로 자동 변환하는 기능** 때문이다.
- 활성화(자동)
  - ```kotlin
    spring.jpa.properties.hibernate.validator.apply_to_ddl=true (기본값)
    ```

- **참고 : 비활성화 방법 (기본값 : true)**

    ```kotlin
    spring.jpa.properties.hibernate.validator.apply_to_ddl=false
    ```


**✅Insert 쿼리 확인**

그렇다면 null값을 넣고 save()메서드 호출 뒤에는 어떻게 될까??

not null 제약 조건을 위배했기 때문에 예상대로 오류가 발생했는데,

> **Insert 쿼리는 발생하지 않았다.**


오류 메시지를 보자

**✅ 오류 메시지**

ConstraintViolationException:

![image](https://github.com/user-attachments/assets/48c10746-5afb-4a66-ae84-86a9ba9b7460)


jakarta.validation.ConstraintViolationException

```kotlin
jakarta.validation.ConstraintViolationException: 
Validation failed for classes [com.example.nullable_test.Item] 
during persist time for groups [jakarta.validation.groups.Default, ]

List of constraint violations:[
	ConstraintViolationImpl{interpolatedMessage='널이어서는 안됩니다', 
	propertyPath=name, rootBeanClass=class com.example.nullable_test.Item, 
	messageTemplate='{jakarta.validation.constraints.NotNull.message}'}
]
```

엔티티를 저장하는 insert쿼리는 나가지 않고 ConstraintViolationException예외를 터트렸다.

즉, @NotNull을 사용하면 하이버네이트는 실제 SQL 쿼리(Insert)를 실행하기 전 단계인 pre-persist 단계에서 Bean Validation을 수행하며 예외를 터트린다는 것이다.

이는, 검증을 DB에서 하는 것이 아니라 스프링 차원에서 하는 것이다.

## @Column(nullable = false) 적용 시

### @Column 기본 개념

- JPA 명세에 포함된 애노테이션이다.
- 실제 데이터베이스 열의 특정 특성을 나타내기 위해 사용
- 따라서, DTO같은 다른 일반 클래스에는 사용할 수 없고, 엔티티에만 적용이 가능하다
- 목적 : 주로, DDL 스키마 생성 시 사용된다.
- 이 애노테이션을 생략하면 @Column의 기본값이 적용된다.(예외는 있는데 일단은 딥하게 X)

### 엔티티 세팅 및 테스트

```kotlin
@Data
@Entity
public class Item {

  @Id
  @GeneratedValue(strategy = UUID)
  private UUID id;

  @Column(nullable = false) // 기본값 : true
  private String name;
```

```kotlin

@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
public class ItemNullTest {
  
  @Autowired
  private TestEntityManager tem;

  @DisplayName("@Column(nullable=false) 테스트")
  @Test
  void nullableTest() {
    tem.persistAndFlush(new Item());
  }
```

### 결과 확인

✅ **테이블 생성 쿼리**

```kotlin
2025-04-20T19:22:38.752+09:00 DEBUG 36528 
    create table item (
        id uuid not null,
        name varchar(255) not null,
        primary key (id)
    )
Hibernate: 
    create table item (
        id uuid not null,
        name varchar(255) not null,
        primary key (id)
    )
```

@NotNull의 경우와 마찬가지로, 테이블 생성시에 하이버네이트가 자동으로 not null 제약조건을 넣어준다

**✅ Insert 쿼리 확인**

❓그렇다면 null값을 넣고 save()메서드 호출 뒤에는 어떻게 될까❓

- not null 제약 조건을 위배했기 때문에 예상대로 오류가 발생했다.
- 하지만 이번에는 **오류 메시지가 다르고, Insert 쿼리도 발생**했다.

**Insert쿼리**

```kotlin
2025-04-20T19:22:39.523+09:00 DEBUG 36528 --- [           main] org.hibernate.SQL                        : 
    insert 
    into
        item
        (name, id) 
    values
        (?, ?)
Hibernate: 
    insert 
    into
        item
        (name, id) 
    values
        (?, ?)
```

**✅ 오류 메시지 :**

오류 메시지를 보자

Caused by: org.postgresql.util.PSQLException: → org.hibernate.exception.ConstraintViolationException

![image](https://github.com/user-attachments/assets/abd5341a-40e2-432e-ac01-d8c65e8bc658)


@NotNull에서는 **ConstraintViolationException**예외만 터졌었는데,

이번 경우에는 **ConstraintViolationException + PSQLException** 오류 메시지가 떴다.

```java
org.hibernate.exception.**ConstraintViolationException**: could not execute statement [오류: "name" 칼럼(해당 릴레이션 "item")의 null 값이 not null 제약조건을 위반했습니다.
  Detail: 실패한 자료: (d069f405-8c04-453f-a88c-39ba797b3f9a, null)] [insert into item (name,id) values (?,?)]

	at org.hibernate.exception.internal.SQLStateConversionDelegate.convert(SQLStateConversionDelegate.java:97)
	at org.hibernate.exception.internal.StandardSQLExceptionConverter.convert(StandardSQLExceptionConverter.java:58)
	.....
Caused by: org.postgresql.util.**PSQLException**: 오류: "name" 칼럼(해당 릴레이션 "item")의 null 값이 not null 제약조건을 위반했습니다.
  Detail: 실패한 자료: (d069f405-8c04-453f-a88c-39ba797b3f9a, null)
	at org.postgresql.core.v3.QueryExecutorImpl.receiveErrorResponse(QueryExecutorImpl.java:2733)
	.....
```

💡 **참고 : DB 접근 전 유효성 검사 시키는 방법**

아래와 같이 설정하면 @Column 방법이라도 SQL이 나가기전에 예외 처리된다.

```java
spring.jpa.properties.hibernate.check_nullability=true
```

![image](https://github.com/user-attachments/assets/9471bda5-ccf1-4f79-a0e6-893acb544938)



### @Column 분석 정리

`@Column(nullabe = false)` 를 사용하는 경우에는 SQL 쿼리가 실제 실행되었고 DB에서 예외가 발생했다.

# 종합 결론

`@NotNull과 @Column(nullabe = false)` 모두 DB에 null값이 들어가는 것을 방지해주는 효과가 있지만

약간의 목적과 과정이 달랐다.

- `@NotNull` : Bean Validation에 기반해, SQL 실행 전 애플리케이션 레벨에서 검증 수행
- `@Column(nullabe = false)` : DDL 스키마에만 영향을 줌. 따라서 SQL 실행 후 검증은 DB에서 수행

**✅ 검증 시점의 관점으로 볼 때 `@NotNull` 승**

이번 내용만 보면, DB에 접근하기전에 검증이 일어나는 것이 더 좋으므로 `@NotNull`을 더 선호해야하는 게 맞다. (하지만 실제론 약간 다름)

실제론 왜 @Column을 쓰는 경우도 많은 데 그 이유는 나중에 볼 것

# [2편] 그럼에도 @Column(nullable=false)을 쓰는 이유

1편에서 말했던 내용대로라면, @Column보다 @NotNull이 훨씬 좋은데 왜 @Column을 쓰는 것일까?

그 이유를 알아보자

> 주장 :  Entity에는 검증을 최소화하고, DTO에서만 유효성 검사를 해야 한다
>
- 이 말은 주로, DDD(도메인 주도 개발) 관점에서 아주 유효한 주장이다.
- 일반적으로 스프링 MVC 아키텍쳐를 설계할 때 검증은 컨트롤러의 DTO에서 검증을한다.

```kotlin
// DTO
public class ItemDTO {
    @NotNull
    private String name;
}

// Entity
public class Item {
    @Column(nullable = false)
    private String name;
}
```

❗근데 엔티티를 생성, 수정 등을 하는 Service계층에서 또 검증을 하면 ‘오버 검증’이라고 생각하기 때문

- 이런 오버 검증은 코드 유지보수가 어렵다는 문제도 있다.

### **@Column(nullable=false) 장점**

**1️⃣ 명시적 (추론력 상승)** ⭐⭐⭐

- 혼자 or 팀원들과 코드 리뷰를 할 때 이 코드만 보고도 DB가 어떤 제약조건을 가진지 예상할 수 있다.
- 보통 스키마는 애플리케이션 코드 작성하는 부분과 다른 위치에 있으므로, 따로 찾아봐야 할텐데, 이렇게 명시하면 찾아보지 않아도 된다.

**2️⃣ Hibernate 의존성 감소** ⭐⭐

- Hibernate가 DDL을 자동으로 만들어주지 않더라도 nullabe=false를 하면 여전히 NotNull 제약이 추가된다.
- 만약, Hibernate가 아닌 다른 ORM 기술 or 다른 접근 기술을 사용시 유연하게 대응할 수 있다,.

**3️⃣ 다른 도구와의 호환 ⭐**

`@Column(nullable = false)` 보고 스키마 자동 생성, ERD 뷰 만들어주는 도구들이 있는데, 이 코드가 없으면 툴 자동화 연동이 깨지는 문제가 발생한다.

### 운영 단계에서 지침

참고 : 일반적으로 **운영 및 배포 단계**에서는 Hibernate DDL 생성 기능을 안 쓴다.

대신, 직접 DDL 스크립트를 수동으로 작성해서 실행시키게 한다.

그렇기 때문에 운영 및 배포할 때 `@Column(nullabe=false)`를 같이, 코드에 제약 조건을 표현하는 것이 필요없다고 느낄 수 있다.

> 하지만 앞서 말했던, **@Column의 장점**들을 살리기 위해 개발자 간 암묵적 계약으로**@Column(nullable=false) 를 표현해 두는 것이 좋은 것 같다.**
>
