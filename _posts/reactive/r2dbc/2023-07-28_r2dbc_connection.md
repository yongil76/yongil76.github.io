---
title:  "R2DBC Connection Pool 설정"

excerpt: "R2DBC Connection Pool 설정"

categories:
- Reactive

tags:
- R2DBC
- Fix


toc: true

toc_sticky: true

toc_label: "Reactive"

last_modified_at: 2023-07-26T23:00:00-05:00

---

## DataAccessResourceFailureException: Failed to obtain R2DBC Connection
- 정상적으로 데이터베이스에 연결이 되고, 배치 시스템이 R2DBC를 통해서 쿼리를 정상적으로 호출하다가,
- 위의 예외가 발생하고, 커넥션을 할당받지 못하는 에러가 발생했음

## Situation
- 배치 시스템 성능을 고도화하려고, 병렬 프로그래밍이 적용된 부분이 있었음
- 병렬 처리로 인해 커넥션 할당이 많아져, 데이터베이스에서 커넥션을 더 이상 할당해주지 못하는 상황으로 인지

## Solution
- 매번 커넥션을 맺는 방식보다는, 일반 MYSQL처럼 커넥션 풀에서 커넥션을 재사용할 수 있도록 설정이 필요
- R2DBC는 `io.r2dbc.r2dbc-pool`를 통해서, 커넥션 풀을 설정할 수 있었음

## Kotlin

#### `Dependency` 추가

~~~groovy
implementation("io.r2dbc:r2dbc-pool:1.0.0.RELEASE")
~~~

#### `@Bean` 추가

~~~kotlin
    @Bean
    fun connectionFactory(): ConnectionFactory {
        val connectionFactory = ConnectionFactories.get(
            builder()
                .option(DRIVER, "pool")
                .option(PROTOCOL, "mysql")
                .option(HOST, mysql.url)
                .option(PORT, mysql.port.toInt())
                .option(USER, mysql.username)
                .option(PASSWORD, mysql.password)
                .option(DATABASE, mysql.database)
                .build()
        )

        val configuration: ConnectionPoolConfiguration =
            ConnectionPoolConfiguration
                .builder(connectionFactory)
                .initialSize(2)
                .maxSize(10)
                .maxLifeTime(Duration.ofSeconds(600))
                .maxIdleTime(Duration.ofSeconds(600))
                .build()

        return ConnectionPool(configuration)
    }
~~~

- `maxSize`는 일반적으로 사용하는 10개를 지정헀지만, 실제로는 응답속도, DB 시스템 관리자와 협의가 필요해보임
- `maxLifeTime`, 커넥션의 생명 주기를 600초로 지정하여 슬로우 쿼리 방지
- `maxIdleTime`, 커넥션 재사용에 가장 핵심인 부분, 600초동안 사용되지 않더라도 커넥션을 해제하지 않음
- 커넥션을 해제하지 않으므로, 커넥션을 재사용할 수 있게 됨

## Result
- 해당 옵션을 추가하고, 위 예외는 모두 처리됨
- 위 설정으로도 문제가 발생한다면, DBA와 실제로 커넥션 사용에 대해서 모니터링하고 적정한 수치를 협의

## Reference
- https://github.com/r2dbc/r2dbc-pool