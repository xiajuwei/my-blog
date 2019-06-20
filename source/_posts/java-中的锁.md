---
title: java 中的锁
date: 2019-06-20 15:19:51
tags:
---
1.分布式锁Spring Integration（企业集成模式）

Spring Integration提供的全局锁目前为如下存储提供了实现：

•Gemfire
•JDBC
•Redis
•Zookeeper

它们使用相同的API抽象——这正是Spring最擅长的。这意味着，不论使用哪种存储，你的编码体验是一样的，有一天想更换实现，只需要修改依赖和配置就可以了，无需修改代码。

1.RedisLockRegistry

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-integration</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.integration</groupId>
      <artifactId>spring-integration-redis</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

2.ZookeeperLockRegistry 

    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-zookeeper</artifactId>
    </dependency>

锁的使用：


    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.data.redis.connection.RedisConnectionFactory;
    import org.springframework.integration.redis.util.RedisLockRegistry;
    
    @Configuration
    public class RedisLockConfiguration {
        @Bean
        public RedisLockRegistry redisLockRegistry(RedisConnectionFactory redisConnectionFactory) {
    
            RedisLockRegistry redisLockRegistry = new RedisLockRegistry(redisConnectionFactory, "spring-redis");
    
            return redisLockRegistry;
        }
    
    }

两种分布式锁都继承来LockRegistry。故可通过LockRegistry来申明变量，获取锁

    Lock lock = redisLockRegistry.obtain(lockKey);
    try {
        log.info("开始争抢锁 ");
        if (lock.tryLock(3, TimeUnit.SECONDS)) {
        //争抢到锁 
        log.info("争抢到锁 ");
         // do business
        }
    } catch (Exception e) {
        log.error("获取锁失败 ： ", e);
    } finally {
        lock.unlock();
    }

![api](java-中的锁\lock封装的常用api.png)


基于表记录
    要实现分布式锁，最简单的方式可能就是直接创建一张锁表，然后通过操作该表中的数据来实现了。当我们想要获得锁的时候，就可以在该表中增加一条记录，想要释放锁的时候就删除这条记录。
