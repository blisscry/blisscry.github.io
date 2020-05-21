# Sundry



## 并发容器中的问题

### 慎用ThreadLocal

考虑这样一个场景，在一个springboot项目中，接口中使用了ThreadLocal来保存用户的一些信息，但是出现下面这种蹊跷的现象，用户A请求接口得到了正常的信息，用户B接着用户A请求接口，但是用户B得到的却是用户A的信息，这是为什么呢？

原来，在接口中，开发者使用ThreadLocal来存放用户的信息，但是在接口处理完毕后，没有显式清空ThreadLocal中的数据。而springboot运行在tomcat容器中，tomcat是基于线程池的，因此，很有可能出现用户B复用的是用户A使用完的线程，导致数据被共享了。

绝大多数web容器都是使用线程池的，因此在使用ThreadLocal的时候，要特别注意有始有终。



### ConcurrentHashMap的size()方法可靠吗

来看下面一段代码

```java
    public String wrong() throws InterruptedException {
        ConcurrentHashMap<String, Long> concurrentHashMap = getData(ITEM_COUNT - 100);
        //初始900个元素
        log.info("init size:{}", concurrentHashMap.size());

        ForkJoinPool forkJoinPool = new ForkJoinPool(THREAD_COUNT);
        //使用线程池并发处理逻辑
        forkJoinPool.execute(() -> IntStream.rangeClosed(1, 10).parallel().forEach(i -> {
            //查询还需要补充多少个元素
            int gap = ITEM_COUNT - concurrentHashMap.size();
            log.info("gap size:{}", gap);
            //补充元素
            concurrentHashMap.putAll(getData(gap));
        }));
        //等待所有任务完成
        forkJoinPool.shutdown();
        forkJoinPool.awaitTermination(1, TimeUnit.HOURS);
        //最后元素个数会是1000吗？
        log.info("finish size:{}", concurrentHashMap.size());
        return "OK";
    }

    public static void main(String[] args) throws InterruptedException {
        //循环十次
        IntStream.rangeClosed(1,10).forEach(i->{
            try {
                new TestConcurrentHashMap().wrong();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
```

打出的日志如下图

```
:05:09.916 [main] INFO java100.TestConcurrentHashMap - init size:900
16:05:09.916 [ForkJoinPool-3-worker-9] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.916 [ForkJoinPool-3-worker-11] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.917 [ForkJoinPool-3-worker-4] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.917 [ForkJoinPool-3-worker-2] INFO java100.TestConcurrentHashMap - gap size:84
16:05:09.917 [ForkJoinPool-3-worker-8] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.917 [ForkJoinPool-3-worker-6] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.917 [ForkJoinPool-3-worker-13] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.917 [ForkJoinPool-3-worker-15] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.917 [ForkJoinPool-3-worker-9] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.917 [ForkJoinPool-3-worker-1] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.919 [main] INFO java100.TestConcurrentHashMap - finish size:1484
16:05:09.922 [main] INFO java100.TestConcurrentHashMap - init size:900
16:05:09.922 [ForkJoinPool-4-worker-9] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.923 [ForkJoinPool-4-worker-13] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.923 [ForkJoinPool-4-worker-9] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.923 [ForkJoinPool-4-worker-9] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.923 [ForkJoinPool-4-worker-15] INFO java100.TestConcurrentHashMap - gap size:100
16:05:09.923 [ForkJoinPool-4-worker-9] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.923 [ForkJoinPool-4-worker-6] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.923 [ForkJoinPool-4-worker-2] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.923 [ForkJoinPool-4-worker-11] INFO java100.TestConcurrentHashMap - gap size:0
16:05:09.927 [ForkJoinPool-4-worker-4] INFO java100.TestConcurrentHashMap - gap size:-100
16:05:09.928 [main] INFO java100.TestConcurrentHashMap - finish size:1200
```

从日志中发现，finish size存在不同的情况，那么问题出在哪里呢？

我们在代码中使用ConcurrentHashMap的size()来做流程控制，但是使用了 ConcurrentHashMap，不代表对它的多个操作之间的状态是一致的，是没有其他线程在操作它的，如果需要确保需要手动加锁。

