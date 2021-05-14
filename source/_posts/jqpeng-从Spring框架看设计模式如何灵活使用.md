---
title: 从Spring框架看设计模式如何灵活使用
tags: ["java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2021-02-08 18:05
---
文章作者:jqpeng
原文链接: [从Spring框架看设计模式如何灵活使用](https://www.cnblogs.com/xiaoqi/p/spring-design-pattern.html)

## Singleton 单例模式

单例模式是确保每个应用程序只存在一个实例的机制。默认情况下，Spring将所有bean创建为单例。

![单例模式](https://gitee.com/jadepeng/pic/raw/master/pic/2021/2/8/1612778162513.png)

你用@Autowired获取的bean，全局唯一。


    @RestController
    public class LibraryController {
        
        @Autowired
        private BookRepository repository;
    
        @GetMapping("/count")
        public Long findCount() {
            System.out.println(repository);
            return repository.count();
        }
    }


## 工厂方法模式

Spring 定义了BeanFactory接口，抽象对象容器：


    public interface BeanFactory {
    
        getBean(Class<T> requiredType);
        getBean(Class<T> requiredType, Object... args);
        getBean(String name);
    
        // ...
    ]


每一个`getBean` 方法其实就是一个工厂方法。

## 代理模式(Proxy)

![代理模式](https://gitee.com/jadepeng/pic/raw/master/pic/2021/2/8/1612778397869.png)

在Spring中，对于事务，我们可以加一个`@Transactional`注解，


    @Service
    public class BookManager {
        
        @Autowired
        private BookRepository repository;
    
        @Transactional
        public Book create(String author) {
            System.out.println(repository.getClass().getName());
            return repository.create(author);
        }
    }


Spring框架，通过AOP做Proxy。

## Decorator装饰器模式

`Spring` 中的`TransactionAwareCacheDecorator` 就做了对`Cache` 的包装：


    public interface Cache {
        String getName();
    
        Object getNativeCache();
    
        @Nullable
        Cache.ValueWrapper get(Object var1);
    
        @Nullable
        <T> T get(Object var1, @Nullable Class<T> var2);
    
        @Nullable
        <T> T get(Object var1, Callable<T> var2);
    
        void put(Object var1, @Nullable Object var2);
    }
    


`TransactionAwareCacheDecorator` 实现了`Cache`接口，构造时传入一个`targetCache`，在调用`put`等方法时，增加了自己装饰逻辑在里面。


    
    public class TransactionAwareCacheDecorator implements Cache {
        private final Cache targetCache;
    
        public TransactionAwareCacheDecorator(Cache targetCache) {
            Assert.notNull(targetCache, "Target Cache must not be null");
            this.targetCache = targetCache;
        }
    
        public void put(final Object key, @Nullable final Object value) {
            if (TransactionSynchronizationManager.isSynchronizationActive()) {
                TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                    public void afterCommit() {
                        TransactionAwareCacheDecorator.this.targetCache.put(key, value);
                    }
                });
            } else {
                this.targetCache.put(key, value);
            }
    
        }
    


### 最佳实践

- 装饰模式是继承的有力补充。相比于继承，装饰模式可以增加代码的可维护性、扩展性、复用性。在一些情况下装饰模式可以替代继承，解决类膨胀问题。
- 装饰模式有利于程序的可扩展性。在一个项目中，有很多因素考虑不周，特别是业务的变更。通过装饰模式重新封装一个装饰类，可以避免修改继承体系中的中间类，而是使用装饰类修饰中间类，这样原有的程序没有变更，通过扩展完成了这次变更。


## 组合模式(Composite)

Spring actuate 提供`HealthIndicator`, 用于监控服务健康状态。


    @FunctionalInterface
    public interface HealthIndicator {
    
        /**
         * Return an indication of health.
         * @return the health for
         */
        Health health();
    
    }
    


实现类里，有一个`CompositeHealthIndicator`, 可以`add`多个`HealthIndicator`,放入`indicators`里，最后返回`health`时，聚合所有`indicators`的`Health`。


    
    public class CompositeHealthIndicator implements HealthIndicator {
    
        private final Map<String, HealthIndicator> indicators;
    
        private final HealthAggregator healthAggregator;
    
        /**
         * Create a new {@link CompositeHealthIndicator}.
         * @param healthAggregator the health aggregator
         */
        public CompositeHealthIndicator(HealthAggregator healthAggregator) {
            this(healthAggregator, new LinkedHashMap<>());
        }
    
        /**
         * Create a new {@link CompositeHealthIndicator} from the specified indicators.
         * @param healthAggregator the health aggregator
         * @param indicators a map of {@link HealthIndicator}s with the key being used as an
         * indicator name.
         */
        public CompositeHealthIndicator(HealthAggregator healthAggregator,
                Map<String, HealthIndicator> indicators) {
            Assert.notNull(healthAggregator, "HealthAggregator must not be null");
            Assert.notNull(indicators, "Indicators must not be null");
            this.indicators = new LinkedHashMap<>(indicators);
            this.healthAggregator = healthAggregator;
        }
    
        public void addHealthIndicator(String name, HealthIndicator indicator) {
            this.indicators.put(name, indicator);
        }
    
        @Override
        public Health health() {
            Map<String, Health> healths = new LinkedHashMap<>();
            for (Map.Entry<String, HealthIndicator> entry : this.indicators.entrySet()) {
                healths.put(entry.getKey(), entry.getValue().health());
            }
            return this.healthAggregator.aggregate(healths);
        }
    
    }


* * *

感谢您的认真阅读。

如果你觉得有帮助，欢迎点赞支持！

不定期分享软件开发经验，欢迎关注作者, 一起交流软件开发：

