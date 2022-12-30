---
title:  "Springboot @Cacheable"
excerpt: "Springboot @Cacheable"

categories:
- Spring
  
tags:
- cache

toc: true

toc_sticky: true

toc_label: "hbase"

last_modified_at: 2022-12-30T22:15:00-05:00

---

## Cache?

- 대기 시간을 줄이기 위한 방법
  - 동일한 결과를 얻기 위해, 다른 시스템에 여러번 접근하는 것보다 특정 장소에 저장하는 것이 효율적
- 처리량을 증가시키기 위한 방법
  - 실제 메모리에 접근하지 않고 캐시 메모리에 접근
- Key, Value 구조로 사용


## @Cacheable?

- 스프링에서 캐시를 쉽게 사용할 수 도와주는 어노테이션
- pom.xml

~~~pom
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
    <version>2.4.0</version>
</dependency>
~~~

- Configuration
- CacheManager는 Interface로 여러 구현체를 가질 수 있음 
  - Distributed-cache(서버 외부)
    - `RedisCacheManager`
  - Embedded-cache
    - `ConcurrentMapCacheManager`
    - `CaffeineCacheManager`

~~~java
@Configuration
@EnableCaching
public class CachingConfig {
  @Bean
  public CacheManager cacheManager() {
    return new ConcurrentMapCacheManager("addresses");
  }
}
~~~

- `@Cacheable`은 프록시 패턴이므로, 같은 클래스 내부에서 호출 시에는 효과가 없음
  - `@Transactional`처럼 프록시 패턴이 적용

~~~java
@Service
@RequiredConstructor
public class StudentService {
    private StudentMapper studentMapper;
    
    @Cacheable(value = "addresses", key = "#name") 
    public String getAddress(String name) {
        return studentMapper.selectAddressByName(name).address;
    }
}
~~~

- 첫번째 호출에는 데이터베이스로 쿼리가 발생하지만, 이후에는 쿼리가 발생하지 않고 캐시로 응답
- 만약, 학생의 주소가 바뀌면 캐시는 어떻게 처리할 것인가? 정답은 `@CacheEvict`

~~~java
@Service
@RequiredConstructor
public class StudentService {
    private StudentMapper studentMapper;

    @CacheEvict(value = "addresses", key ="#name") 
    public String updateAddress(String name, String address) {
        return studentMapper.updateAddress(name, address);   
    }
}
~~~

- `@CacheEvict`로 메소드가 호출되는 순간, 캐시에 있는 데이터를 Expire


## Cache는 항상 좋은가?

- 다양한 캐시 정책은 캐시 적중률을 높이기 위해서 존재
- 캐시 적중률이 낮으면, 캐시를 처리하는 오버헤드가 오히려 증가
- 따라서, 캐시를 무작정 사용하기보다 캐시 적중률이 충분히 발생할 수 있는 환경에서 사용하는 것이 중요


## Reference
- [A Guide To Caching Spring](https://www.baeldung.com/spring-cache-tutorial)