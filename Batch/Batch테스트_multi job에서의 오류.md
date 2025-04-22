### Spring-Batch 테스트가 어려운 이유

Spring Batch를 공부하면 할 수록 완성도가 부족한 프레임워크라는게 느껴진다.(비단 테스트만이 아니다)

일단은 테스트 코드에 관한 글이니 Spring-Batch에서 테스트의 불편함을 써보겠다.

**☑️ 테스트에서의 불편함** 

- 테스트 도구들의 설정이 너무 번거롭다.
- Job이 여러 개일 경우, 생성중인 테스트 도구들이 어떤 Job을 기준으로 테스트할지 명확하지 않아 충돌이 발생한다.
    - 이 문제는 단순히 @Qualifier, @Resource, @Autowired 등으로 해결이 되지 않는다.
- 메타데이터 정리가 자동으로 되지않아 상태관리가 힘들다.
- 공식 레퍼런스도 설명이 부실하고, 커뮤니티도 활발하지 않다.

**☑️ 일반 불편함** 

되게 많다. 진짜… 스프링 배치는 사용할수록 프레임워크가 너무한 것 같다.

자세하게 말하면 너무 길어져서 짧게 말하자면 

- ChunkProcessing을 지원하는 구현체는 많은데 성능이 뛰어나면서 유연한 구현체가 없다.
- 백업에 많이 쓰일텐데 AWS에도 호환성이 부족하고 Querydsl과 호환되는 구현체도 없다.
- 또한, 메타데이터 자동 drop안되기 등 여러 부분에서 불편한게 많다.

## 문제 상황

- spring-batch를 이용해 만든 Job을 테스트하는 상황
- 빈으로 등록한 job은 한 개가 아니다.
    - 한 개일때는 테스트 설정이 쉬운데…. 2개 이상만 되면 에러가 많이 남
    - batch는 자료도 부족하고 레퍼런스도 별로 친절하지 못해서 에러로 하루를 날린 상홯

### 문제의 코드

```kotlin
@SpringBootTest
@SpringBatchTest
@ActiveProfiles("test")
class NotificationDeleteBatchTest {

  @Autowired
  JobLauncherTestUtils jobLauncherTestUtils;

  @Autowired
  @Qualifier("notificationDeleteJob")
  Job notificationDeleteJob;

  @DisplayName("Job을 실행시키는 테스트")
  @Test
  void testNotificationDeleteJob() throws Exception {
```

### 에러 메시지

```kotlin

org.junit.jupiter.api.extension.ParameterResolutionException: Failed to resolve parameter [org.springframework.batch.core.Job arg0] in constructor [public com.sprint.monew.common.batch.ArticleS3BackupBatchTest(org.springframework.batch.core.Job)]: Failed to load ApplicationContext for [WebMergedContextConfiguration@65cfd565 testClass = com.sprint.monew.common.batch.ArticleS3BackupBatchTest, locations = [], classes = [com.sprint.monew.MonewApplication, com.sprint.monew.common.batch.ArticleS3BackupBatchTest.BatchTestConfig], contextInitializerClasses = [], activeProfiles = ["test"], propertySourceDescriptors = [], propertySourceProperties = ["org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true"], contextCustomizers = [org.springframework.batch.test.context.BatchTestContextCustomizer@6bd16207, org.springframework.boot.test.autoconfigure.OnFailureConditionReportContextCustomizerFactory$OnFailureConditionReportContextCustomizer@7ac9af2a, org.springframework.boot.test.autoconfigure.actuate.observability.ObservabilityContextCustomizerFactory$DisableObservabilityContextCustomizer@1f, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizer@42435b98, org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@9d200de, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@7573e12f, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@5dbf5634, org.springframework.boot.test.web.reactor.netty.DisableReactorResourceFactoryGlobalResourcesContextCustomizerFactory$DisableReactorResourceFactoryGlobalResourcesContextCustomizerCustomizer@749ab7b4, org.springframework.test.context.support.DynamicPropertiesContextCustomizer@0, org.springframework.boot.test.context.SpringBootTestAnnotation@f891b7ac], resourceBasePath = "src/main/webapp", contextLoader = org.springframework.boot.test.context.SpringBootContextLoader, parent = null]
Caused by: java.lang.IllegalStateException: Failed to load ApplicationContext for [WebMergedContextConfiguration@65cfd565 testClass = com.sprint.monew.common.batch.ArticleS3BackupBatchTest, locations = [], classes = [com.sprint.monew.MonewApplication, com.sprint.monew.common.batch.ArticleS3BackupBatchTest.BatchTestConfig], contextInitializerClasses = [], activeProfiles = ["test"], propertySourceDescriptors = [], propertySourceProperties = ["org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true"], contextCustomizers = [org.springframework.batch.test.context.BatchTestContextCustomizer@6bd16207, org.springframework.boot.test.autoconfigure.OnFailureConditionReportContextCustomizerFactory$OnFailureConditionReportContextCustomizer@7ac9af2a, org.springframework.boot.test.autoconfigure.actuate.observability.ObservabilityContextCustomizerFactory$DisableObservabilityContextCustomizer@1f, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizer@42435b98, org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@9d200de, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@7573e12f, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@5dbf5634, org.springframework.boot.test.web.reactor.netty.DisableReactorResourceFactoryGlobalResourcesContextCustomizerFactory$DisableReactorResourceFactoryGlobalResourcesContextCustomizerCustomizer@749ab7b4, org.springframework.test.context.support.DynamicPropertiesContextCustomizer@0, org.springframework.boot.test.context.SpringBootTestAnnotation@f891b7ac], resourceBasePath = "src/main/webapp", contextLoader = org.springframework.boot.test.context.SpringBootContextLoader, parent = null]
Caused by: org...BeanCreationException: Error creating bean with name 'jobLauncherTestUtils': No qualifying bean of type 'org.springframework.batch.core.Job' available: more than one 'primary' bean found among candidates: [articleCollectJob, notificationDeleteJob, s3BackupJob]
Caused by: org...**NoUniqueBeanDefinitionException**: **No qualifying bean of type 'org.springframework.batch.core.Job' available: more than one 'primary' bean found among candidates: [articleCollectJob, notificationDeleteJob, s3BackupJob]**
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.determinePrimaryCandidate(DefaultListableBeanFactory.java:201

```

주요 오류 내용은 아래와 같다
```kotlin
Caused by: org...**NoUniqueBeanDefinitionException**: **No qualifying bean of type 'org.springframework.batch.core.Job' available: more than one 'primary' bean found among candidates: [articleCollectJob, notificationDeleteJob, s3BackupJob]**

Caused by: org...BeanCreationException: Error creating bean with name '**jobLauncherTestUtils**': No qualifying bean of type 'org.springframework.batch.core.Job' available: more than one 'primary' bean found among candidates: **[articleCollectJob, notificationDeleteJob, s3BackupJob]**
```

### 에러 분석

`SpringBatchTest`는 `JobLauncherTestUtils` 빈을 자동으로 등록함으로써 Job을 테스트하기 편하게 한다.

이 빈을 만들 때 Spring Batch는 내부적으로

- `JobLauncherTestUtils` 에 Job 빈을 주입하려고 시도
- 후보가 3 개(`articleCollectJob`, `notificationDeleteJob`, `s3BackupJob`)라서 **NoUniqueBeanDefinitionException** 발생

## 여러 시도

### 시도 1. @Primary

특정 Job에 `@Primary`를 붙임으로써 그 빈만 주입이 되도록 해보았다.

❗❗❗❗❗❗❗결과는??? 에러 ❗❗❗❗❗❗❗

❓`@Primary` 처리를 했는데 왜 문제??

실제 프로덕션 코드를 실행할때는 `@Primary`가 적용되서 전혀 문제가 없는데, 테스트 코드에서만 문제가 발생한다. 

**✅ 시기의 문제** 

`JobLauncherTestUtils` 가 Job을 주입받는 시기는 언제일까???

빈의 생성 시점이다. 

→ 컨텍스트 로딩 단계에서 바로 터지므로, 테스트 클래스 안에서 `@Qualifier`나 `@Primary`를 아무리 써도 소용이 없습니다.

### 시도2 : 직접 JobLauncherTestUtils 빈 생성

문제의 원인이 `JobLauncherTestUtils`가 생성될 때 어떤 Job을 주입받아야될지 모르는 거니까

내가 만들어서 Job을 정해주면 되지 않을까???

```kotlin
    @Bean(name = "jobLauncherTestUtils")  // ← 이름만 변경
    @Primary
    JobLauncherTestUtils jobLauncherTestUtils(
        @Qualifier("s3BackupJob") Job s3BackupJob) {

      JobLauncherTestUtils utils = new JobLauncherTestUtils();
      utils.setJob(s3BackupJob);
      return utils;
    }
```

> **결과 : 💢터짐💢**
> 

흠….

생각해보니 이렇게 내가 빈을 만드록 @Primary를 한다해도 결국 @SpringBatchTest가 내부적으로 JobLauncherTestUtils 을 생성하는건 매한가지네…..

### 시도3 : 설정값 조절 - 역시나 실패

계속 안되니까 별 생각 없이 아무거나 막 시도해봤다.

Job을 실행시키지 않게 yml파일을 수정하는건데(지금은 그냥 코드 상 제외)

애초에 bean이 생성되는 과정에서 문제니까 당연히 해결되지 않는게 정상이다.

```kotlin
@SpringBootTest(
    properties = {"spring.batch.job.name=s3BackupJob", "spring.batch.job.enabled=false"}
)
```

> @SpringBatchTest 애노테이션은 exclude같은 특정 빈 생성안되게 하는 옵션이 없어서 너무 힘들다.
 

이 외에도 50번 이상은 삽질하며 돌린 것 같다.

## 해결방법

https://multifrontgarden.tistory.com/291

위의 글을 통해 중요한 힌트를 얻었다. 

이 분도 나랑 마찬가지로 spring batch 테스트의 고통을 느낀 것 같다.

어떻게 해결했는지는 코드를 보고 설명을 해보겠다.

### 일단 해결된 세팅

**(따로 애노테이션 커스텀)**

```kotlin
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@SpringBatchTest
@EnableBatchProcessing
@EnableAutoConfiguration // Spring Boot 자동 설정(테스트 코드에서 Spring Boot가 여러 Job등록 못하게 해야하니 수동)
@ComponentScan(
    basePackages = "com.sprint.monew",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.REGEX,
        pattern = ".*Batch$" 
        // “Batch"로 끝나는 빈의 이름은 모두 제외(Batch클래스명 만들때 주의)
    )
)
public @interface BatchTestConfig {

}

**************************************************************************
@BatchTestConfig
@SpringBootTest(classes = {NotificationDeleteBatch.class})
@ActiveProfiles("test")
@Sql(
    scripts = {"classpath:schema-batch-test.sql"},
    executionPhase = ExecutionPhase.BEFORE_TEST_METHOD
)
//@Import(NotificationDeleteBatch.class)
class NotificationDeleteBatchTest {

```

왜 이런 설정이 나왔는지 앞으로 알아볼 것 

### **✅ 문제의 본질 다시 생각하기**

🟣 **스프링 부트의 자동 등록** 

`@SpringBootTest`를 사용하면 Spring Boot의 자동 구성 기능 (`@EnableAutoConfiguration`) 덕분에 애플리케이션 설정이 자동으로 적용된다. 그런데 여기엔 문제가 있다.**`@ComponentScan` 범위 내의 모든 Job이 빈으로 등록되기 때문**이다.

**🟣 Batch의 테스트 도구 생성 도중 Job이 필요**  

**`@SpringBatchTest`**가 **`JobLauncherTestUtils`**를 초기화하려 할 때, Job이 여러개라 

**"어떤 Job을 실행해야 하지?"** 라는 선택의 문제가 생긴다.

> 이것이 마로, mulit Job에서 충돌이나 예외가 발생하는 이유다.
> 

### ✅ 해결 아이디어

아쉽게도 정말 부족한 Spring Batch의 완성도는 테스트 코드에서 생기는 multil job 문제를 해결할 생각이 없었나보다(레퍼런스만 봐도 해결할 생각 없음) 

특정 빈을 제외시키는 그런 속성은 없다.

**1️⃣ 불필요한 Job 빈 등록 막기 `@ComponentScan` 필터링 활용** 

```kotlin
@ComponentScan(
    basePackages = "com.sprint.monew",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.REGEX,
        pattern = ".*Batch$" 
        // “Batch"로 끝나는 빈의 이름은 모두 제외(Batch클래스명 만들때 주의)
    )
```

- 이번 프로젝트에서는 Job으로 등록되는 빈의 클래스명은 ~Batch로 끝나도록 설정했기에 이렇게 했다.

2️⃣ 테스트할 Job만 명시적으로 로딩 (`@SpringBootTest` or @Import)

```kotlin
@SpringBootTest(classes = {NotificationDeleteBatch.class})
```

- NotificationDeleteBatch안에는 테스트하려는 Job, Step, Tasklet, writer 등의 빈이 모두 있다.
- 테스트할 클래스를 지정

**3️⃣ 자동 설정 활성화 (`@EnableAutoConfiguration`)** 

```kotlin
@EnableAutoConfiguration // Spring Boot 자동 설정(테스트 코드에서 Spring Boot가 여러 Job등록 못하게 해야하니 수동)
```

**Spring Boot의 자동 설정 기능 키기** 

- `@ComponentScan` 커스터마이징 때문에 `@SpringBootTest`의 자동 설정 효과가 무력화될 수 있다.
- 이 설정이 없으면 `DataSource`, `TransactionManager` 등 기본 설정이 누락될 수 있다

4️⃣ `@EnableBatchProcessing`

`@EnableBatchProcessing`은 

- **`JobRepository`, `JobLauncher`, `JobExplorer`, `BuilderFactory`** 등 핵심 빈을 구성

스프링 부트 3.X부터는 `@EnableBatchProcessing`을 자동으로 활성화시켜줬는데, 지금 테스트 코드에서는 컴포넌트 스캔을 조정하고 해서 자동 활성화가 안되는 것 같다. 그래서 추가 


**참고 : @SpringBatchTest는 Batch관련 테스트용 유틸 활성화** 

### 참고 : filter사용 법

![image.png](attachment:890e8d0f-4a55-40be-af6d-7c8dc7c8f8a7:image.png)

- @