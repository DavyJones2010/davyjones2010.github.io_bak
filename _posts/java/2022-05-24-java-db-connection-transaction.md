---
layout: post
title: Java中常用的数据库连接池以及事务框架的原理探究
tags: [java, spring, spring-jdbc, database, connection, transaction]
---

# 思考1: Spring里各种事务隔离级别, 本质上是如何实现的?
很有意思的一个问题. 单个connection或者DB原生是无法实现的, 本质上是通过不同的connection来实现的.

[https://stackoverflow.com/questions/19526604/how-does-transaction-suspension-work-in-spring](https://stackoverflow.com/questions/19526604/how-does-transaction-suspension-work-in-spring)

> The point of suspending a transaction is to change the current transaction for a thread to a new one. This would NOT line up with the semantics of nested transactions because the new and suspended transactions are completely independent of each other. There is no connection-level API to support suspending transactions so this has to be done by using a different connection. If you are using JTA with Spring, this is done by the JTA transaction manager. If you are using DataSourceTransactionManager, you can look in the code and see that it will be saving off the current connection as a "suspended resource" and grabbing a new connection from the data source for the new transaction.


- [https://wiyi.org/physical-and-logical-transactions.html](https://wiyi.org/physical-and-logical-transactions.html)
- [https://wiyi.org/how-does-transaction-suspension-in-spring.html](https://wiyi.org/how-does-transaction-suspension-in-spring.html)

## 简化后的样例
### REQUIRES_NEW
- 新建事务，如果当前存在事务，则把当前事务挂起
- 这个方法会独立提交事务，不受调用者的事务影响，父级异常，它也是正常提交

```
// 假定为REQUIRE_NEW传播类型
// 1. 外部事务执行
Connection conn = getConnection(); // 这里是新建connection
Statement stmt = conn.createStatement();
stmt.execute("update xxx set xxx = xxx");

// 2. 到这里, 内部事务开始, 需要执行方法2, REQUIRE_NEW传播级别, 那本质上是新建了个connection. 在Spring里, 其实就是suspend(), 即将上一个conn从ThreadLocal里移除; 然后把conn2放入ThreadLocal里.
Connection conn2 = getConnection(); // 这个conn2与上边的conn是不同的连接
Statement stmt2 = conn.createStatement();
stmt2.execute("update xxx set xxx = xxx");
conn2.commit(); // 这里内部事务执行commit();
stmt2.close();
conn2.close();

// 3. 外部事务继续执行
stmt.execute("update xxx set xxx=xxx");
conn.commit();  // 这里外部事务执行commit();
stmt.close();
conn.close();
```

### PROPAGATION_REQUIRED

- 支持当前事务，如果当前没有事务，则新建事务
- 如果当前存在事务，则加入当前事务，合并成一个事务

```
// 假定为PROPAGATION_REQUIRED传播类型: 
// 1. 支持当前事务，如果当前没有事务，则新建事务
// 2. 如果当前存在事务，则加入当前事务，合并成一个事务

// 1. 外部事务执行
Connection conn = getConnection(); // 这里是新建connection
Statement stmt = conn.createStatement();
stmt.execute("update xxx set xxx = xxx");

// 2. 到这里, 内部事务开始, 需要执行方法2, PROPAGATION_MANDATORY 传播级别, 那本质上是复用了上一个连接.
Connection conn2 = conn; // 这个conn2与上边的conn是相同的连接
Statement stmt2 = conn.createStatement();
stmt2.execute("update xxx set xxx = xxx");
// conn2.commit(); // 这里无法做到子事务单独提交, 因为跟父事务已经合并成了一个事务.

// 3. 外部事务继续执行
stmt.execute("update xxx set xxx=xxx");
conn.commit();  // 这里整体事务执行commit();
stmt.close();
conn.close();
```

### NESTED

- 如果当前存在事务，它将会成为父级事务的一个子事务，方法结束后并没有提交，只有等父事务结束才提交
- 如果当前没有事务，则新建事务
- 如果它异常，父级可以捕获它的异常而不进行回滚，正常提交
- 但如果父级异常，它必然回滚，这就是和 `REQUIRES_NEW` 的区别


# 思考2: Druid连接池: 如何知道池中哪些connection在用, 哪些connection没在用?
## Druid实现:
通常我们使用标准的对象池, 会有如下方法:
1. Object borrow(): 即从池中借用对象, 从而该对象从池中移除, 不能再借给他人.
2. return(Object obj): 即把对象归还给池子, 以便该对象能被其他人复用.

但看标准的JDBC定义, 以及druid的实现, 发现只有如下定义, 即获取连接的定义.

```
public interface DataSource {
  Connection getConnection();
  Connection getConnection(String username, String password);
}
```
看Druid连接池的实现

1. getConnection(): 是从连接池中获取到一个可用的连接, 参见: com.alibaba.druid.pool.DruidDataSource#getConnectionInternal
2. **但为什么没有归还方法呢? 如果没有归还方法, 那连接池就没有意义了呀!!**
   1. 解惑: getConnection()拿到的是: DruidPooledConnection
   2. 在 DruidPooledConnection.close() 的时候, 其实本质上不是将连接关闭, 而是将连接归还回了连接池!!
       1. 这个设计还是挺巧妙的.
3. 那么真正销毁连接是在什么时候呢?
    1. 即整个连接池销毁的时候. DruidDataSource.close()

## 扩展: DBCP的数据库连接池是如何实现的?
核心代码参见: org.apache.commons.dbcp.BasicDataSource

1. 使用了工厂模式 
2. 依赖apache的commons.pool 架构整体比较优秀

### 从池中获取连接

```
org.apache.commons.dbcp.BasicDataSource#createDataSource {
	createConnectionPool(); // 1. 创建 GenericObjectPool 实例, 
  createPoolableConnectionFactory(); // 2. 创建ConnectionFactory, (这里用了工厂模式), Factory用来创建connection
  // 底层使用 org.apache.commons.dbcp.PoolableConnectionFactory#makeObject 来新建物理连接
  createDataSourceInstance(); // 3. 创建DbcpDataSource对象, 即该connectionpool
}
```

### 将连接归还回池子

```
// 1. 实际在池子里的connection是: 
org.apache.commons.dbcp.PoolableConnection#PoolableConnection

// 2. 也是在connection.close的时候将connection归还回池子:
public synchronized void close() throws SQLException {
	if (!isUnderlyingConectionClosed) {
	// Normal close: underlying connection is still open, so we
	// simply need to return this proxy to the pool
	try {
		_pool.returnObject(this); // XXX should be guarded to happen at most once
	}
}

```


# 思考3: CustomTransaction实现
## 背景
为了防止单个大事务, 导致锁表问题. 因此需要把大事务拆分成多个子事务.

# 思考4: TBTransactionImpl 实现
## 背景

1. 为了解决多个DataSource, 多数据源事务问题.
   1. 例如 DB1 (sql1), DB2 (sql2); 希望这俩数据库的SQL执行, 要不同时成功,  要不同时失败.
2. Best Efforts 1PC (One Phase Commit)
   1. 本质是采用的是1阶段提交方式.
3. 优缺点:
   1. 优点: 实现&使用简单.
   2. 缺点: 无法彻底保证一致性. 例如db1已经commit, db2 commit失败, 则整体回滚会失败掉!
4. 代码实现:

```java
    public void commit() throws SQLException {
        boolean var11 = false;

        try {
            var11 = true;
            TBConnection conn;
            Iterator i$;
            if (this.m_conn.size() > 1) {
                i$ = this.m_conn.values().iterator();

                while(i$.hasNext()) {
                    conn = (TBConnection)i$.next();
                    conn.judgeConnAvailable();
                }
            }

            i$ = this.m_conn.values().iterator();
			// 将db1, db2的connection都进行提交
            while(i$.hasNext()) {
                conn = (TBConnection)i$.next();
                conn.realCommit();
                conn.realClose();
            }

            var11 = false;
        } catch (Throwable var13) {
			// 将db1, db2的connection都进行回滚
            this.m_isCommitError = true;
            Iterator i$ = this.m_conn.values().iterator();

            while(i$.hasNext()) {
                TBConnection conn = (TBConnection)i$.next();
                try {
                    conn.realRollback();
                    conn.realClose();
                } catch (Throwable var12) {
                    log.fatal("Rollback ERROR:" + var12.getMessage(), var12);
                }
            }

            throw new SQLException(var13.getMessage());
        } finally {
            if (var11) {
                if (log.isWarnEnabled() && this.m_startHoldConnectionTime > 0L) {
                    long spendtime = System.currentTimeMillis() - this.m_startHoldConnectionTime;
                    if (spendtime >= (long)this.warnTime) {
                        log.warn("事务持续时间过长:" + spendtime, this.m_addr);
                    }
                }

                this.clear();
            }
        }

        this.clear();
    }
```
