---
title: "datasource-proxy를 활용해 n+1 쿼리 방지하기"
date: 2021-09-10T08:27:27+09:00
draft: false
categories: ['spring']
tags: ['kotlin', 'spring', 'test']
---

## 기존 상황
n+1 쿼리를 db 호출 로그를 통해 육안으로 확인하고 있었습니다.  

## 목적
테스트를 통해서 n+1 쿼리를 빠르게 방지할 수 있는 방법을 찾고 싶습니다. 

## 해결 방안
datasource-proxy 라이브러리를 통해 실행한 쿼리문의 개수를 불러와서 예상한 쿼리문의 수와 맞는지 비교합니다. 

## 설정

`build.gradle.kts`에 다음과 같은 내용을 `dependencies`에 추가합니다. 

```yaml
 implementation("net.ttddyy:datasource-proxy:1.7")
```

그리고 spring-boot에서 proxybean을 생성하게 위해서 BeanPostProcessor를 하나 만들어서 등록해줍니다. 
테스트에서만 사용할 거라 적절한 `@Profile`도 붙여줍니다. 

```kotlin
@Configuration
@Profile("test")
class DatasourceProxyBeanPostProcessor : BeanPostProcessor {
    override fun postProcessAfterInitialization(bean: Any, beanName: String): Any? {
        if (bean is DataSource && bean !is ProxyDataSource) {
            // Instead of directly returning a less specific datasource bean
            // (e.g.: HikariDataSource -> DataSource), return a proxy object.
            // See following links for why:
            //   https://stackoverflow.com/questions/44237787/how-to-use-user-defined-database-proxy-in-datajpatest
            //   https://gitter.im/spring-projects/spring-boot?at=5983602d2723db8d5e70a904
            //   http://blog.arnoldgalovics.com/2017/06/26/configuring-a-datasource-proxy-in-spring-boot/
            val factory = ProxyFactory(bean)
            factory.isProxyTargetClass = true
            factory.addAdvice(ProxyDataSourceInterceptor(bean))
            return factory.proxy
        }
        return bean
    }

    override fun postProcessBeforeInitialization(bean: Any, beanName: String): Any? {
        return bean
    }

    private class ProxyDataSourceInterceptor(dataSource: DataSource?) : MethodInterceptor {
        private val dataSource: DataSource

        @Throws(Throwable::class)
        override operator fun invoke(invocation: MethodInvocation): Any? {
            val proxyMethod = ReflectionUtils.findMethod(
                dataSource::class.java,
                invocation.method.name
            )
            return if (proxyMethod != null) {
                proxyMethod.invoke(dataSource, *invocation.arguments)
            } else invocation.proceed()
        }

        init {
            val listener = ChainListener()
            val loggingListener = SLF4JQueryLoggingListener()
            listener.addListener(loggingListener)
            listener.addListener(DataSourceQueryCountListener())
            this.dataSource = ProxyDataSourceBuilder.create(dataSource)
                .name("MyDS")
                .multiline()
                .listener(listener)
                .build()
        }
    }
}
```

## 예제

사용법은 간단합니다. 저도 자세한 사용법은 모르지만 
`QueryCountHolder`를 통해서 쿼리가 날라간 횟수를 조회가 가능하고 
`clear()` 함수를 통해서 초기화가 가능하기 때무네 이를 적절히 사용해서 
쿼리의 호출 횟수가 예상과 일치하는지 테스트할 수 있습니다. 

```kotlin
 @Test
    @Transactional
    fun dbDataProxy() {
        teamRepository.save(Team(
        ).apply {
            players.add(Player("돌다리").also { it.team = this })
            players.add(Player().also { it.team = this })
            players.add(Player().also { it.team = this })
        })

        val insertCount = QueryCountHolder.getGrandTotal().insert

        insertCount shouldBe 4

        TestTransaction.flagForCommit()
        TestTransaction.end()
        TestTransaction.start()

        val team = teamRepository.findAll().first()
        val playerNames = team.players.map { it.name }

        val queryCount = QueryCountHolder.getGrandTotal().select
        
        queryCount shouldBe 1 // !!!!!!!!!!!!error should be 2 !!!!!!!!!!!!!!
    }
```

## 결론

단순히 n+1쿼리를 detecting 하는 것 외에도 로그라던지, 여러가지로 활용할 여지가 있습니다. 
우선, 저도 깊게 알아보지 않았으므로 bibliography에 링크를 첨부하도록 하겠습니다.

## Bibliography

[jdbc 로그 최선책](https://vladmihalcea.com/the-best-way-to-log-jdbc-statements/)

[datasource-proxy 깃헙 리파지토리](https://github.com/ttddyy/datasource-proxy)

[공식 문서](http://ttddyy.github.io/datasource-proxy/docs/current/user-guide/)

