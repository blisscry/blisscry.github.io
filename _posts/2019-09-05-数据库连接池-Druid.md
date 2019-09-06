---

layout:     post
title:      数据库连接池
subtitle:   PPT文档
date:       2019-01-25
author:     skaleto
catalog: true
tags:
    - datasource

---

# 





我们知道，在java中，服务与数据库之间的交互需要借助jdbc来连接和操作数据库。一次数据库操作一般包含了如下步骤

```java
//1.加载驱动
Class.forName("com.mysql.jdbc.Driver");
//2.连接数据库URL
String url = "jdbc:mysql://localhost:3306/test?" + "user=root&password=root";
//3.获取数据库连接
conn = DriverManager.getConnection(url);
```

1. “DriverManager”检查并注册驱动程序。
2. 在驱动程序类中调用“connect(url…)”方法。
3. connect方法根据我们请求的“connUrl”，创建一个“Socket连接”，连接到IP为“your.database.domain”，默认端口3306的数据库。
4. 创建的Socket连接将被用来查询我们指定的数据库，并最终让程序返回得到一个结果。

但是数据库连接的创建非常耗时，当我们在需要高效查询的时候，数据库连接的时间显然是我们不希望耗费的。因此，有这样一种想法，我们提前加载并创建好几个连接，并放在某个地方管理起来，这样，当需要使用数据库连接的时候，直接拿来用就可以了。

想法非常实际，我们先来想想看，假如自己实现这个管理连接的东西，我们如何来设计？

首先，由于数据库存在多种多样，但是对每种数据库的操作大同小异，存在一些共同的部分，这种情况下，很容易能够想到使用模板方法模式。将一些共同的部分，比如连接的创建获取，连接池的大小配置等放在基类中完成。此外，我们需要一个数据结构来存放创建的连接，每个连接需要带上一些属性，例如创建时间、连接配置、超时时间等等。最简单地可以想到，使用一个ConcurrentHashMap来存放我们创建的连接的key-value。



我们先来看看现在比较主流的几种数据库连接池

### Tomcat jdbc

一个最简单的Tomcat jdbc使用方式如下

```java
private DataSource dataSource;

public void init() {
    PoolProperties poolProperties = new PoolProperties();
    // setDefault
    poolProperties.setInitialSize(3);
    poolProperties.setTimeBetweenEvictionRunsMillis(1000);
    poolProperties.setDefaultAutoCommit(true);
    poolProperties.setLogAbandoned(false);
    poolProperties.setMaxActive(15);
    poolProperties.setMinIdle(3);
    poolProperties.setTestOnBorrow(false);
    poolProperties.setTestOnReturn(false);

    dataSource = new DataSource(poolProperties);
}

public Connection getConnection() throws SQLException {
    return dataSource.getConnection();
}
```

我们跟进dataSource.getConnection中去

