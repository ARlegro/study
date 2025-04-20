> JPA는 4개월 전에 처음 배웠고, 드문드문 사용하면서 참고하는 정도로만 하다가,<br>
> 기억이 안 나는 것들이 많아져서 시간 날때 조금씩 복습하기로 했다.<br>
> 그러던 중 @ManyToOne의 여러 속성과 내부적으로 어떻게 구현이 되고 조건들이 넣어지는지 알아보고 싶어서 공부를 하기로 했다.

## ManyToOne 기본 구조

JPA가 구현한 manyToOne 인터페이스를 들어가보면 아래의 구조와 같다

```java
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ManyToOne {

  Class targetEntity() default void.class;

  CascadeType[] cascade() default {};

  FetchType fetch() default FetchType.EAGER;

  boolean optional() default true;
}
```
- targetEntity
- cascade
- fetch
- optional

**✅ 기본 값일 때의 예시**

아래처럼 @ManyToOne의 옵션을 사용하지 않을 시 구현된 default값이 적용되면 실행될 것이다.

```java
  @ManyToOne
  private Parent parent;
```

```postgresql
    create table child (
        id uuid not null,
        parent_id uuid,
        age varchar(255),
        primary key (id)   <<< id
    )
    
    alter table if exists child 
       add constraint FK7dag1cncltpyhoc2mbwka356h 
       foreign key (parent_id) 
       references parent    
```

