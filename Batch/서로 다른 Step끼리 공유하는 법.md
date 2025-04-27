스프링 배치를 활용한 프로젝트를 하면서 Step끼리 자원을 공유해야하는 일이 있었다.
여러 방법이 있는데, 나는 그 중 제일 안 좋은 방법을 쓰고 있었다.
이 글에서는, 내가 쓰는 데이터 공유 방법의 문제와 더 나은 방법(레퍼런스가 추천하는)에 대해 작성해보려 한다.

아래는 나의 프로젝트의 구조다.

## 문제

![기사 수집 배치 작업 흐름도.png](attachment:e8eb09e0-453d-4d63-ae90-03a33255067f:기사_수집_배치_작업_흐름도.png)

### ‼️**상황 요약**

- Article Step에서 가져온 List<Interest>를 JobExecutionContext에 저장 후 향후 Step에서 사용
- Flow에서 가져온 뉴스 기사 정보를 JobExecutionContext에 저장 후 향후 Step에서 사용

### **💢 문제 상황**

- Step 중간에 저장할 데이터를 JobExecutionContext에 바로 저장하는 것의 잠재적 문제가 있다는 것을 발견
    
    기존에는 아래처럼 저장했었다.
    
    ```java
      @Bean
      public Step executionTestStep1() {
        return new StepBuilder("executionTestStep", jobRepository)
            .tasklet((contribution, chunkContext) -> {
              String saveData = "저장할 데이터";
              
              ExecutionContext jobContext = contribution
                  .getStepExecution()
                  .getJobExecution()
                  .getExecutionContext();
    
              jobContext.put("저장", saveData);
              return RepeatStatus.FINISHED;
            }, transactionManager)
    ```
    

### 🚨 JobExecutionContext에 바로 저장하는 것의 문제

- **Step 실패 시 문제**
    - 저장 후 Step 실패 시 롤백되는데 뭐가 문제냐? 라고 생각할 수 있다.
    - 하지만 완전하지 않은 Step에서 JobContext를 오염시키는 것은 불안정적이다.
- **높은 결합도 → 재사용성 감소**
    - JobExecutionContext에 높은 의존도를 가진다.
    - 이러한 결합도는 재사용성을 감소 시킨다.

## 최고의 방법. PromotionListener 사용

### 레퍼런스

스프링 배치 레퍼런스에서 공식적으로 권장하는 방식이다. 

https://docs.spring.io/spring-batch/reference/common-patterns.html#passingDataToFutureSteps

**(다음 Step에 전달하는 방법에 대한 레퍼런스)**

![image.png](attachment:229deefa-1a69-46f7-aba3-6ef285744355:image.png)

### 사용 방법

**1️⃣ExecutionContextPromotionListener를 Step에 등록**

2️⃣ 데이터 승격 

StepExecution에 저장한 데이터를 JobExecution으로 승격할 데이터를 setKeys()로 지정

```java
  @Bean
  public ExecutionContextPromotionListener promotionListener() {
				 
 1️⃣ExecutionContextPromotionListener listener = new ExecutionContextPromotionListener();
 2️⃣listener.setKeys(new String[]{"step에 저장") wd
   return listener;
  }
```

3️⃣ Step에  Listener 등록 

```java
  @Bean
  public Step executionTestStep1() {
    return new StepBuilder("executionTestStep", jobRepository)
        .tasklet((contribution, chunkContext) -> {
          String saveData = "저장할 데이터";

          ExecutionContext stepContext = contribution
				          .getStepExecution()
				          .getExecutionContext();
				          
          stepContext.put("step에 저장", saveData);

          return RepeatStatus.FINISHED;
        }, transactionManager)
        .listener(promotionListener())    // promotion Listener 등록 
        .build();
  }
```

✅ JobContext에서 꺼내기 in 다음 Step 에서 

```java
  @Bean
  public Step executionTestStep2() {
    return new StepBuilder("executionTestStep2", jobRepository)
        .tasklet((contribution, chunkContext) -> {
          ExecutionContext jobContext = contribution
              .getStepExecution()
              .getJobExecution()
              .getExecutionContext();

          String pullData = (String) jobContext.get("step에 저장");
          System.out.println("꺼낸 값 = " + pullData);
          return RepeatStatus.FINISHED;
        }, transactionManager)
        .build();
  }
```

![image.png](attachment:18d70153-5bb2-4a41-a588-e9fbee699f2b:image.png)

분명 StepContext에 넣었는데 JobContext에 꺼내는게 성공했다. 

이는 PromotionListener 덕분이다.

### **✅ PromotionListner 사용 시 장점**

1. **안정성** 
    - Step이 완전히 정상적으로 끝났을 때, 필요한 데이터만 JobExecutionContext에 복사
    - 이로 인해, JobExecutionContext에 잘못된 데이터가 안 들어감.
2. **재사용성**
    - Step이 JobContext에 결합하지 않기 떄문에 재사용성이 좋음