```java
public Connection getConnection() throws SQLException {
    if (pool == null)
        //初始化的时候，将创建一个ConnectionPool
        return createPool().getConnection();
    return pool.getConnection();
}

public ConnectionPool(PoolConfiguration prop) throws SQLException {
    //创建pool的主要方法
    init(prop);
}

protected void init(PoolConfiguration properties) throws SQLException {
        poolProperties = properties;

        //连接池大小各项配置校验
        if (properties.getMaxActive()<1) {
            log.warn(...);
            //DEFAULT_MAX_ACTIVE = 100;
            properties.setMaxActive(PoolProperties.DEFAULT_MAX_ACTIVE);
        }
        if (properties.getMaxActive()<properties.getInitialSize()) {
            log.warn(...);
            properties.setInitialSize(properties.getMaxActive());
        }
        if (properties.getMinIdle()>properties.getMaxActive()) {
            log.warn(...);
            properties.setMinIdle(properties.getMaxActive());
        }
        if (properties.getMaxIdle()>properties.getMaxActive()) {
            log.warn(...);
            properties.setMaxIdle(properties.getMaxActive());
        }
        if (properties.getMaxIdle()<properties.getMinIdle()) {
            log.warn(...);
            properties.setMaxIdle(properties.getMinIdle());
        }

        //make space for 10 extra in case we flow over a bit
    	//用于存放连接的busy队列是一个阻塞队列
        busy = new LinkedBlockingQueue<>();
    	//isFairQueue这个属性默认情况下是true
        if (properties.isFairQueue()) {
            //用于存放连接的idle队列是一个公平阻塞队列，等待线程按照先来后到的顺序获取连接
            idle = new FairBlockingQueue<>();
        } else {
            idle = new LinkedBlockingQueue<>();
        }

        //开启连接池清理者
        if (properties.isPoolSweeperEnabled()) {
            poolCleaner = new PoolCleaner(this, properties.getTimeBetweenEvictionRunsMillis());
            poolCleaner.start();
        }

        //create JMX MBean
        if (this.getPoolProperties().isJmxEnabled()) createMBean();

    	//如果在配置中有设置jdbc拦截器，会在这个地方通知各个拦截器连接池已被创建好
        PoolProperties.InterceptorDefinition[] proxies = getPoolProperties().getJdbcInterceptorsAsArray();
        for (int i=0; i<proxies.length; i++) {
            try {
                if (log.isDebugEnabled()) {
                    log.debug(...));
                }
                JdbcInterceptor interceptor = proxies[i].getInterceptorClass().newInstance();
                interceptor.setProperties(proxies[i].getProperties());
                interceptor.poolStarted(this);
            }catch (Exception x) {
                log.error("Unable to inform interceptor of pool start.",x);
                if (jmxPool!=null) jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.NOTIFY_INIT, getStackTrace(x));
                close(true);
                SQLException ex = new SQLException();
                ex.initCause(x);
                throw ex;
            }
        }

        //创建一个initialSize大小的数组用来存放数据库连接对象
        PooledConnection[] initialPool = new PooledConnection[poolProperties.getInitialSize()];
        try {
            for (int i = 0; i < initialPool.length; i++) {
                initialPool[i] = this.borrowConnection(0, null, null); //don't wait, should be no contention
            } //for

        } catch (SQLException x) {
            if (jmxPool!=null) jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.NOTIFY_INIT, getStackTrace(x));
            close(true);
            throw x;
        } finally {
            //将初始化创建好的连接放入空闲队列中
            for (int i = 0; i < initialPool.length; i++) {
                if (initialPool[i] != null) {
                    //这个地方捕获了异常但是没有处理我觉得不太妥
                    try {this.returnConnection(initialPool[i]);}catch(Exception x){/*NOOP*/}
                }
            }
        }

        closed = false;
    }

```

来看看borrowConnection方法