![image](https://github.com/user-attachments/assets/05d8f699-96b9-42d0-b07a-2ff790deb836)


> **기본 값은 됐고, optional과 cascade 속성을 사용하는 방법에 대해 좀 더 알아볼 것**
>

# 1. optional 속성

**✅ 기본 개념**

```java
public class Child {

  @ManyToOne(optional = true)  // 기본값이다.
  private Parent parent;
}
```

- **선택적 : optional = true**

    - 자바 객체 관계상 이 연관관계의 필드는 **null이 허용**된다는 의미
    - 즉, Child 객체 생성 시 Parent가 없어도 된다는 것
    - 일반적으로 연관관계 있으면 null을 허용하지 않을 것이다.
    - 쿼리가 left outer join (이후 얘기)


- **필수적 : optional = false**
    - **연관 엔티티는 필수**.
    - `null`이면 예외 발생 (`PersistenceException`)
    - 조회 시 **INNER JOIN** 사용

## **optional에 따른 쿼리 차이**

optional의 속성값에 따라 조회 쿼리가 달라지는데, 이를 테스트해보면서 알아볼 것

공통으로 아래의 코드로 테스트를 진행할 것이다. 조회 시 쿼리가 어떻게 나가는지 보자

```java
  @Autowired
  private TestEntityManager tem;

  @Test
  void manyToOneOptionalTest() {
    Child child = new Child();
    tem.persistAndFlush(child);
    tem.clear();

    Child foundChild = tem.find(Child.class, child.getId());

  }
```

### **1️⃣ Case 1 : optional = true (기본값)**

> optional = true ⇒ 없어도 된다는 말

연관관계 조인 시 그 연관관계가 없을 수도 있다는 거니까 hibernate는 LEFT OUTER JOIN이 발생할 수 밖에 없다.

**💭(엔티티)**

```java
@Data
@Entity
public class Child {

  @ManyToOne
  private Parent parent;
```

**💭(테스트)**

```java
Child child = new Child();
tem.persistAndFlush(child);  // 저장
tem.clear();

Child found = tem.find(Child.class, child.getId());
```

**💭(쿼리)**

```sql
Hibernate: 
    select
        c1_0.id,
        c1_0.age,
        p1_0.id,
        p1_0.name 
    from
        child c1_0 
    **left join**
        parent p1_0 
            on p1_0.id=c1_0.parent_id 
    where
        c1_0.id=?
```

- LEFT JOIN 발생
- 이유 : 부모가 없을 수도 있기 때문

### **2️⃣ Case 2 : `optional = false` + 필드값이 null일 경우**

**💭(엔티티)**

```java
@Data
@Entity
public class Child {

  @ManyToOne(optional = false)
  private Parent parent;
```

**💭(테스트)**

```java
Child child = new Child();
tem.persistAndFlush(child); // 예외 발생 ❌
tem.clear();

Child found = tem.find(Child.class, child.getId());
```

**💭(예외 메시지)**

```java
org.hibernate.exception.ConstraintViolationException: could not execute statement [오류: "parent_id" 칼럼(해당 릴레이션 "child")의 null 값이 not null 제약조건을 위반했습니다.
  Detail: 실패한 자료: (15, d4415f11-314b-4124-8289-79d083906708, null)] [insert into child (age,parent_id,id) values (?,?,?)]
	....
Caused by: org.postgresql.util.PSQLException: 오류: "parent_id" 칼럼(해당 릴레이션 "child")의 null 값이 not null 제약조건을 위반했습니다.
```

parent 가 없다보니 위와 같이 DB에서 오류를 터트렸다.

### **3️⃣ Case 3 : `optional = false` + 필드값이 존재할 경우**

- 일단 parent 객체가 등장하므로 테스트 코드도 변경

**💭(엔티티)**

```java
@Data
@Entity
public class Child {

  @ManyToOne(optional = false)
  private Parent parent;
```

**💭(테스트)**

```java
  @Autowired
  private TestEntityManager tem;
  
  @Test
  @DisplayName("manyToOne-optional 필드값이 있는 경우")
  void manyToOneOptionalTest2() {
    Parent parent = new Parent();
    tem.persistAndFlush(parent);
    
    Child child = new Child();
    child.setParent(parent);
    tem.persistAndFlush(child);
    tem.clear();
    
		Child foundChild = tem.find(Child.class, child.getId());
    
```

**💭(로그)**

```java
    select
        c1_0.id,
        c1_0.age,
        c1_0.parent_id,
        p1_0.id,
        p1_0.name 
    from
        child c1_0 
    join
        parent p1_0 
            on p1_0.id=c1_0.parent_id 
    where
        c1_0.id=?
```

- INNER JOIN 발생 : 이번에는 innerJoin의 쿼리가 나왔다.
- 이유: 부모는 반드시 있어야 하므로

## optional = false 만으로는 못 하는 것

**✅ 하이버 네이트의 자동 추론**

- Hibernate는 optional=false 속성이 있으면 DDL 생성 시 그 필드에 NOT NULL 제약조건을 넣어준다.
- 이러한 좋은 기능으로 인해 ManyToOne(optional=false) 만 해도 된다고 느낄 것이다.

**하지만 만약 다른 JPA 구현체라면?**

- optional=false 만으로 DDL 에 반영이 안 될 수 있다.
- 이 때는 @JoinColumn(nullable = false) 을 사용해야 한다.
- 아래는 hibernate 간섭이 없을 떄 ManyToOne과 JoinColumn이 어떤 의미가 될지이다.

    ```kotlin
    **[Hibernate 간섭이 없는 경우]**
      @ManyToOne(optional = false)  // 자바 객체 상 null 금지 
      @JoinColumn(nullable = true)  // DB는 NULL 허용
      private Parent parent;
    ```

    - DDL 생성 시에는 nullable이 허용된다는 것


<aside>
💡

근데…. ManyToOne이 JPA인데 구현체가 바뀔 일이 거의 없어서 그냥 optional=false만 사용해도 문제 없을 것 같다.

</aside>

**✅ @JoinColumn(nullable = false) + optional = false**

Hibernate라는 좋은 기능을 쓰고 있다면 자동 추론으로 인해 optional 기능만 써도 DDL 생성 시 제약조건 문제가 없다.<br>
그럼에도 불구하고 동시에 쓰는 몇가지 이유가 있는데 그걸 알아보자 (안 써도 큰 문제는 없음)

1. **명확한 의미 전달**
2. **이식성**
    - hibernate가 아닌 다른 구현체, ORM을 쓴다면 코드 바꿔야 할 것

# 2. casecade

일반적으로 연관관계의 주인이 아닌 곳에서 cascade 설정만 가능한 줄 알았다.<br>
하지만, @ManyToOne의 cascade 속성도 있다는 것을 알아서 공부 좀 해보려 한다.

Cacade

- 연관관계들이 서로에 종속적인 것
- JPA에서는 Entity의 상태 변화를 전파시키는 옵션이다.
- 기본값 : 전파 X

ManyToOne의 cascade 옵션을 사용하는 경우를 예시를 통해 알아볼 것

### ☑️ Case 1. cascade.ALL

시나리오<br>
- `Child` → `Parent` 방향으로 `@ManyToOne(cascade = ALL)` 설정됨
- 테스트 : 부모 1, 자식 2을 `persist`, 그 후 자식 한 명을 `delete` 시도


**💴 엔티티**

```java
@Entity
public class Child {
	...
  @ManyToOne(cascade = ALL)
  private Parent parent;
}

@Entity
public class Parent {
	...
  private String name;
}
```

**💴 테스트**

```java
  @Test
  @DisplayName("ManyToOne에 cascade 걸린 경우")
  @Transactional
  @Rollback(value = false)
  void cascadeTest() {
    Parent parent = new Parent();
    tem.persistAndFlush(parent);

    Child child1 = new Child();
    Child child2 = new Child();
    child1.setParent(parent);
    child2.setParent(parent);

    tem.persistAndFlush(child1);
    tem.persistAndFlush(child2);
    tem.clear();

    childRepository.deleteById(child1.getId());
  }
```

**💴 에러 메시지**

![image](https://github.com/user-attachments/assets/78ce575e-d024-4979-897b-ec7b98b6289e)

에러가 떴다. DataIntegrityViolationException, PSQLException

대충 참조 키 제약조건을 위반했다는 소리이다.

**💢 에러 원인 분석**

- `@ManyToOne(cascade = ALL)`은 **자식이 persist될 때 부모도 persist되게 하는 것**이지, `delete` 시 반대방향 cascade가 적용되지 않음.
- 자식 제거 → 부모 제거 : 이 때, 이 부모를 참조하고 있는 다른 자식의 필드값이 NULL이 되어버리니까 DB 수준에서는 참조 무결성 제약조건을 위반하게 되어 예외가 터진 것

### ☑️ Case 2. cascade.ALL + orpan

시나리오

- 1번과 거의 동일
- 추가된 점 : `Parent`에 `@OneToMany(mappedBy = "parent", orphanRemoval = true)` 추가
- `orphanRemoval = true`
    - Hibernate가 `Parent`의 `children` 컬렉션을 추적하면서 고아 엔티티들을 자동 삭제해줌
    - 즉, 이 시나리오 상에서 Parent가 삭제되면 자식들은 모두 고아가 되므로 자동 삭제가 되어야 함

**💴 엔티티**

```java
@Entity
public class Child {
	...
  @ManyToOne(cascade = ALL)
  private Parent parent;
}

@Entity
public class Parent {
	...
  private String name;
  
  @OneToMany(mappedBy = "parent", orphanRemoval = true)
  private List<Child> children = new ArrayList<>();
}
```

**💴 테스트**

```java
  @Test
  @DisplayName("ManyToOne에 cascade 걸린 경우 + orphanRemoval")
  @Transactional
  @Rollback(value = false)
  void cascadeTest() {
    Parent parent = new Parent();
    tem.persistAndFlush(parent);

    Child child1 = new Child();
    Child child2 = new Child();
    child1.setParent(parent);
    child2.setParent(parent);

    tem.persistAndFlush(child1);
    tem.persistAndFlush(child2);
    tem.clear();

    childRepository.deleteById(child1.getId());
  }
```

**💴 쿼리 로그 (3개의 delete 쿼리)**

```java
2025-04-20T23:54:35.497+09:00 DEBUG 36584 --- [           main] org.hibernate.SQL                        : 
    delete from child
    where  id=?
Hibernate: 
    delete from child
    where  id=?
2025-04-20T23:54:35.499+09:00 DEBUG 36584 --- [           main] org.hibernate.SQL                        : 
    delete from child
    where  id=?
Hibernate: 
    delete from child
    where  id=?
2025-04-20T23:54:35.500+09:00 DEBUG 36584 --- [           main] org.hibernate.SQL                        : 
    delete from parent 
    where  id=?
Hibernate: 
    delete from parent 
    where  id=?
```

자식 삭제 → 부모 삭제

- `cascade = ALL`이기 때문에 자식 삭제 시 부모도 cascade되어 삭제 대상에 포함

부모 삭제 → 자식 삭제

- `orphanRemoval = true`로 인해 parent의 **컬렉션인 childeren들이 삭제됨**

> Cascade 속성은 기본적으로 부모 → 자식 으로 전파되도록 설계하는 것이 좋을 것 같다.
>

### ✅ ManyToOne cascde에 대한 생각

앞선 시나리오들을 보면서 문득, 이런 생각이 들었다.

아래와 같이 @ManyToOne으로 Cascade를 걸어놓으면

```java
public class child {
		@ManyToOne(cascade=CascadeType.ALL)
		private Parent parent;
```

Child 객체 하나 죽음 → 부모 죽음 → 부모의 Child들이 죽음

> **이게 과연 의도된 것일까? No**
>
>
> 하나의 자식이 죽으면 나머지 자식도 다 죽게 처리하는 시나리오가 있을까??? 없다고 본다.
>
> 결론 : cascade 사용 시에는 주로 부모쪽에 걸려고 노력하자



# 추가로 공부할 것 
ManyToOne에서 cascade 없이 삭제하는법이 있다.
@OnDelete 인데,
@OnDelete는 cascade와 달리, DB에서 처리해주기 때문에 단일한 쿼리를 통해 연쇄적으로 제거할 수 있다.
이전에 봤던 쿼리 3개 나가는 문제를 @OnDelete를 사용하면 해결할 수 있다.