诸如 size、isEmpty 和 containsValue 等聚合方法，在并发情况下可能会反映 ConcurrentHashMap 的中间状态。因此在并发情况下，这些方法的返回值只能用作参考，而不能用于流程控制。诸如 putAll 这样的聚合方法也不能确保原子性，在 putAll 的过程中去获取数据可能会获取到部分数据。



### CopyOnWriteArrayList

CopyOnWriteArrayList 虽然是一个线程安全的 ArrayList，但因为其实现方式是，每次修改数据时都会复制一份数据出来，所以有明显的适用场景，即读多写少或者说希望无锁读的场景。如果读写比例均衡或者有大量写操作的话，使用 CopyOnWriteArrayList 的性能会非常糟糕。

下面是分别使用copyOnWriteArrayList和synchronizedList进行读写测试的日志，可以看到在读场景下，copyOnWriteArrayList比synchronizedList更快，但是在写场景下copyOnWriteArrayList比synchronizedList小号的时间高了几百倍，因此要根据场景慎重选择合适的容器。

```
16:35:26.714 [main] INFO java100.TestCopyOnWriteList - StopWatch '': running time (millis) = 159
-----------------------------------------
ms     %     Task name
-----------------------------------------
00032  020%  Read:copyOnWriteArrayList
00127  080%  Read:synchronizedList

16:35:30.751 [main] INFO java100.TestCopyOnWriteList - StopWatch '': running time (millis) = 4035
-----------------------------------------
ms     %     Task name
-----------------------------------------
04021  100%  Write:copyOnWriteArrayList
00014  000%  Write:synchronizedList
```





## 连接池中的问题

对于一些面向连接的SDK，一般提供的不同类别的API

- 连接池和连接分离的 API：有一个 XXXPool 类负责连接池实现，先从其获得连接 XXXConnection，然后用获得的连接进行服务端请求，完成后使用者需要归还连接。通常，XXXPool 是线程安全的，可以并发获取和归还连接，而 XXXConnection 是非线程安全的。
- 内部带有连接池的 API：对外提供一个 XXXClient 类，通过这个类可以直接进行服务端请求；这个类内部维护了连接池，SDK 使用者无需考虑连接的获取和归还问题。一般而言，XXXClient 是线程安全的。
- 非连接池的 API：一般命名为 XXXConnection，以区分其是基于连接池还是单连接的。直接连接方式的 API 基于单一连接，每次使用都需要创建和断开连接，性能一般，且通常不是线程安全的。

举个🌰

```
JedisPool

是一种连接池和连接分离的API，可以通过它获得connection即连接
在springboot2.x以后默认的redis的链接池从JedisPool变成了LettucePool，Lettuce主要利用netty实现与redis的同步和异步通信。所以更安全和性能更好；
默认的数据库连接池也变更为HikariCP，HiKariCP 号称是业界跑得最快的数据库连接池。

MongoClient
MongoClient的父类Mongo内置了ServerSessionPool池，因此是属于第二种内部带有连接池的API

Jedis

单纯的Jedis事实上是一种非连接池的API，并且内部的读写以来于同一个OutputStream，如果存在多线程复用单个Jedis实例进行读写的情况，很可能会出现问题。
```





## Spring事务可能遇到的坑

### @Transactional 注解的生效条件

##### 注解标记的方法必须是public的

spring默认通过动态代理来实现AOP对方法进行增强，而private方法无法被代理到（因为cglib是通过继承的方式实现动态代理）

##### 必须通过代理过的类从外部调用目标方法才能生效

Spring 通过 AOP 技术对方法进行增强，要调用增强过的方法必然是调用代理后的对象，如下图如果直接使用this来调用，那么调用的将是自身，而不是代理后的对象