为了更好的演示，我们先创建一张数据库表，参考如下：

    CREATE TABLE `database_lock` (
        `id` BIGINT NOT NULL AUTO_INCREMENT,
        `resource` int NOT NULL COMMENT '锁定的资源',
        `description` varchar(1024) NOT NULL DEFAULT "" COMMENT '描述',
        PRIMARY KEY (`id`),
        UNIQUE KEY `uiq_idx_resource` (`resource`) 
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据库分布式锁表';

当我们想要获得锁时，可以插入一条数据：
    
    INSERT INTO database_lock(resource, description) VALUES (1, 'lock');
    
注意：在表database_lock中，resource字段做了唯一性约束，这样如果有多个请求同时提交到数据库的话，数据库可以保证只有一个操作可以成功（其它的会报错：ERROR 1062 (23000): Duplicate entry '1' for key 'uiq_idx_resource'），那么那么我们就可以认为操作成功的那个请求获得了锁。

当需要释放锁的时，可以删除这条数据：

    DELETE FROM database_lock WHERE resource=1;

这种实现方式非常的简单，但是需要注意以下几点：

这种锁没有失效时间，一旦释放锁的操作失败就会导致锁记录一直在数据库中，其它线程无法获得锁。这个缺陷也很好解决，比如可以做一个定时任务去定时清理。
这种锁的可靠性依赖于数据库。建议设置备库，避免单点，进一步提高可靠性。
这种锁是非阻塞的，因为插入数据失败之后会直接报错，想要获得锁就需要再次操作。如果需要阻塞式的，可以弄个for循环、while循环之类的，直至INSERT成功再返回。
这种锁也是非可重入的，因为同一个线程在没有释放锁之前无法再次获得锁，因为数据库中已经存在同一份记录了。想要实现可重入锁，可以在数据库中添加一些字段，比如获得锁的主机信息、线程信息等，那么在再次获得锁的时候可以先查询数据，如果当前的主机信息和线程信息等能被查到的话，可以直接把锁分配给它。

乐观锁
    系统认为数据的更新的大多数情况下就不会产生冲突的，只在数据库更新操作提交的时候才对数据作冲突检查，如果检测的结果出现了与预期数据不一致的情况，则返回失败信息。
乐观锁的优点比较明显，由于在检测数据冲突时并不依赖数据库本身的锁机制，不会影响请求的性能，当产生并发且并发量较小的时候只有少部分请求会失败。缺点是需要对表的设计增加额外的字段，增加了数据库的冗余，另外，当应用并发量高的时候，version值在频繁变化，则会导致大量请求失败，影响系统的可用性。我们通过上述sql语句还可以看到，数据库锁都是作用于同一行数据记录上，这就导致一个明显的缺点，在一些特殊场景，如大促、秒杀等活动开展的时候，大量的请求同时请求同一条记录的行锁，会对数据库产生很大的写压力。所以综合数据库乐观锁的优缺点，乐观锁比较适合并发量不高，并且写操作不频繁的场景。


**_`悲观锁`_**

除了可以通过增删操作数据库表中的记录以外，我们还可以借助数据库中自带的锁来实现分布式锁。在查询语句后面增加FOR UPDATE，数据库会在查询过程中给数据库表增加悲观锁，也称排他锁。当某条记录被加上悲观锁之后，其它线程也就无法再改行上增加悲观锁。

悲观锁，与乐观锁相反，总是假设最坏的情况，它认为数据的更新在大多数情况下是会产生冲突的。

在使用悲观锁的同时，我们需要注意一下锁的级别。MySQL InnoDB引起在加锁的时候，只有明确地指定主键(或索引)的才会执行行锁 (只锁住被选取的数据)，否则MySQL 将会执行表锁(将整个数据表单给锁住)。

在使用悲观锁时，我们必须关闭MySQL数据库的自动提交属性（参考下面的示例），因为MySQL默认使用autocommit模式，也就是说，当你执行一个更新操作后，MySQL会立刻将结果进行提交。

    mysql> SET AUTOCOMMIT = 0;
    Query OK, 0 rows affected (0.00 sec)

这样在使用FOR UPDATE获得锁之后可以执行相应的业务逻辑，执行完之后再使用COMMIT来释放锁。

我们不妨沿用前面的database_lock表来具体表述一下用法。假设有一线程A需要获得锁并执行相应的操作，那么它的具体步骤如下：

STEP1获取锁：SELECT * FROM database_lock WHERE id = 1 FOR UPDATE;。
STEP2执行业务逻辑。
STEP3释放锁：COMMIT。
如果另一个线程B在线程A释放锁之前执行STEP1，那么它会被阻塞，直至线程A释放锁之后才能继续。注意，如果线程A长时间未释放锁，那么线程B会报错，参考如下（lock wait time可以通过innodb_lock_wait_timeout来进行配置）：

    ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

上面的示例中演示了指定主键并且能查询到数据的过程（触发行锁），如果查不到数据那么也就无从“锁”起了。

如果未指定主键（或者索引）且能查询到数据，那么就会触发表锁，比如STEP1改为执行（这里的version只是当做一个普通的字段来使用，与上面的乐观锁无关）：

    SELECT * FROM database_lock WHERE description='lock' FOR UPDATE;

或者主键不明确也会触发表锁，又比如STEP1改为执行：

    SELECT * FROM database_lock WHERE id>0 FOR UPDATE;

注意，虽然我们可以显示使用行级锁（指定可查询的主键或索引），但是MySQL会对查询进行优化，即便在条件中使用了索引字段，但是否真的使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它有可能不会使用索引，在这种情况下InnoDB将使用表锁，而不是行锁。

在悲观锁中，每一次行数据的访问都是独占的，只有当正在访问该行数据的请求事务提交以后，其他请求才能依次访问该数据，否则将阻塞等待锁的获取。悲观锁可以严格保证数据访问的安全。但是缺点也明显，即每次请求都会额外产生加锁的开销且未获取到锁的请求将会阻塞等待锁的获取，在高并发环境下，容易造成大量请求阻塞，影响系统可用性。另外，悲观锁使用不当还可能产生死锁的情况。

