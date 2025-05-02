## 문제 상황

Spring Batch를 활용해 대량의 데이터를 수집하고 DB에 저장하는 작업을 수행하던 중, 다음과 같은 예외가 발생했다.

```java
Caused by: org.postgresql.util.PSQLException: Can't infer the SQL type to use for an instance of … Use setObject() with an explicit Types value to specify the type to use.
```

흠. DB에서 보낸 오류다.

Use setObject() with an explicit Types value to specify the type to use.

특정 타입의 값을 사용하기 위해서 setObjetc를 쓰라는 경고가 떴다.

특정 타입???

## 코드 분석

현재 저장하려는 클래스는 아래와 같고, 벌크 연산을 위해 만들어 놓은 BatchJdbcWriter도 아래와 같다.

```java
**[저장 객체]**
public class Article {

  private UUID id;
  private Source source;    <<< Enum
  private String sourceUrl;
  private String title;
  private Instant publishDate;
  private String summary;
  private boolean deleted;

[jdbcWriter]
  
  @Bean
  @StepScope
  public JdbcBatchItemWriter<Article> articleJdbcItemWriter() {

    String articleInsertSql =
        "INSERT INTO articles (id, source, source_url, title, publish_date, summary, deleted) "
            + "VALUES ( :id, :source, :sourceUrl, :title, :publishDate, :summary, false)";

    return new JdbcBatchItemWriterBuilder<Article>()
        .dataSource(dataSource)
        .assertUpdates(false)
        .sql(articleInsertSql)
        .beanMapped()
        .build();
  }
  
```

이전에 간단히 로컬에서 다른 객체로 테스트 할 때는 됐는데 왜 오류가 나는 것인가?

이유는 간단했다. 타입을 지원하지 않아서이다.

이전에 테스트 했던 코드는 String, int 처럼 단순한 타입이었다.

하지만 실전에서는 Source같은 Enum타입이 존재했다.

### beenMapped방식의 한계

현재 사용하고 있는 객체와 SQL 매핑 방식은 “beenMapped”방식이다.

beenMapped 방식은 객체의 필드명과 :필드명 을 자동으로 매핑해준다.

<aside>
💡

PreparedStatement : 자바에서 DB에 쿼리 보낼 때 사용하는 객체

```java
INSERT INTO articles (id, source, source_url, ...) VALUES (?, ?, ?, ...);
```

</aside>

JdbcBatchItemWriter가  PreparedStatement에 자동으로 매핑하는데, 내부적으로 어떤 SQL 타입인지 추론을 해야한다. 하지만, **Enum 타입은 자동 추론이 불가능**하다.

## 해결법

### Enum을 명시적으로 변환

이러한 문제를 해결하기 위해서는  itemPreparedStatementSetter를 통해 직접 세팅해주면된다.

ps = preparedStatement

item = 매핑시키려는 객체

```java
  @Bean
  @StepScope
  public JdbcBatchItemWriter<Article> articleJdbcItemWriter() {

    String articleInsertSql =
        "INSERT INTO articles (id, source, source_url, title, publish_date, summary, deleted) "
            + "VALUES (?, ?, ?, ?, ?, ?, ?)";

    return new JdbcBatchItemWriterBuilder<Article>()
        .dataSource(dataSource)
        .assertUpdates(false)
        .sql(articleInsertSql)
        .itemPreparedStatementSetter((item, ps) -> {
          ps.setObject(1, item.getId());
          ps.setString(2, item.getSource().name());
          ps.setString(3, item.getSourceUrl());
          ps.setString(4, item.getTitle());
          ps.setTimestamp(5, Timestamp.from(item.getPublishDate()));
          ps.setString(6, item.getSummary());
          ps.setBoolean(7, false);
        })
        .beanMapped()
        .build();
```

.get~enum필드.name() 을 통해 문자열로 저장

Instant타입인 publishDate도 저장시 PostgreSQL이 원하는 Timestamp로 변환

## 번외 : PreparedStatement

JdbcBatchItemWriter는 내부적으로 preparedStatement방식을 사용하여 SQL 을 실행함

**특징**

- SQL을 실행할 때 사용하는 객체
- 파라미터(?)를 미리 정의하고 값만 나중에 바인딩
- 따라서 성능 최적화 굿