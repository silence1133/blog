title: Spring boot使用技巧归纳
author: Silence
date: 2018-09-28 13:50:38
tags:
---
## 事务控制技巧

1. 在Transactional标注的方法里面处理一段事务逻辑提交之后，再处理非事务的逻辑

```
    @Transactional(rollbackFor = Exception.class)
    @Override
    public int insertUser(User user) {
        int result = userMapper.insertSelective(user);
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter(){
            @Override
            public void afterCommit() {
                System.out.println("Transaction is commit");
            }
        });
        return result;
    }
```

2. 手动控制事务。

```
@Autowired
private TransactionTemplate transactionTemplate;

@Test
public void transcationByhand(){
        OrderTransboundaryInfoDO insert1 = new OrderTransboundaryInfoDO();
        insert1.setOrderCode("zxyTest333");
        insert1.setReceiveRealName("zxy333");
        insert1.setUserPhone("zxy333");
        insert1.setUserId(2143124312412412412L);
        insert1.setReceiveIdentityCard("dfadsfads");
        insert1.setCreateTm(new Date());
        //只有excute内部的方法才有事务
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                orderTransboundaryInfoDAO.insertSelective(insert1);
                orderTransboundaryInfoDAO.insert(insert1);//此行代码会报错，回滚
            }
        });
    }

```

## 将service的方法快速异步化

配置线程池
```
@Configuration
@EnableAsync
@Slf4j
public class ExecutorConfiguration {

    @Bean
    public Executor asyncServiceExecutor() {
        log.info("start asyncServiceExecutor");
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //配置核心线程数
        executor.setCorePoolSize(5);
        //配置最大线程数
        executor.setMaxPoolSize(5);
        //配置队列大小
        executor.setQueueCapacity(2000);
        //配置线程池中的线程的名称前缀
        executor.setThreadNamePrefix("async-service-");

        // rejection-policy：当pool已经达到max size的时候，如何处理新任务
        // CALLER_RUNS：不在新线程中执行任务，而是由调用者所在的线程来执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        //执行初始化
        executor.initialize();
        return executor;
    }
}
```

异步service方法：

```
    @Async("asyncServiceExecutor")
    @Override
    public void testAsyncExecutor() {
        log.info("testAsyncExecutor start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("testAsyncExecutor done");
    }
```

参考：https://blog.csdn.net/boling_cavalry/article/details/79120268

## spring boot的监控

spring boot的监控很强大，spring boot监控项大部分默认都是关闭的。

如何开启：

```
management.server.port=8081
management.endpoints.web.exposure.include=*
```
则访问的url为：xxx:8081/actuator/xxx

比较有用的项：beans、env、threaddump、httptrace、mappings

具体参考：[官方文档](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#Spring%20Boot%20Actuator:%20Production-ready%20features)、[监控文档](https://docs.spring.io/spring-boot/docs/2.0.5.RELEASE/actuator-api//html/)

## session共享

spring-boot提供了基于redis的session共享功能

1、引入依赖

```
<dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-redis</artifactId>  
</dependency>  
<dependency>  
        <groupId>org.springframework.session</groupId>  
        <artifactId>spring-session-data-redis</artifactId>  
</dependency> 
```
2、配置redis链接

。。。

3、添加@EnableRedisHttpSession注解

```
@Configuration  
@EnableRedisHttpSession  
public class RedisSessionConfig {  
}  
```


## 实时修改线上日志级别

此项是spring boot监控提供的功能，通过访问loggers监控项，修改配置（发起http post请求修改）：

```
curl -H "Content-Type:application/json" -X POST --data "{\"configuredLevel\":\"INFO\"}" http://10.204.240.38:8082/loggers/org.apache.zookeeper
```

## 统一异常处理

http://blog.didispace.com/springbootexception/

## 使用@Scheduled创建定时任务

注意：此功能不适合分布式应用。

启动类添加@EnableScheduling注解

然后在具体service方法上标注@Scheduled即可

具体参见：[官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#scheduling-annotation-support-scheduled)

## 使用spring管理线程池

使用spring管理线程池的好处是：将线程池统一管理、将线程池的销毁交给spring。

- **配置ThreadPoolTaskExecutor：**


```
    @Bean
    public ThreadPoolTaskExecutor threadPoolTaskExecutor() {
        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        threadPoolTaskExecutor.setCorePoolSize(4);
        threadPoolTaskExecutor.setMaxPoolSize(20);
        //任务队列深度
        threadPoolTaskExecutor.setQueueCapacity(5000);
        threadPoolTaskExecutor.setThreadNamePrefix("z-manager-thread-");
        //待任务处理完之后才销毁线程池（类似于jdk线程池的shutdown和shutdownNow的区别）
        threadPoolTaskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        //处理不过来时的拒绝策略
        threadPoolTaskExecutor.setRejectedExecutionHandler((runnable, executor) -> {
            //
        });
        return threadPoolTaskExecutor;
    }
```

```
//关闭spring容器的时候会帮你关闭线程
o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'threadPoolTaskExecutor'
```
