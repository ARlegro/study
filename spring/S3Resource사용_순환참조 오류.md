> **코드 설명 : Spring-Batch의 FileItemWriter에 쓰일 S3Resource 객체를 만들기 위한 코드**
>

## 문제 발생

문제의 코드

- 아래는 S3관련 설정 빈들을 만들기 위한 클래스다.

```java

@Configuration
@EnableConfigurationProperties(S3ConfigProperties.class)
@RequiredArgsConstructor
public class S3Config {

  private final S3ConfigProperties s3Properties; // 이건 @ConfigurationProperties로 정의한 클래스 
  private final S3OutputStreamProvider s3OutputStreamProvider;

  @Primary
  @Bean
  public S3Client s3Client() {
    AwsBasicCredentials credentials = getAwsBasicCredentials();
    Region s3Region = Region.of(s3Properties.region());

    return S3Client.builder()
        .credentialsProvider(StaticCredentialsProvider.create(credentials))
        .region(s3Region)
        .build();
  }

  @Bean(name = "articleS3Resource")
  public S3Resource articleS3Resource() {
    String location = "s3://" + s3Properties.bucket() + "/";
    return S3Resource.create(location, s3Client(), s3OutputStreamProvider);
  }
}
```

```kotlin
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'entityManagerFactory' defined in class path resource [org/springframework/boot/autoconfigure/orm/jpa/HibernateJpaConfiguration.class]: Failed to initialize dependency 'batchDataSourceInitializer' of LoadTimeWeaverAware bean 'entityManagerFactory': Error creating bean with name 'batchDataSourceInitializer' defined in class path resource [org/springframework/boot/autoconfigure/batch/BatchAutoConfiguration$DataSourceInitializerConfiguration.class]: Unable to load resources from classpath:org/springframework/batch/core/schema-postgresql.sql
Caused by: org....BeanCreationException: Error creating bean with name 'batchDataSourceInitializer' defined in class path resource [org/springframework/boot/autoconfigure/batch/BatchAutoConfiguration$DataSourceInitializerConfiguration.class]: Unable to load resources from classpath:org/springframework/batch/core/schema-postgresql.sql
Caused by: java.lang.IllegalStateException: Unable to load resources from classpath:org/springframework/batch/core/schema-postgresql.sql
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 's3Config' defined in file [C:\monu\sb1-monew-team04\out\production\classes\com\sprint\monew\global\config\S3Config.class]: Unsatisfied dependency expressed through constructor parameter 1: Error creating bean with name 'inMemoryBufferingS3StreamProvider' defined in class path resource [io/awspring/cloud/autoconfigure/s3/S3AutoConfiguration.class]: Unsatisfied dependency expressed through method 'inMemoryBufferingS3StreamProvider' parameter 0: Error creating bean with name 's3Client': Requested bean is currently in creation: Is there an unresolvable circular reference or an asynchronous initialization dependency?
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'inMemoryBufferingS3StreamProvider' defined in class path resource [io/awspring/cloud/autoconfigure/s3/S3AutoConfiguration.class]: Unsatisfied dependency expressed through method 'inMemoryBufferingS3StreamProvider' parameter 0: Error creating bean with name 's3Client': Requested bean is currently in creation: Is there an unresolvable circular reference or an asynchronous initialization dependency?
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 's3Client': Requested bean is currently in creation: Is there an unresolvable circular reference or an asynchronous initialization dependency?

```

일단 몇 가지 에러를 보자면

**1️⃣ schema-postgresql.sql 을 load할 수 없다**

- 이 sql은 spring-batch에서 자동으로 제공하는 sql문이다.
- 보통 개발 단계에서는 아래와 같이, 해놓고 항상 실행 되게 하는데 갑자기 이게 왜 문제가 발생한거지?

    ```kotlin
      batch:
        job:
            initialize-schema: always
    ```


**2️⃣ inMemoryBufferingS3StreamProvider을 생성하는 도중 에러**

- 이게 뭐지…???
- 일단, 내가 멀티파트, 메모리 기능 등을 조절할 수 있는 S3Resource를 생성하기 위해
  `S3OutputStreamProvider`를 사용했는데 이게 문젠가??
- 직역 : S3에 연결하기 위한 Stream을 제공해주는 객체가 인메모리에서 버퍼링이 걸렸다???

**3️⃣ 's3Client’을 생성하는 도중 에러발생**

- S3Client는 이전에 S3 다운로드 업로드 기능을 구현할 때와 거의 똑같은데….

🚫 **핵심 에러** 🚫

`BeanCurrentlyInCreationException`

`is there an unresolvable circular reference or an asynchronous initialization dependency?`

구글 번역 결과 : 해결할 수 없는 순환 참조나 비동기 초기화 종속성이 있나요?

## 원인 분석 - 순환 참조

왜 순환참조 에러 메시지가 뜬걸까???

코드를 다시 보자

```java

@Configuration
@EnableConfigurationProperties(S3ConfigProperties.class)
@RequiredArgsConstructor
public class S3Config {

  private final S3ConfigProperties s3Properties;
  private final S3OutputStreamProvider s3OutputStreamProvider;     // 참조 부분 

  @Primary
  @Bean
  public S3Client s3Client() {     // 참조 부분 
    AwsBasicCredentials credentials = getAwsBasicCredentials();
    Region s3Region = Region.of(s3Properties.region());

    return S3Client.builder()
        .credentialsProvider(StaticCredentialsProvider.create(credentials))
        .region(s3Region)
        .build();
  }

  @Bean(name = "articleS3Resource")
  public S3Resource articleS3Resource() {
    String location = "s3://" + s3Properties.bucket() + "/";
    // 참조 부분 
    return S3Resource.create(location, s3Client(), s3OutputStreamProvider);
  }
}
```

이 코드에서 등록되는 빈

1. `S3Config` : Bean들을 등록하는 `@Configuration` 클래스
2. `S3Client` :
3. `S3OutputStreamProvider`
4. `S3Resource`

이 코드에서 자동 주입시키는 빈

1. `S3ConfigProperties`  : 큰 문제 없음
2. `S3OutputStreamProvider` : S3Resource를 만드는 데 사용

⭐ 순환 참조 과정

1. S3OutputStreamProvider 생성 시점에 S3Client가 필요
2. 그런데, Spring이 S3Client 빈을 만들려고 보니 S3Config를 초기화 중

`S3OutputStreamProvider` 의 생성자 주입으로 인해, S3Config 빈의 생성 시점에 내부의 S3Client가 필요해서 부르는 상황. 근데 S3Client는 아직 S3Config가 초기화가 되지 않는 상황에서 꺼내짐

즉, 흔한 두 개 이상의 빈이 서로 물려있어서 참조가 맞물리는 상황이 발생

⇒ S3Client는 아직 빈을 초기화 중인데 를 호출하면서

## 해결

결국 문제는 S3Config 빈의 생성과정에서  `S3OutputStreamProvider` 랑 S3Client가 필요하다보니 S3Config 내부의 S3Client 생성 로직과 맞물리는 상황

✅ 해결 : `S3OutputStreamProvider` 가 필요하면 파라미터 주입 방식을 쓰자

- 파라미터 방식은 컨테이너가 **이미 준비된 빈**을 주입하므로 순환이 발생하지 않는다.

```java
@Configuration
@EnableConfigurationProperties(S3ConfigProperties.class)
@RequiredArgsConstructor
public class S3Config {

  private final S3ConfigProperties s3Properties;
  //private final S3OutputStreamProvider s3OutputStreamProvider; << 없앰 

  @Primary
  @Bean
  public S3Client s3Client() {
    AwsBasicCredentials credentials = getAwsBasicCredentials();
    Region s3Region = Region.of(s3Properties.region());

    return S3Client.builder()
        .credentialsProvider(StaticCredentialsProvider.create(credentials))
        .region(s3Region)
        .build();
  }

  @Bean(name = "articleS3Resource") // 여기서 
  public S3Resource articleS3Resource(S3OutputStreamProvider s3OutputStreamProvider) {
    String location = "s3://" + s3Properties.bucket() + "/";
    return S3Resource.create(location, s3Client(), s3OutputStreamProvider);
  }
}
```