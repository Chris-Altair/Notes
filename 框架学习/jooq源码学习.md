[TOC]

本文涉及的jooq版本为3.13

### 1. jooq事务原理

先抛出jooq嵌套事务的正确使用姿势之一：

```java
//外层事务
ctx.transaction((configuration) -> {
    		//configuration很关键
            DSL.using(configuration).insertInto(TABLE_1)
                    ...
                    .execute();
    		//内层事务
            DSL.using(configuration).transaction((configuration1 -> {
                DSL.using(configuration1).insertInto(TABLE_2)
                        ...
                        .execute();
                throw new RuntimeException("模拟异常");
            }));
        });
```

首先嵌套事务不是数据库的功能而是程序代码的功能，这里之所以嵌套事务生效关键在于内外层事务使用的是同一个configuration，

```java
DSL.using(configuration) //不同事务使用同一个配置对象,configuration和configuration1是一个对象
```

configuration实现：

```java
public class DefaultConfiguration implements Configuration {
    ...
    // These objects may be user defined and thus not necessarily serialisable
    private transient ConnectionProvider                connectionProvider;
    private transient ConnectionProvider                interpreterConnectionProvider;
    private transient ConnectionProvider                systemConnectionProvider;
    private transient TransactionProvider               transactionProvider;
    private transient TransactionListenerProvider[]     transactionListenerProviders;
    // [#7062] Apart from the possibility of containing user defined objects, the data
    //         map also contains the reflection cache, which isn't serializable (and
    //         should not be serialized anyway).
    private transient ConcurrentHashMap<Object, Object> data; //这个map是重点，连接和事务相关的信息存到这里
    ...
}
```

重点注意configuration的data，这个map在事务开始时会设置下面三个key：

```java
//枚举在Tool中
enum DataKey {
    /**
     * [#1629] The {@link Connection#getAutoCommit()} flag value before starting
     * a new transaction.
     */
    DATA_DEFAULT_TRANSACTION_PROVIDER_SAVEPOINTS,//savepoints堆栈，用来控制嵌套事务
    /**
     * [#1629] The {@link DefaultConnectionProvider} instance to be used during
     * the transaction.
     */
    DATA_DEFAULT_TRANSACTION_PROVIDER_CONNECTION,//DefaultConnectionProvider:存储连接池的jdbc连接，保证事务始终在一个连接中
    ...
}
enum BooleanDataKey {
        /**
         * [#1629] The {@link Connection#getAutoCommit()} flag value before starting
         * a new transaction.
         */
        DATA_DEFAULT_TRANSACTION_PROVIDER_AUTOCOMMIT,//开启事务前会关闭自动提交，事务提交或回滚再开启
    ...
}
```

事务实现是通过核心方法*dslContext.transactionResult(TransactionalCallable<T> transactional)*实现的，业务功能的lambda表达式最后都会包装成*TransactionalCallable<T>*形式，解析见注释

```java
public class DefaultDSLContext extends AbstractScope implements DSLContext, Serializable{
    @Override
    public <T> T transactionResult(TransactionalCallable<T> transactional) {
        return transactionResult0(transactional, configuration(), false);
    }

    private static <T> T transactionResult0(TransactionalCallable<T> transactional, Configuration configuration, boolean threadLocal) {

        // If used in a Java 8 Stream, a transaction should always be executed
        // in a ManagedBlocker context, just in case Stream.parallel() is called

        // The same is true for all asynchronous transactions, which must always
        // run in a ManagedBlocker context.
        return blocking(() -> {
            T result = null;
            //configuration.derive():根据当前configuration创建一个新的configuration对象
            //创建个事务上下文ctx
            DefaultTransactionContext ctx = new DefaultTransactionContext(configuration.derive());
            //获取ctx.config的transactionProvider
            //如果类型是NoTransactionProvider则new DefaultTransactionProvider(connectionProvider)
            //否则直接拿config里的
            TransactionProvider provider = ctx.configuration().transactionProvider();
            //创建事务监听器，监听器内部是TransactionListener[] listeners;具体实现看下面
            TransactionListeners listeners = new TransactionListeners(ctx.configuration());
            boolean committed = false;
            try {
                try {
                    listeners.beginStart(ctx);
                    //主要是savepoints.push入栈操作
                    provider.begin(ctx);
                }
                finally {
                    listeners.beginEnd(ctx);
                }
                //这个run执行的就是业务代码的lambda，可见业务代码取的config就是事务context的配置
                //事务内业务代码继续使用DSL.using(config)，这保证了事务内始终使用一个config
                //即保证了数据库连接和事务相关信息的连续性
                result = transactional.run(ctx.configuration());
                try {
                    listeners.commitStart(ctx);
                    //主要是savepoints.pop出栈操作
                    provider.commit(ctx);
                    committed = true;
                }
                finally {
                    listeners.commitEnd(ctx);
                }
            }
            // [#6608] [#7167] Errors are no longer handled differently
            catch (Throwable cause) {
                // [#8413] Avoid rollback logic if commit was successful (exception in commitEnd())
                if (!committed) {
                    if (cause instanceof Exception)
                        ctx.cause((Exception) cause);
                    else
                        ctx.causeThrowable(cause);
                    listeners.rollbackStart(ctx);
                    try {
                        provider.rollback(ctx);
                    }
                    // [#3718] Use reflection to support also JDBC 4.0
                    catch (Exception suppress) {
                        cause.addSuppressed(suppress);
                    }
                    listeners.rollbackEnd(ctx);
                }
                // [#6608] [#7167] Errors are no longer handled differently
                if (cause instanceof RuntimeException)
                    throw (RuntimeException) cause;
                else if (cause instanceof Error)
                    throw (Error) cause;
                else
                    throw new DataAccessException(committed
                        ? "Exception after commit"
                        : "Rollback caused"
                        , cause
                    );
            }
            return result;
        }, threadLocal).get();
    }
    ...
}
//TransactionListeners方法
TransactionListeners(Configuration configuration) {
        TransactionListenerProvider[] providers = configuration.transactionListenerProviders();
        listeners = new TransactionListener[providers.length];

        for (int i = 0; i < providers.length; i++)
            listeners[i] = providers[i].provide();
    }
```

由上面代码可见：

- 嵌套事务的实现是一个堆栈操作，先入后出
- 嵌套事务要控制始终在同一个jdbc连接上
- *TransactionProvider*和*TransactionListeners*很关键，这两个具体做了什么还需要进一步分析

