## 概述

我们知道，创建线程是一个昂贵的操作，在多线程场景下，如果有大量的小任务需要执行，并且频繁地创建和销毁线程，实际上会消耗大量的系统资源，往往创建和消耗线程所耗费的时间比执行任务的时间还长，所以，为了提高效率，可以用线程池。

类似的，在执行JDBC的增删改查的操作时，如果每一次操作都来一次打开连接，操作，关闭连接，那么创建和销毁JDBC连接的开销就太大了。为了避免频繁地创建和销毁JDBC连接，我们可以通过连接池（Connection Pool）复用已经创建好的连接。

JDBC连接池有一个标准的接口 `javax.sql.DataSource`，注意这个类位于Java标准库中，但仅仅是接口。要使用JDBC连接池，我们必须选择一个JDBC连接池的实现。常用的JDBC连接池有：

- HikariCP
- C3P0
- BoneCP
- Druid

## Druid

[首页 · alibaba/druid Wiki · GitHub](https://github.com/alibaba/druid/wiki/%E9%A6%96%E9%A1%B5)

## HikariCP

目前使用最广泛的是HikariCP。以下示例通过 HikariCP 创建一个`DataSource`实例，这个实例就是连接池：

```java
HikariConfig config = new HikariConfig();
// 设置数据库连接信息
config.setJdbcUrl("jdbc:mysql://localhost:3306/test");
config.setUsername("root");
config.setPassword("password");

// 配置连接池属性
config.addDataSourceProperty("connectionTimeout", "1000"); // 连接超时：1秒
config.addDataSourceProperty("idleTimeout", "60000"); // 空闲超时：60秒
config.addDataSourceProperty("maximumPoolSize", "10"); // 最大连接数：10

DataSource ds = new HikariDataSource(config);
```

注意创建DataSource也是一个非常昂贵的操作，所以通常DataSource实例总是作为一个全局变量存储，并贯穿整个应用程序的生命周期。获取`Connection`时，就直接从连接池中获取，就不需要再去直接创建连接 `ds.getConnection()`。

通过连接池获取连接时，并不需要指定JDBC的相关URL、用户名、口令等信息，因为这些信息已经存储在连接池内部了（创建HikariDataSource时传入的HikariConfig持有这些信息）。一开始，连接池内部并没有连接，所以，第一次调用 `ds.getConnection()` ，会迫使连接池内部先创建一个`Connection` ，再返回给客户端使用。当我们调用 `conn.close()` 方法时（在`try(resource){...}`结束处），不是真正“关闭”连接，而是释放到连接池中，以便下次获取连接时能直接返回。

因此，连接池内部维护了若干个 `Connection` 实例，如果调用 `ds.getConnection()`，就选择一个空闲连接，并标记它为“正在使用”然后返回，如果对 `Connection` 调用 `close()`，那么就把连接再次标记为“空闲”从而等待下次调用。这样一来，我们就通过连接池维护了少量连接，但可以频繁地执行大量的SQL语句。

通常连接池提供了大量的参数可以配置，例如，维护的最小、最大活动连接数，指定一个连接在空闲一段时间后自动关闭等，需要根据应用程序的负载合理地配置这些参数。此外，大多数连接池都提供了详细的实时状态以便进行监控。
