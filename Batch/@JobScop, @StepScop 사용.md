## **✅ Scope(스코프)란??**

- 스프링 컨테이너에서 스코프란, 빈의 생명주기와 접근 범위를 정의하는 개념이다
- 종류로는 singleton, prototype, request. session 등이 있으며,
- 스프링에서 빈은 기본적으로 singleton으로 생성된다.

## **✅ 배치 Scope**

### 기본 개념

배치를 처리하다보면 동적으로 값을 주입받아야 하는 경우가 많다.

스프링 배치는 동적으로 값을 받아 빈을 생성시키기 위해 커스텀 스코프를 제공해주는데 대표적인 2가지가 아래와 같다.

1. @JopScope
2. @StepScope

<aside>
💡

**이 기능을 쓰면 좋은 점 미리 알기 (뒤에서 또 언급)**

- 단순히 동적 주입만이 장점이 아니다
- 좋은 점은 **스레드 안정성**이 있다는 것이다.
- 빈이 늦게 생성되어서 성능이 부담된다고 느낄 수 있지만, 재사용성-동적-스레드 안정성의 장점 때문에 습관적으로 @~Scope를 쓰는 것 같다.
</aside>

### 그림 이해

![image](https://github.com/user-attachments/assets/9df4e6af-67b3-4aa3-bf65-09404083ca4f)

이 애노테이션이 붙은 빈은 프록시 모드로 선언되어 내부적으로 Proxy객체가 생성된다.

애플리케이션 구동시점에는 프록시 객체가 생성되고, 실제 이 빈이 실행되는 시점에 실제 빈을 호출한다.

큰 장점으로는, 병렬 처리 시 각 스레드마다 생성된 스코프 빈이 할당되기 때문에 스레드에 안전하게 실행이 가능하다.

![image](https://github.com/user-attachments/assets/c9a22fd8-6ed7-4c77-a8a7-2225c7e75dc6)


즉, Job이 실행될 때 실제 빈 생성 후 사용

> **아래로 가면 더 자세한 그림이 나오니 그거 확인할 것**
>

# 시작 
## 1. @JobScope

**✅ 개념**

- Job 실행 시점마다 빈이 생성된다.
- **사용 가능한 곳 :** step 선언문에만 정의가 가능하다.

  🚨 즉, Tasklet, Reader, Processor, Writer 등에는 붙이지 않는다.

- **@Value 활용하면**
    - JobParameter 또는 JobExecutionContext 값을 주입 받을 수 있다
    - 이를 통해 동적 주입이 가능한 것이다.

    ```kotlin
      @Bean("s3BackupStep")
      @JobScope
      public Step s3BackupStep(
            @Value("#{jobParameters['backupDateTime']}") LocalDateTime toBackupDateTime) 
    ```


## 2. @StepScope

**✅ 개념**

- Step 실행 시점마다 빈이 생성된다.
- 1번의 경우와 달리 JobParameter, JobExecutionContext, StepExecutionContext까지 주입 가능
- **사용 가능한 곳** : Tasklet, ItemReader, ItemProcessor, ItemWriter에 사용

- @Value를 활용하면
    - 1번의 경우와 마찬가지로 JobParameter 또는 JobExecutionContext 값을 주입 받을 수 있다
    - 또한, @StepScope는 stepExecutionContext의 값도 주입 받을 수 있따.

    ```kotlin
      @Bean("s3BackupStep")
      @StepScope
      public ItemWriter<? ?>s3BackupItemWriter(
            @Value("#{jobParameters['backupDateTime']}") LocalDateTime toBackupDateTime) 
    ```


# 아키텍쳐(동작 원리)

## Proxy가 어떻게 호출을 할까?

❓@StepScope, @JobScope가 붙은 빈은 Proxy객체로 생성될텐데 어떤 원리로 실제 객체를 불러올까❓

사실, 실제 빈의 위치는 JobContext, StepContext에 있다!!

**JobContext, StepContext**

- **스프링 컨테이너에서 생성된 빈을 저장하는 컨텍스트** 역할
    - Job,StepContext가 필요한 빈들만 저장하고 있음
- 즉, Proxy빈이 실제 참조할 때 활용됨
    - 실제 빈 참조시 ApplicationContext가 아닌 Job(Step)Context를 부르는 것
- JobContext → Job 실행 중 동적으로 생성된 Bean을 저장/관리
- StepContext → Step 실행 중 동적으로 생성된 Bean을 저장/관리

**그림으로 이해하기**

![image](https://github.com/user-attachments/assets/7ce1d18e-c98b-420a-a2cb-4b0ab6aeaf0c)

(출처 : 인프런 정수원님 배치 강의)

- Job이 Proxy객체인 Step을 호출
- JobScope클래스는 내부적으로 JobContext 속성을 가지고 있다.
- 이를 통해 Proxy객체가 참조할 빈을 JobScope에게 요청하면 JobScope는 자신이 갖고 있는 JobContext속성을 참고하여 실제 빈을 찾음

**✅ 런타임 시점에 빈 생성 + @Valid 바인딩 (그림 참고)**

- @JobScope를 붙인 step은 애플리케이션 구동 시점에는 빈이 없기 떄문에 호출 당하면, BeanFactory에서 빈을 생성해준다.
- 이 생성 시점에!!! @Value 바인딩처리가 이루어짐
- 따라서, @Job(step)Scope없이 @Value를 쓰면 오류가 나는 이유가 이것때문이다.

## 활용하기 좋은 경우 (장점)

**❓ 언제 이 Scope를 사용하면 좋을까❓**

- `@Value`나 JobParameter를 통해 외부 파라미터를 주입 받아야 할 때

  ex. Job 실행마다 다른 파라미터를 동적으로 처리하고 싶을 때

- 병렬로 실행되는 Step들에서 각각 다른 상태를 가지는 Bean을 생성하고 싶을 때

# 번외 : 실제 구현된 거 살펴보기

### 맛보기 - 기본구조

@JobScope의 기본 구조를 보자

```kotlin
/**
 * <p>
 * Convenient annotation for **job-scoped beans that defaults the proxy mode**, so that it
 * does not have to be specified explicitly on every bean definition. **Use this on any
 * &#64;Bean that needs to inject &#64;Values from** the job context, and any bean that
 * needs to share a lifecycle with a job execution (such as an JobExecutionListener). The
 * following listing shows an example:
 * Marking a &#64;Bean as &#64;JobScope is equivalent to marking it as
 * <code>&#64;Scope(value="job", proxyMode=TARGET_CLASS)</code>

 */
@Scope(value = "job", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface JobScope {

```

**주석 해석**

- 프록시 모드로 되는 것
- 동적 주입이 가능하다는 것
    - 사용해라 → @Value로 ~~ 어떤걸 할 떄??

**다음은 @StepScope의 기본구조**

```kotlin
/**
 * <p>
 * Convenient annotation for step-scoped beans. It defaults the proxy mode so that it need
 * not be specified explicitly on every bean definition. Use this on any &#64;Bean that
 * needs to inject &#64;Values from the step context and on any bean that needs to share a
 * lifecycle with a step execution (such as an ItemStream). The following listing shows an
 * example:
 * </p>
 *
 * <pre class="code">
 * &#064;Bean
 * &#064;StepScope
 * protected Callable&lt;String&gt; value(@Value(&quot;#{stepExecution.stepName}&quot;)
 * final String value) {
 *     return new SimpleCallable(value);

 */
@Scope(value = "step", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface StepScope 
```