```java
private PooledConnection borrowConnection(int wait, String username, String password) throws SQLException {

        if (isClosed()) {
            throw new SQLException("Connection pool closed.");
        } //end if

        //get the current time stamp
        long now = System.currentTimeMillis();
        //see if there is one available immediately
    	//在初始化的时候，空闲队列里显然是没有连接存在的,这里的con为null
        PooledConnection con = idle.poll();
		
    	//这里是一个死循环等待
        while (true) {
            if (con!=null) {
                //configure the connection and return it
                PooledConnection result = borrowConnection(now, con, username, password);
                //null should never be returned, but was in a previous impl.
                if (result!=null) return result;
            }

            //if we get here, see if we need to create one
            //this is not 100% accurate since it doesn't use a shared
            //atomic variable - a connection can become idle while we are creating
            //a new connection
            //这里说不能保证到这边的时候所有连接一定都不是空闲的
            //这里的size是个AtomicInteger，保证多线程情况下计数永远正确
            if (size.get() < getPoolProperties().getMaxActive()) {
                //size自增1，但因为多线程情况下，有可能别的线程已经完成了自增，这个时候可能会超过max
                if (size.addAndGet(1) > getPoolProperties().getMaxActive()) {
                    //if we got here, two threads passed through the first if
                    size.decrementAndGet();
                } else {
                    //在初始化的时候，这里的username和password入参都是null，但是由于我们在poolProperties中配置了使用的jdbc driver，username，password等，这里的内部逻辑会直接从poolProperties中取出相应的配置，初始化driver并创建连接
                    //连接创建完成后，都会被默认放入busy队列
                    return createConnection(now, con, username, password);
                }
            }

            //假如没有空闲的连接，并且连接数也已经达到了上限，这个时候就需要等待
            long maxWait = wait;
            //入参wait为-1的时候，从poolProperties中取maxwait值，假如未配置或配置项小于0，那么最大等待时间将被设置为长整型的最大值0x7fffffffffffffffL
            if (wait==-1) {
                maxWait = (getPoolProperties().getMaxWait()<=0)?Long.MAX_VALUE:getPoolProperties().getMaxWait();
            }

            long timetowait = Math.max(0, maxWait - (System.currentTimeMillis() - now));
            waitcount.incrementAndGet();
            try {
                //尝试从空闲队列中取出一个连接
                con = idle.poll(timetowait, TimeUnit.MILLISECONDS);
            } catch (InterruptedException ex) {
                Thread.interrupted();//clear the flag, and bail out
                SQLException sx = new SQLException("Pool wait interrupted.");
                sx.initCause(ex);
                throw sx;
            } finally {
                waitcount.decrementAndGet();
            }
            
            //假如maxWait入参为0，则认为无需循环等待，抛出异常
            if (maxWait==0 && con == null) {
                throw new SQLException(...);
            }
            //假如连接没有拿到，且未超时，则循环等待
            if (con == null) {
                if ((System.currentTimeMillis() - now) >= maxWait) {
                    throw new SQLException(...);
                } else {
                    //no timeout, lets try again
                    continue;
                }
            }
        }
}
```

来看看returnConnection方法

```java
/**
 * Returns a connection to the pool
 * If the pool is closed, the connection will be released
 * If the connection is not part of the busy queue, it will be released.
 * If {@link PoolProperties#testOnReturn} is set to true it will be validated
 * @param con PooledConnection to be returned to the pool
 */
protected void returnConnection(PooledConnection con) {
        if (isClosed()) {
            release(con);
            return;
        }

        if (con != null) {
            try {
                con.lock();
                //从busy队列移除该连接
                if (busy.remove(con)) {

                    if (!shouldClose(con,PooledConnection.VALIDATE_RETURN)) {
                        con.setStackTrace(null);
                        con.setTimestamp(System.currentTimeMillis());
                        if (((idle.size()>=poolProperties.getMaxIdle()) && !poolProperties.isPoolSweeperEnabled()) || (!idle.offer(con))) {
                            //假如空闲队列已经达到上限，或者入队失败，则销毁这个连接
                            release(con);
                        }
                    } else {
                        release(con);
                    }
                } else {
                    release(con);
                }
            } finally {
                con.unlock();
            }
        }
}
```

至此，数据库连接池初始化完毕。

#### 连接池清理者

在我们分析代码的过程中，有一个PoolSweeper连接池清理者我们略过了，那么它是做什么用的呢？我们先来看它的创建过程

