

# 스프링 배치 (Spring Batch)

------

## 1. 스프링 배치 메타테이블

### 1-1. BATCH_JOB_INSTANCE

| JOB_INSTANCE_ID | VERSION | JOB_NAME  | JOB_KEY                          |
| --------------- | ------- | --------- | -------------------------------- |
| 1               | 0       | simpleJob | d41d8cd98f00b204e9800998ecf8427e |

- JOB_INSTANCE_ID : BATCH_JOB_INSTANCE 테이블의 PK
- JOB_NAME : 수행한 Batch Job Name

**BATCH_JOB_INSTANCE 테이블**은 Job Parameter에 따라 생성되는 테이블  
**Job Parameter**는 Spring Batch가 실행될때 외부에서 받을 수 있는 파라미터

```
같은 Batch Job 이라도 Job Parameter가 다르면 Batch_JOB_INSTANCE에는 기록되며, Job Parameter가 같다면 기록되지 않음
```

### 1-2. BATCH_JOB_EXECUTION

- **JOB_EXECUTION**와 **JOB_INSTANCE**는 부모-자식 관계

- **JOB_EXECUTION**은 자신의 부모 **JOB_INSTACNE**가 성공/실패했던 모든 내역을 갖고 있음

  동일한 Job Parameter로 성공한 기록이 있을때만 재수행이 안됨

### 1-3. BATCH_JOB_EXECUTION_PARAMS

- BATCH_JOB_EXECUTION 테이블이 생성될 당시에 입력 받은 Job Parameter를 담고 있음


------

## 2. Spring Batch Job Flow

### 2-1. Job & Step

- Spring Batch 의 Job을 구성하는데 Step 이 존재함

```java
    @Bean
    public Job simpleJob() {
        return jobBuilderFactory.get("simpleJob")
                .start(step1())
                .next(step2())
                .next(step3())
                .build();
    }
    
    @Bean
    public Step step1() {
        return stepBuilderFactory.get("step1")
                .tasklet((contribution, chunkContext) -> {
                    log.info(">>>>> This is Step1");
                    return RepeatStatus.FINISHED;
                })
                .build();
    }

    @Bean
    public Step step2() {
        return stepBuilderFactory.get("step2")
                .tasklet((contribution, chunkContext) -> {
                    log.info(">>>>> This is Step2");
                    return RepeatStatus.FINISHED;
                })
                .build();
    }

    @Bean
    public Step step3() {
        return stepBuilderFactory.get("step3")
                .tasklet((contribution, chunkContext) -> {
                    log.info(">>>>> This is Step3");
                    return RepeatStatus.FINISHED;
                })
                .build();
    }
```

- Job 에는 처리하는 코드가 없고 Step에 실제 처리 로직이 들어감
- Step은 Batch로 실제 처리하고자 하는 기능과 설정을 모두 포함하는 장소

#### Next

- 순차적으로 Step들 연결시킬때 사용  
  (step1 -> step2 -> stpe3 순으로 하나씩 실행시킬때)

### 2-2. 지정한 Batch Job만 실행 옵션

- application.yml 의 옵션 추가

> **spring.batch.job.names: ${job.name:NONE}**  
> Spring Batch가 실행될때, Program arguments로 job.name 값이 넘어오면 해당 값과 일치하는 Job만 실행

- job.name이 있으면 job.name값을 할당하고, 없으면 NONE을 할당하겠다는 의미
- spring.batch.job.names에 NONE이 할당되면 어떤 배치도 실행하지 않겠다는 의미
- 즉, 혹시라도 값이 없을때 모든 배치가 실행되지 않도록 막는 역할



### 2-3. 조건별 흐름 제어 (Flow)

- Next를 이용하면 Step을 순차적으로 실행 하고 중간에 실패 시 다음 Step을 실행하지 않음
- 상황에 따라 Step A 가 정상이면 Step B를, Step A가 실패하면 Step C를 실행해야 할 때가 있음
- 전체 Flow를 관리

```java
    @Bean
    public Job stepNextConditionalJob() {
        return jobBuilderFactory.get("stepNextConditionalJob")
                .start(conditionalJobStep1())
                    .on("FAILED") // FAILED 일 경우
                    .to(conditionalJobStep3()) // step3으로 이동한다.
                    .on("*") // step3의 결과 관계 없이
                    .end() // step3으로 이동하면 Flow가 종료한다.
                .from(conditionalJobStep1()) // step1로부터
                    .on("*") // FAILED 외에 모든 경우
                    .to(conditionalJobStep2()) // step2로 이동한다.
                    .next(conditionalJobStep3()) // step2가 정상 종료되면 step3으로 이동한다.
                    .on("*") // step3의 결과 관계 없이
                    .end() // step3으로 이동하면 Flow가 종료한다.
                .end() // Job 종료
                .build();
    }
```

- on()
  - 캐치할 ExitStatus
  - \* 인 경우 모든 ExitStatus 캐치
- to()
  - 다음으로 이동할 Step 지정
- from()
  - 일종의 이벤트 리스너 역할
  - 상태값을 보고 일치하는 상태라면 to()에 포함된 step을 호출
  - step1의 이벤트 캐치가 FAILED로 되있는 상태에서 추가로 이벤트 캐치하려면 from을 써야만 함
- end()
  - end는 FlowBuilder를 반환하는 end와 FlowBuilder를 종료하는 end 2개가 있음
  - on("*")뒤에 있는 end는 FlowBuilder를 반환하는 end
  - build() 앞에 있는 end는 FlowBuilder를 종료하는 end
  - FlowBuilder를 반환하는 end 사용시 계속해서 from을 이어갈 수 있음
  


### 2-4. Decide
- Spring Batch에서는 Step들의 Flow속에서 분기만 담당하는 타입인 `JobExecutionDecider` 

```java
    public static class OddDecider implements JobExecutionDecider {

        @Override
        public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
            Random rand = new Random();

            int randomNumber = rand.nextInt(50) + 1;
            log.info("랜덤숫자: {}", randomNumber);

            if(randomNumber % 2 == 0) {
                return new FlowExecutionStatus("EVEN");
            } else {
                return new FlowExecutionStatus("ODD");
            }
        }
    }
```

- `JobExecutionDecider` 인터페이스를 구현한 OddDecider
- 랜덤하게 숫자를 생성하여 홀수/짝수인지에 따라 서로 다른 상태를 반환  
*※ Step으로 처리하는게 아니기 때문에 ExitStatus가 아닌 FlowExecutionStatus로 상태를 관리*








> 참고 : https://jojoldu.tistory.com/328?category=635883