```java

@Service
@Slf4j
public class UserService {
    @Autowired
    private UserRepository userRepository;

    //一个公共方法供Controller调用，内部调用事务性的私有方法
    public int createUserWrong1(String name) {
        try {
            //这个地方的this调用事务会生效吗，不会
            this.createUserPrivate(new UserEntity(name));
        } catch (Exception ex) {
            log.error("create user failed because {}", ex.getMessage());
        }
        return userRepository.findByName(name).size();
    }

    //标记了@Transactional的private方法
    @Transactional
    private void createUserPrivate(UserEntity entity) {
        userRepository.save(entity);
        if (entity.getName().contains("test"))
            throw new RuntimeException("invalid username!");
    }

    //根据用户名查询用户数
    public int getUserCount(String name) {
        return userRepository.findByName(name).size();
    }
}
```



### @Transactional事务回滚的条件

@Transactional 注解的方法默认只会在出现RuntimeException或Error时回滚，如果方法中捕获了异常，那么事务回滚将不会生效

在 Spring 的 TransactionAspectSupport 里有个 invokeWithinTransaction 方法，里面就是处理事务的逻辑。可以看到，只有捕获到异常才能进行后续事务处理：

```java
try {
   // This is an around advice: Invoke the next interceptor in the chain.
   // This will normally result in a target object being invoked.
   retVal = invocation.proceedWithInvocation();
}
catch (Throwable ex) {
   // target invocation exception
   completeTransactionAfterThrowing(txInfo, ex);
   throw ex;
}
finally {
   cleanupTransactionInfo(txInfo);
}
```

```java
//除非手动进行事务回滚
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
//或者更改注解的配置捕获所有异常
@Transactional(rollbackFor = Exception.class)
```

### 事务传播配置

如果方法涉及多次数据库操作，并希望将它们作为独立的事务进行提交或回滚，那么我们需要考虑进一步细化配置事务传播方式，也就是 @Transactional 注解的 Propagation 属性。

```java
//支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择
@Transactional(propagation = Propagation.REQUIRED)
//支持当前事务，如果当前没有事务，就以非事务方式执行
@Transactional(propagation = Propagation.SUPPORTS)
//支持当前事务，如果当前没有事务，就抛出异常
@Transactional(propagation = Propagation.MANDATORY)
//新建事务，如果当前存在事务，把当前事务挂起
@Transactional(propagation = Propagation.REQUIRES_NEW)
//以非事务方式执行操作，如果当前存在事务，就把当前事务挂起
@Transactional(propagation = Propagation.NOT_SUPPORTED)
//以非事务方式执行，如果当前存在事务，则抛出异常。 
@Transactional(propagation = Propagation.NEVER)
//支持当前事务，如果当前事务存在，则执行一个嵌套事务，如果当前没有事务，就新建一个事务。 
@Transactional(propagation = Propagation.NESTED)
```

### 事务并发引起的三种情况

**1) Dirty Reads 脏读** 
一个事务正在对数据进行更新操作，但是更新还未提交，另一个事务这时也来操作这组数据，并且读取了前一个事务还未提交的数据，而前一个事务如果操作失败进行了回滚，后一个事务读取的就是错误数据，这样就造成了脏读。


**2) Non-Repeatable Reads 不可重复读** 
一个事务多次读取同一数据，在该事务还未结束时，另一个事务也对该数据进行了操作，而且在第一个事务两次次读取之间，第二个事务对数据进行了更新，那么第一个事务前后两次读取到的数据是不同的，这样就造成了不可重复读。


**3) Phantom Reads 幻像读** 
第一个数据正在查询符合某一条件的数据，这时，另一个事务又插入了一条符合条件的数据，第一个事务在第二次查询符合同一条件的数据时，发现多了一条前一次查询时没有的数据，仿佛幻觉一样，这就是幻像读。


**非重复度和幻像读的区别：**
非重复读是指同一查询在同一事务中多次进行，由于其他提交事务所做的修改或删除，每次返回不同的结果集，此时发生非重复读。

幻像读是指同一查询在同一事务中多次进行，由于其他提交事务所做的插入操作，每次返回不同的结果集，此时发生幻像读。

表面上看，区别就在于非重复读能看见其他事务提交的修改和删除，而幻像能看见其他事务提交的插入。 

 

### 事务隔离级别

**1) DEFAULT （默认）** 
这是一个PlatfromTransactionManager默认的隔离级别，使用数据库默认的事务隔离级别。另外四个与JDBC的隔离级别相对应。

**2) READ_UNCOMMITTED （读未提交）** 
这是事务最低的隔离级别，它允许另外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。 

**3) READ_COMMITTED （读已提交）** 
保证一个事务修改的数据提交后才能被另外一个事务读取，另外一个事务不能读取该事务未提交的数据。这种事务隔离级别可以避免脏读出现，但是可能会出现不可重复读和幻像读。 

**4) REPEATABLE_READ （可重复读）** 
这种事务隔离级别可以防止脏读、不可重复读，但是可能出现幻像读。它除了保证一个事务不能读取另一个事务未提交的数据外，还保证了不可重复读。

**5) SERIALIZABLE（串行化）** 
这是花费最高代价但是最可靠的事务隔离级别，事务被处理为顺序执行。除了防止脏读、不可重复读外，还避免了幻像读。 

 

**隔离级别解决事务并行引起的问题：**

![img](https://img2018.cnblogs.com/blog/1715232/201906/1715232-20190614173333961-597554027.png)



## 线程池中的问题

### 线程池的声明手动进行

Java 中的 Executors 类定义了一些快捷的工具方法，来帮助我们快速创建线程池。《阿里巴巴 Java 开发手册》中提到，禁止使用这些方法来创建线程池，而应该手动 new ThreadPoolExecutor 来创建线程池。

最典型的就是 newFixedThreadPool 和 newCachedThreadPool，可能因为资源耗尽导致 OOM 问题。

**newFixedThreadPool**

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

//其中的工作队列为LinkedBlockingQueue的默认实现，最大长度为MAX_VALUE
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}
```

虽然newFixedThreadPool的工作线程数量是固定的，但工作队列长度很长，当任务很多且处理较慢的情况下，很容易出现塞满工作队列的情况而导致OOM

**newCachedThreadPool**

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
//其中的工作队列SynchronousQueue没有容量，是无缓冲等待队列，是一个不存储元素的阻塞队列
```

newCachedThreadPool的工作线程max为Integer.MAX_VALUE，也就是说会任务很多的情况下会无节制地创建新的工作线程，导致出现线程打满的情况

```
[11:30:30.487] [http-nio-45678-exec-1] [ERROR] [.a.c.c.C.[.[.[/].[dispatcherServlet]:175 ] - Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: unable to create new native thread] with root cause
java.lang.OutOfMemoryError: unable to create new native thread
```



### 线程管理策略

##### 线程池默认的工作行为

- 不会初始化 corePoolSize 个线程，有任务来了才创建工作线程；
- 当核心线程满了之后不会立即扩容线程池，而是把任务堆积到工作队列中；
- 当工作队列满了后扩容线程池，一直到线程个数达到 maximumPoolSize 为止；
- 如果队列已满且达到了最大线程后还有任务进来，按照拒绝策略处理；
- 当线程数大于核心线程数时，线程等待 keepAliveTime 后还是没有任务需要处理的话，收缩线程到核心线程数。

##### 拒绝策略

- AbortPolicy，添加到线程池失败会抛出 RejectedExecutionException
- CallerRunsPolicy，这个要特别注意，这个策略会在提交任务失败时，将任务交还给提交任务的线程，因此很可能出现虽然使用了线程池，但实际上却是同步调用的情况

线程池的几个核心参数，包括核心线程数、最大线程数、线程回收策略、工作队列的类型、以及拒绝策略，需要合理评估，确保线程池的工作行为符合需求，一般都需要设置有界的工作队列和可控的线程数。

```
TIPS: Java 8 的 parallel stream 功能，可以让我们很方便地并行处理集合中的元素，其背后是共享同一个 ForkJoinPool，默认并行度是 CPU 核数 -1。对于 CPU 绑定的任务来说，使用这样的配置比较合适，但如果集合操作涉及同步 IO 操作的话（比如数据库操作、外部服务调用等），建议自定义一个 ForkJoinPool（或普通线程池）。
```





## 数据库索引的误区

### InnoDB的聚簇索引和二级索引

InnoDB 使用 B+ 树，既可以保存实际数据，也可以加速数据搜索，这就是聚簇索引。由于数据在物理上只会保存一份，所以包含实际数据的聚簇索引只能有一个。InnoDB 会自动使用主键（唯一定义一条记录的单个或多个字段）作为聚簇索引的索引键（如果没有主键，就选择第一个不包含 NULL 值的唯一列）。

为了实现非主键字段的快速搜索，就引出了二级索引，也叫作非聚簇索引、辅助索引，也是利用的 B+ 树的数据结构。

#### 额外创建二级索引的代价

- 维护代价。创建 N 个二级索引，就需要再创建 N 棵 B+ 树，新增数据时不仅要修改聚簇索引，还需要修改这 N 个二级索引。
- 空间代价。虽然二级索引不保存原始数据，但要保存索引列的数据，所以会占用更多的空间。
- 回表的代价。二级索引不保存原始数据，通过索引找到主键后需要再查询聚簇索引，才能得到我们要的数据。

#### 索引开销的最佳实践

- 无需一开始就建立索引，可以等到业务场景明确后，或者是数据量超过 1 万、查询变慢后，再针对需要查询、排序或分组的字段创建索引。创建索引后可以使用 EXPLAIN 命令，确认查询是否可以使用索引。
- 尽量索引轻量级的字段，比如能索引 int 字段就不要索引 varchar 字段。索引字段也可以是部分前缀，在创建的时候指定字段索引长度。针对长文本的搜索，可以考虑使用 Elasticsearch 等专门用于文本搜索的索引数据库。
- 尽量不要在 SQL 语句中 SELECT *，而是 SELECT 必要的字段，甚至可以考虑使用联合索引来包含我们要搜索的字段，既能实现索引加速，又可以避免回表的开销。



### 不是所有针对索引列的查询都能用上索引

#### 索引只能匹配列前缀

比如下面的 LIKE 语句，搜索 name 后缀为 name123 的用户无法走索引，执行计划的 type=ALL 代表了全表扫描：

```
EXPLAIN SELECT * FROM person WHERE NAME LIKE '%name123' LIMIT 100
```

![img](https://static001.geekbang.org/resource/image/e1/c9/e1033c6534938f8381fce051fb8ef8c9.png)

把百分号放到后面走前缀匹配，type=range 表示走索引扫描，key=name_score 看到实际走了 name_score 索引：

```
EXPLAIN SELECT * FROM person WHERE NAME LIKE 'name123%' LIMIT 100
```

![img](https://static001.geekbang.org/resource/image/95/5a/95074c69e68039738046fd4275c4d85a.png)

原因很简单，索引 B+ 树中行数据按照索引值排序，只能根据前缀进行比较。

#### 条件涉及函数操作无法走索引

比如搜索条件用到了 LENGTH 函数，肯定无法走索引：

```
EXPLAIN SELECT * FROM person WHERE LENGTH(NAME)=7
```

![img](https://static001.geekbang.org/resource/image/f1/08/f1eadcdd35b96c9f982115e528ee6808.png)

同样的原因，索引保存的是索引列的原始值，而不是经过函数计算后的值。如果需要针对函数调用走数据库索引的话，只能保存一份函数变换后的值，然后重新针对这个计算列做索引。

#### 联合索引只能匹配左边的列

```
EXPLAIN SELECT * FROM person WHERE SCORE>45678
```

![img](https://static001.geekbang.org/resource/image/77/17/77c946fcf49059d40673cf6075119d17.png)

虽然对 name 和 score 建了联合索引，但是仅按照 score 列搜索无法走索引，原因也很简单，在联合索引的情况下，数据是按照索引第一列排序，第一列数据相同时才会按照第二列排序。也就是说，如果我们想使用联合索引中尽可能多的列，查询条件中的各个列必须是联合索引中从最左边开始连续的列。



### 数据库基于成本决定是否走索引

MySQL查询数据可以直接在聚簇索引上进行全表扫描，也可以走二级索引扫描后到聚簇索引回表。那么，MySQL 到底是怎么确定走哪种方案的呢。

MySQL 在查询数据之前，会先对可能的方案做执行计划，然后依据成本决定走哪个执行计划。

```
这里的成本，包括 IO 成本和 CPU 成本
要计算全表扫描的代价需要两个信息：聚簇索引占用的页面数，用来计算读取数据的 IO 成本；表中的记录数，用来计算搜索的 CPU 成本。
```

在尝试通过索引进行 SQL 性能优化的时候，务必通过执行计划或实际的效果来确认索引是否能有效改善性能问题，否则增加了索引不但没解决性能问题，还增加了数据库增删改的负担。



## 数值精度、舍入、溢出问题

- 使用 BigDecimal 表示和计算浮点数，且务必使用字符串的构造方法来初始化 BigDecimal

- 浮点数的字符串格式化也要通过 BigDecimal 进行。

使用 equals 方法比较 1.0 和 1 这两个 BigDecimal

```java
System.out.println(new BigDecimal("1.0").equals(new BigDecimal("1")))
```

结果是 false，BigDecimal 的 equals 方法的注释中说明了原因，equals 比较的是 BigDecimal 的 value 和 scale，1.0 的 scale 是 1，1 的 scale 是 0，所以结果一定是 false。如果我们希望只比较 BigDecimal 的 value，可以使用 compareTo 方法。

那么，我们把值为 1.0 的 BigDecimal 加入 HashSet，然后判断其是否存在值为 1 的 BigDecimal，得到的结果是 false，怎么办呢

```
办法1.使用 TreeSet 替换 HashSet。TreeSet 不使用 hashCode 方法，也不使用 equals 比较元素，而是使用 compareTo 方法，所以不会有问题。
办法2.把 BigDecimal 存入 HashSet 或 HashMap 前，先使用 stripTrailingZeros 方法去掉尾部的零，比较的时候也去掉尾部的 0，确保 value 相同的 BigDecimal，scale 也是一致的
```

行数值运算时要小心溢出问题，虽然溢出后不会出现异常，但得到的计算结果是完全错误的。我们考虑使用 Math.xxxExact 方法来进行运算，在溢出时能抛出异常，更建议对于可能会出现溢出的大数运算使用 BigInteger 类。





## 集合类的问题

### Arrays.asList

- 不能直接使用 Arrays.asList 来转换基本类型数组

  ```java
  int[] arr = {1, 2, 3};
  List list = Arrays.asList(arr);
  ```

  上述代码执行后，list中的元素为一个1x3的二维数组，这是因为自动装箱不支持装箱int数组，除非变成Integer数组

- Arrays.asList 返回的 List 不支持增删操作

  ```java
  String[] arr = {"1", "2", "3"};
  List list = Arrays.asList(arr);
  arr[1] = "4";
  try {
      list.add("5");
  } catch (Exception ex) {
      ex.printStackTrace();
  }
  ```

  上述代码执行后会抛出异常，那是因为Arrays.asList返回的类型是 Arrays 的内部类 ArrayList

- 对原始数组的修改会影响到我们获得的那个 List

  上例中arr[1]="4"修改后，得到的list也会变更，那是因为Arrays.asList直接引用了原始的数组，并没有创建新的



### Sublist的坑

Sublist报OOM或ConcurrentModificationException

```java

public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    protected transient int modCount = 0;
    //...

  public List<E> subList(int fromIndex, int toIndex) {
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList(this, offset, fromIndex, toIndex);
  }

  private class SubList extends AbstractList<E> implements RandomAccess {
    private final AbstractList<E> parent;
    private final int parentOffset;
    private final int offset;
    int size;

    SubList(AbstractList<E> parent,
          int offset, int fromIndex, int toIndex) {
          //...
    }

        public E set(int index, E element) {
            rangeCheck(index);
            checkForComodification();
            return l.set(index+offset, element);
        }

    public ListIterator<E> listIterator(final int index) {
                checkForComodification();
                ...
    }

    private void checkForComodification() {
        if (ArrayList.this.modCount != this.modCount)
            throw new ConcurrentModificationException();
    }
    ...
  }
}
```

从上面的代码看到，遍历 SubList 的时候会先获得迭代器，比较原始 ArrayList modCount 的值和 SubList 当前 modCount 的值，因此当我们对原始list进行操作后增加元素后，再对sublist进行操作，是有可能会报ConcurrentModificationException的。