```java
protected class PoolCleaner extends TimerTask {
        protected ConnectionPool pool;
        protected long sleepTime;
        protected volatile boolean run = true;
        protected volatile long lastRun = 0;

    	//构造方法没啥好说的，默认运行间隔是30秒
        PoolCleaner(ConnectionPool pool, long sleepTime) {
            this.pool = pool;
            this.sleepTime = sleepTime;
            if (sleepTime <= 0) {
                log.warn(...);
                this.sleepTime = 1000 * 30;
            } else if (sleepTime < 1000) {
                log.warn(...);
            }
        }

        @Override
        public void run() {
            ConnectionPool pool = this.pool.get();
            if (pool == null) {
                stopRunning();
            } else if (!pool.isClosed()) {
                try {
                    if (pool.getPoolProperties().isRemoveAbandoned()
                            || pool.getPoolProperties().getSuspectTimeout() > 0)
                        pool.checkAbandoned();
                    if (pool.getPoolProperties().getMinIdle() < pool.idle
                            .size())
                        pool.checkIdle();
                    if (pool.getPoolProperties().isTestWhileIdle()) {
                        pool.testAllIdle(false);
                    } else if (pool.getPoolProperties().getMaxAge() > 0) {
                        pool.testAllIdle(true);
                    }
                } catch (Exception x) {
                    log.error("", x);
                }
            }
        }

        public void start() {
            registerCleaner(this);
        }

        public void stopRunning() {
            unregisterCleaner(this);
        }
}
```



```java
/**
 * checkAbandoned
 */
public void checkAbandoned() {
        try {
            if (busy.size()==0) return;
            Iterator<PooledConnection> locked = busy.iterator();
            int sto = getPoolProperties().getSuspectTimeout();
            while (locked.hasNext()) {
                PooledConnection con = locked.next();
                boolean setToNull = false;
                try {
                    con.lock();
                    //当前连接已经被放到空闲队列了，就继续即可
                    if (idle.contains(con))
                        continue;
                    long time = con.getTimestamp();
                    long now = System.currentTimeMillis();
                    //shouldAbandon方法检查busy队列长度与配置的最大活跃连接数比值是否大于配置的百分比。假如满足abandon要求，并且在队列中的时间已经超过了abandontimeout，则把当前连接从busy队列中移出并销毁
                    if (shouldAbandon() && (now - time) > con.getAbandonTimeout()) {
                        busy.remove(con);
                        abandon(con);
                        setToNull = true;
                    } else if (sto > 0 && (now - time) > (sto*1000)) {
                        //假如超过了suspect的时间，则给这个连接的suspect设为true，suspect起的作用后面分析
                        suspect(con);
                    } else {
                    }
                } finally {
                    con.unlock();
                    if (setToNull)
                        con = null;
                }
            }
        } catch (ConcurrentModificationException e) {
            ...
        } catch (Exception e) {
            ...
        }
}

/**
 * checkIdle
 */
public void checkIdle() {
        try {
            if (idle.size()==0) return;
            long now = System.currentTimeMillis();
            Iterator<PooledConnection> unlocked = idle.iterator();
            while ( (idle.size()>=getPoolProperties().getMinIdle()) && unlocked.hasNext()) {
                PooledConnection con = unlocked.next();
                boolean setToNull = false;
                try {
                    con.lock();
                    //检查是否在占用，如果在busy队列中，则不处理
                    if (busy.contains(con))
                        continue;
                    long time = con.getTimestamp();
                    //空闲时间已经大于连接的释放时间，并且连接池中的连接数大于最小空闲连接数，则将它释放掉
                    if ((con.getReleaseTime()>0) && ((now - time) > con.getReleaseTime()) && (getSize()>getPoolProperties().getMinIdle())) {
                        release(con);
                        idle.remove(con);
                        setToNull = true;
                    } else {
                    }
                } finally {
                    con.unlock();
                    if (setToNull)
                        con = null;
                }
            }
        } catch (ConcurrentModificationException e) {
        } catch (Exception e) {
        }

}

/**
 * testAllIdle
 */
public void testAllIdle(boolean checkMaxAgeOnly) {
        try {
            if (idle.isEmpty()) return;
            Iterator<PooledConnection> unlocked = idle.iterator();
            while (unlocked.hasNext()) {
                PooledConnection con = unlocked.next();
                try {
                    con.lock();
                    //同样，当前的idle对象
                    if (busy.contains(con))
                        continue;

                    boolean release;
                    //重连检查是否有效，若还是无效则销毁连接对象
                    if (checkMaxAgeOnly) {
                        release = !reconnectIfExpired(con);
                    } else {
                        release = !reconnectIfExpired(con) || !con.validate(PooledConnection.VALIDATE_IDLE);
                    }
                    if (release) {
                        idle.remove(con);
                        release(con);
                    }
                } finally {
                    con.unlock();
                }
            }
        } catch (ConcurrentModificationException e) {
        } catch (Exception e) {
        }
}

protected boolean reconnectIfExpired(PooledConnection con) {
    	//检查距离上一次连接是否已经超过失效时间，超过则尝试重连
        if (con.isMaxAgeExpired()) {
            try {
                con.reconnect();
                reconnectedCount.incrementAndGet();
                if ( isInitNewConnections() && !con.validate( PooledConnection.VALIDATE_INIT)) {
                    return false;
                }
            } catch(Exception e) {
                return false;
            }
        }
        return true;
}

```

