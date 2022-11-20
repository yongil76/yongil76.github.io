---
title:  "스프링 배치 잡에서 Deadlock이 발생해서 잡이 실행되지 못한 이유"

excerpt: "Deadlock in spring batch spring job"

categories:
- Spring batch

tags:
- Spring batch
- JobRepository
- Deadlock

toc: true

toc_sticky: true

toc_label: "Deadlock in job repository"

last_modified_at: 2022-11-06T09:59:00-05:00

---

## Problem?

> Spring batch에서 JobInstance 생성시, Job이 Deadlock으로 롤백되는 현상

---

## 필요한 지식 

---

- [Deadlock](https://yongil76.github.io/db/InnoDB_Deadlock/)
- [Isolation level](https://yongil76.github.io/db/Isolation_level/)
- [Inno DB Lock](https://yongil76.github.io/db/InnoDB_Locking/)


## 프록시 적용

---

~~~java
public abstract class AbstractJobRepositoryFactoryBean implements FactoryBean<JobRepository>, InitializingBean {
  private void initializeProxy() throws Exception {
    if (this.proxyFactory == null) {
      this.proxyFactory = new ProxyFactory();
      TransactionInterceptor advice = new TransactionInterceptor(this.transactionManager, PropertiesConverter.stringToProperties("create*=PROPAGATION_REQUIRES_NEW," + this.isolationLevelForCreate + "\ngetLastJobExecution*=PROPAGATION_REQUIRES_NEW," + this.isolationLevelForCreate + "\n*=PROPAGATION_REQUIRED"));
      if (this.validateTransactionState) {
        DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(new MethodInterceptor() {
          public Object invoke(MethodInvocation invocation) throws Throwable {
            if (TransactionSynchronizationManager.isActualTransactionActive()) {
              throw new IllegalStateException("Existing transaction detected in JobRepository. Please fix this and try again (e.g. remove @Transactional annotations from client).");
            } else {
              return invocation.proceed();
            }
          }
        });
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.addMethodName("create*");
        advisor.setPointcut(pointcut);
        this.proxyFactory.addAdvisor(advisor);
      }

      this.proxyFactory.addAdvice(advice);
      this.proxyFactory.setProxyTargetClass(false);
      this.proxyFactory.addInterface(JobRepository.class);
      this.proxyFactory.setTarget(this.getTarget());
    }
  }
}
~~~

- createJobInstance는 Proxy에 의해서, create* 메소드는 모두 REQUIRES_NEW로 진행되기 때문에 하나의 트랜잭션이라고 볼 수 있음

## JobInstance 생성

---

~~~java
public class JdbcJobInstanceDao extends AbstractJdbcBatchMetadataDao implements JobInstanceDao, InitializingBean {
    public JobInstance createJobInstance(String jobName, JobParameters jobParameters) {
        Assert.notNull(jobName, "Job name must not be null.");
        Assert.notNull(jobParameters, "JobParameters must not be null.");
        Assert.state(this.getJobInstance(jobName, jobParameters) == null, "JobInstance must not already exist");
        Long jobId = this.jobIncrementer.nextLongValue();
        JobInstance jobInstance = new JobInstance(jobId, jobName);
        jobInstance.incrementVersion();
        Object[] parameters = new Object[] { jobId, jobName, this.jobKeyGenerator.generateKey(jobParameters), jobInstance.getVersion()
        };
        this.getJdbcTemplate().update(this.getQuery(
                "INSERT into %PREFIX%JOB_INSTANCE(JOB_INSTANCE_ID, JOB_NAME, JOB_KEY, VERSION) values (?, ?, ?, ?)"), parameters, new int[] { -5, 12, 12, 4 });
        return jobInstance;
    }

  public JobInstance getJobInstance(String jobName, JobParameters jobParameters) {
    Assert.notNull(jobName, "Job name must not be null.");
    Assert.notNull(jobParameters, "JobParameters must not be null.");
    String jobKey = this.jobKeyGenerator.generateKey(jobParameters);
    RowMapper<JobInstance> rowMapper = new JobInstanceRowMapper();
    List instances;
    if (StringUtils.hasLength(jobKey)) {
      instances = this.getJdbcTemplate().query(this.getQuery("SELECT JOB_INSTANCE_ID, JOB_NAME from %PREFIX%JOB_INSTANCE where JOB_NAME = ? and JOB_KEY = ?"), rowMapper, new Object[]{jobName, jobKey});
    } else {
      instances = this.getJdbcTemplate().query(this.getQuery("SELECT JOB_INSTANCE_ID, JOB_NAME from %PREFIX%JOB_INSTANCE where JOB_NAME = ? and (JOB_KEY = ? OR JOB_KEY is NULL)"), rowMapper, new Object[]{jobName, jobKey});
    }

    if (instances.isEmpty()) {
      return null;
    } else {
      Assert.state(instances.size() == 1, "instance counte must be 1 but was " + instances.size());
      return (JobInstance)instances.get(0);
    }
  }
}
~~~

- 이 메소드는, getJobInstance(SELECT)로 JobInstance가 없음을 확인하고, JobInstance(INSERT)를 생성
- this.getJobInstance 메소드를 보면, JOB_NAME, JOB_KEY에 걸릴 것을 예측할 수 있음
  - JOB_NAME, JOB_KEY는 JOB_INST_UN이라는 Unique Index
  - Unique Index이기 때문에, REPEATABLE_READ 이상은 Row-level lock이 발생

## 트랜잭션 상황

--- 

- 2개의 세션에 트랜잭션 1개씩 있는데 이름을 T1, T2
  - T1, T2는 로드밸런스 환경의 서버에서 각각 발생한 트랜잭션이라고 가정
  - T1과 T2는 모든 JobParameter가 동일한 상황
- T1에서 먼저,
  - createJobInstance로 들어갔을 때,
    - JOB_NAME("testJob")으로 SELECT문 실행해서,
    - JOB_NAME에 Row-level lock을 획득
- T2에서,
  - createJobInstance로 들어갔을 때,
    - 동일한 JOB_NAME("testJob")에 대해서 lock을 요청하고 기다리는 상황
- T1에서,
  - INSERT 쿼리가 필요해서 lock을 요청했지만, T1에서 먼저 lock을 요청하고 있는 상황때문에 Deadlock이 발생


## More...

---

- 현재, Deadlock이 발생하는 환경에서 JobParameter가 동일할 수 없는데, Deadlock이 발생할 수가 없는 구조
- JobRepostiory의 JOB_EXECUTION 테이블의 데이터도 없는 상황