---
title: "kotlin-spring 간단한 통합 테스트 설정 2"
date: 2021-08-26T08:27:27+09:00 
draft: false 
categories: ['spring']
tags: ['kotlin', 'spring', 'test', 'testcontainers']
---

> 이전 [포스트]({{< ref "../spring_integration_test_ex">}})에서 testcontainers를 이용한 스프링 통합 테스트를 다뤘습니다.
> 이 포스트는 해당 포스트에서 작성한 환경 설정에서 더 나아가는 환경 설정을 다뤘습니다.

### 기존 상황

> testcontainers를 통해 docker 가 설치된 곳이라면 어디서든 독립적으로 통합 테스트를 실행할 수 있도록 만들었습니다.
> 하지만! 여전히 문제가 있었습니다.
>
> 이전 포스트와 같은 설정으로 돌릴 경우, spring의 컨테이너가 시작하기 위해서 띄우는 db 컨테이너와 별개로 테스트에서 쓰기 위해 컨테이너를 하나 더 띄웁니다.

### 목적

> docker container를 한번만 띄우고 싶다.

### 해결 방법

> [youtube 영상](https://youtu.be/gF-YG6YZxZk)을 보고 테스트에서 띄운 도커 컨테이너의 설정을
> spring 설정에 동적으로 할당하는 방법이 있었습니다. 이렇게 되면, spring web container가 뜰 때 db 연결을 이미 떠있는 테스트용 컨테이너로 합니다.
> 따라서 db 컨테이너를 두번 띄우는 낭비를 범하지 않습니다.

### 구체적인 설정

#### dependency 설정

이전과 달라지는 것은 없습니다.

#### test 용 spring 설정

`driver-class-name`을 바꾸고, `url`을 주석처리나 지워줍니다.

```yaml
spring:
  datasource:
    #   !!!바뀐 설정!!! url은 동적으로 설정될 거라 url 설정은 필요없습니다. 주석처리나 삭제를 부탁드립니다.
    #    url: jdbc:tc:postgresql:///test-database?stringtype=unspecified

    #   !!!바뀐 설정!!! 기존 DB에 붙을 것이기 때문에 정상적인 postgresql 드라이버를 써도 됩니다.. 
    driver-class-name: org.postgresql.Driver

    # 컨테이너로 띄운 postgresql 접속할 때 쓰는 driver, 기존 postgresql 드라이버는 동작하지 않습니다.. 
    # ㄴ-> 설명을 잘못했습니다. 도커 컨테이너로 띄운 db에 접속하기 위한 게 아니라 컨테이너를 띄우는 것까지 포함한 driver. 
    # 실제로 이미 떠있는 postgresql db 컨테이너 접속할 때는 해당 드라이버로 동작하지 않습니다. ㅈㅅ 
    # driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
  jpa:
    hibernate:
      ddl-auto: create
    database-platform: org.hibernate.dialect.PostgreSQL9Dialect
    show-sql: true
```

#### test class 설정

`static` 메소드로 다이나믹하게 설정하는 부분을 추가합니다. 아래에 자세한 내용을 참고해주세요.

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = ["spring.profiles.active=test"]) // can override any propertysource
@TestConstructor(autowireMode = TestConstructor.AutowireMode.ALL) // autowire without @autowired
class SpringBootIntegrateTestExApplicationTests(
    private val restTemplate: TestRestTemplate
) {

    @Test
    @Transactional
    fun contextLoads() {
        println("context loads")
        // 실제 스프링에 구현되어 있는 컨트롤러 url을 스펙에 맞춰서 호출한다.
        // 나는 해당 url을 호출하면 entity 하나를 자동 생성하는 api를 만들었다. 
        val response = restTemplate.postForObject<TeamResponse>("/team")
        // 자동 생성이고, 테스트마다 db를 생성하기 때문에 매번 1이 출력된다.
        println(response?.id)

    }

    // 해당 static 변수를 설정해야 컨테이너를 띄운다. 
    // database 이름은 하기의 이름과 yml 설정에서 같게 만든다. 
    companion object {
        @JvmStatic
        protected val dbContainer = PostgreSQLContainer<Nothing>("postgres:latest").apply {
            withDatabaseName("test-database")
        }

        // 해당 부분을 추가한다. 
        // 다이나믹하게 스프링 설정을 추가할 수 있게 해주는 것이 @DynamicPropertySource다. 
        // 애초에 testcontainers 지원을 목적으로 만들어졌다. 
        @JvmStatic
        @DynamicPropertySource
        fun datasourceConfig(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", dbContainer::getJdbcUrl)
            registry.add("spring.datasource.password", dbContainer::getPassword)
            registry.add("spring.datasource.username", dbContainer::getUsername)
        }

        init {
            dbContainer.start()
        }
    }
}
```

나머지는 변경 사항이 없습니다.

https://github.com/sukyology/spring-boot-integrate-test-ex <= 예제 코드

-끝-

#### Bibliography

https://youtu.be/gF-YG6YZxZk