#### 连接池拦截器

在初始化的过程中，还有一个初始化拦截器的操作，我们再来看这一段代码，假如我们在初始化的poolProperties中加入poolProperties.setJdbcInterceptors("org.apache.tomcat.jdbc.pool.interceptor.ConnectionState;org.apache.tomcat.jdbc.pool.interceptor.StatementFinalizer");

```java
PoolProperties.InterceptorDefinition[] proxies = getPoolProperties().getJdbcInterceptorsAsArray();
        for (int i=0; i<proxies.length; i++) {
            try {
              	//所有拦截器都是代理类，通过反射的方式创建实例
                JdbcInterceptor interceptor = proxies[i].getInterceptorClass().getConstructor().newInstance();
                interceptor.setProperties(proxies[i].getProperties());
                //这个地方通过反射构造器的方式创建了拦截器，但事实上拦截器实现了InvocationHandler接口，并没有用来做代理
                interceptor.poolStarted(this);
            }catch (Exception x) {
                log.error("Unable to inform interceptor of pool start.",x);
                if (jmxPool!=null) jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.NOTIFY_INIT, getStackTrace(x));
                close(true);
                SQLException ex = new SQLException();
                ex.initCause(x);
                throw ex;
            }
        }

@Override
public InterceptorDefinition[] getJdbcInterceptorsAsArray() {
        if (interceptors == null) {
            if (jdbcInterceptors==null) {
                interceptors = new InterceptorDefinition[0];
            } else {
                String[] interceptorValues = jdbcInterceptors.split(";");
                InterceptorDefinition[] definitions = new InterceptorDefinition[interceptorValues.length+1];
                //TrapException作为默认的第一个拦截器
                definitions[0] = new InterceptorDefinition(TrapException.class);
                for (int i=0; i<interceptorValues.length; i++) {
                    int propIndex = interceptorValues[i].indexOf('(');
                    int endIndex = interceptorValues[i].indexOf(')');
                    if (propIndex<0 || endIndex<0 || endIndex <= propIndex) {
                        definitions[i+1] = new InterceptorDefinition(interceptorValues[i].trim());
                    } else {
                        //这个地方的配置支持在括号中配置拦截器的各项属性，以(key=value,key=value)的形式跟在拦截器类名后方
                        String name = interceptorValues[i].substring(0,propIndex).trim();
                        definitions[i+1] = new InterceptorDefinition(name);
                        String propsAsString = interceptorValues[i].substring(propIndex+1, endIndex);
                        String[] props = propsAsString.split(",");
                        for (int j=0; j<props.length; j++) {
                            int pidx = props[j].indexOf('=');
                            String propName = props[j].substring(0,pidx).trim();
                            String propValue = props[j].substring(pidx+1).trim();
                            definitions[i+1].addProperty(new InterceptorProperty(propName,propValue));
                        }
                    }
                }
                interceptors = definitions;
            }
        }
        return interceptors;
}
```







### Durid

### C3p0

### Dbcp

### HikariCP



### 如何设计数据库连接池大小

https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing

