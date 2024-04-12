---
layout: mypost
title: Flink自定义sink实现从mysql同步数据到redis
categories: [FlinkCDC, JAVA]
---

## Flink 实现 Mysql to Redis

### pom.xml

````xml
		<dependency>
			<groupId>org.apache.flink</groupId>
			<artifactId>flink-connector-redis_2.11</artifactId>
			<version>1.1.5</version>
		</dependency>
````

大多数情况都用这个包

### 核心实现  

非常简单

````java
        //创建一个jedis连接配置
        FlinkJedisPoolConfig config = new FlinkJedisPoolConfig.Builder()
                .setHost("127.0.0.1")
                .setPort(6379)
                .build();

        userStream.addSink(new RedisSink<UserVo>(config,new MyRedisMapper()));
````

### 完整代码

````java
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 从数据表读取数据
        tableEnv.executeSql(
                "CREATE TABLE `user`("
                        + "  `id` INT NOT NULL,"
                        + "  `unique_id` STRING NOT NULL,"
                        + "  `username` STRING NOT NULL,"
                        + "  `c_username` STRING NOT NULL,"
                        + "  PRIMARY KEY (`id`) NOT ENFORCED "
                        + ") WITH ("
                        + " 'connector' = 'mysql-cdc', "
                        + " 'hostname' = '127.0.0.1',"
                        + " 'port' = '3306', "
                        + " 'username' = 'root', "
                        + "  'password' = '123456', "
                        + " 'database-name' = 'mytest', "
                        + " 'table-name' = 'user' "
                        + ")"
        );
        Table resultTable = tableEnv.from("user").select($("id"), $("unique_id"), $("username"), $("c_username")).filter($("id").isEqual(2));
        DataStream<Row> dataStream = tableEnv.toChangelogStream(resultTable);

        DataStream<UserVo> userStream = dataStream.map(new MapFunction<Row, UserVo>() {
            @Override
            public UserVo map(Row value) throws Exception {
                return new UserVo(
                        value.getField(1).toString(), 
                        value.getField(2).toString()
                );
            }
        });


        //创建一个jedis连接配置
        FlinkJedisPoolConfig config = new FlinkJedisPoolConfig.Builder()
                .setHost("127.0.0.1")
                .setPort(6379)
                .build();

        userStream.addSink(new RedisSink<UserVo>(config,new MyRedisMapper()));
        env.execute("Flink mysql to redis");


````

MyRedisMapper

````java
import lombok.extern.slf4j.Slf4j;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommand;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommandDescription;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisMapper;

@Slf4j
public class MyRedisMapper implements RedisMapper<UserVo> {

    @Override
    //getCommandDescription()方法主要是返回当前Redis操作命令的描述，
    //我们当前想要做的操作是想要把当前每一个用户的访问事件这个数据写入到Redis保存
    public RedisCommandDescription getCommandDescription() {
        //希望把所有用户的访问信息都保存在一张表里面，接下来要操作的是一张hash表
        //第一个参数是像Redis里面一张hash表去写入的命令，第二个参数是表的名称
        return new RedisCommandDescription(RedisCommand.HSET,"userMap");
    }

    @Override
    //针对每一个用户去写入，相当于同一个用户的数据要不停的更新
    //当前的key是user
    public String getKeyFromData(UserVo user) {
        return user.getUniqueId();
    }

    @Override
    //还需要定义当前的data是什么
    public String getValueFromData(UserVo user) {
        return user.getUsername();

    }
}
````

UserVo

````java
  @Data
  public class UserVo {
      public String uniqueId;
      public String username;

      public UserVo(String uniqueId, String username) {
          this.uniqueId = uniqueId;
          this.username = username;
      }
  }
````

## 局限性

我们点开 org.apache.flink.streaming.connectors.redis.common.mapper.RedisMapper

发现
````java
public interface RedisMapper<T> extends Function, Serializable {

	/**
	 * Returns descriptor which defines data type.
	 *
	 * @return data type descriptor
	 */
	RedisCommandDescription getCommandDescription();

	/**
	 * Extracts key from data.
	 *
	 * @param data source data
	 * @return key
	 */
	String getKeyFromData(T data);

	/**
	 * Extracts value from data.
	 *
	 * @param data source data
	 * @return value
	 */
	String getValueFromData(T data);
}
````

发现他没有关于过期时间的设置，这个不能忍

如果我们想要实现 setex 就抓瞎了， 只能自己动手丰衣足食

## 自定义redisSink

RedisCustomSink
````java
import com.myproject.api.logic.impl.flink.UserVo;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;
import redis.clients.jedis.Jedis;

public class RedisCustomSink extends RichSinkFunction<UserVo> {
    private transient Jedis jedis;
    private String redisHost;
    private int redisPort;
    private int expirySeconds;
    private String keyPrefix;

    public RedisCustomSink(String redisHost, int redisPort, int expirySeconds, String keyPrefix) {
        this.redisHost = redisHost;
        this.redisPort = redisPort;
        this.expirySeconds = expirySeconds;
        this.keyPrefix = keyPrefix;

    }

    @Override
    public void open(org.apache.flink.configuration.Configuration parameters) throws Exception {
        super.open(parameters);
        jedis = new Jedis(redisHost, redisPort);
    }

    @Override
    public void close() throws Exception {
        jedis.close();
        super.close();
    }

    @Override
    public void invoke(UserVo vo, Context context) {
        jedis.setex(this.keyPrefix+vo.getUniqueId(), expirySeconds, vo.getUsername());
    }
}
````

完整代码：

````java
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // 从数据表读取数据
        tableEnv.executeSql(
                "CREATE TABLE `user`("
                        + "  `id` INT NOT NULL,"
                        + "  `unique_id` STRING NOT NULL,"
                        + "  `username` STRING NOT NULL,"
                        + "  `c_username` STRING NOT NULL,"
                        + "  PRIMARY KEY (`id`) NOT ENFORCED "
                        + ") WITH ("
                        + " 'connector' = 'mysql-cdc', "
                        + " 'hostname' = '127.0.0.1',"
                        + " 'port' = '3306', "
                        + " 'username' = 'root', "
                        + "  'password' = '123456', "
                        + " 'database-name' = 'mytest', "
                        + " 'table-name' = 'user' "
                        + ")"
        );
        Table resultTable = tableEnv.from("user").select($("id"), $("unique_id"), $("username"), $("c_username")).filter($("id").isEqual(2));
        DataStream<Row> dataStream = tableEnv.toChangelogStream(resultTable);

        // 变更stream结构
        DataStream<UserVo> userStream = dataStream.map(new MapFunction<Row, UserVo>() {
            @Override
            public UserVo map(Row value) throws Exception {
                return new UserVo(
                        value.getField(1).toString(),
                        value.getField(2).toString()
                );
            }
        });

        userStream.addSink(new RedisCustomSink("localhost", 6379, 3600, "user_")); // 设置过期时间为3600秒
        env.execute("Flink mysql to redis");

````

## 自定义redis操作

上例子中, 我把invoke的操作逻辑写死了，如果这会我想实现一个hset，只能在写一个sink
````java
    @Override
    public void invoke(UserVo vo, Context context) {
        jedis.setex(this.keyPrefix+vo.getUniqueId(), expirySeconds, vo.getUsername());
    }
````

如果我希望redis操作，作为参数传入sink 改怎么做呢？

这通常通过使用接口或者函数式接口来实现。我将创建一个接口 `RedisOperation` 作为一个函数式接口，然后修改 `RedisCustomSink` 来接受一个 `RedisOperation` 实例。这将允许你在添加 sink 时，定义如何与 Redis 进行交互。

首先，定义 `RedisOperation` 接口：  
````java
@FunctionalInterface
public interface RedisOperation {
    void execute(Jedis jedis, UserVo vo, String keyPrefix, int expirySeconds);
}
````

然后，修改 `RedisCustomSink` 来接受一个 `RedisOperation`：

````java
public class RedisCustomSink extends RichSinkFunction<UserVo> {
    private final transient RedisOperation redisOperation;
    private transient Jedis jedis;
    private final String redisHost;
    private final int redisPort;
    private final int expirySeconds;
    private final String keyPrefix;

    public RedisCustomSink(String redisHost, int redisPort, int expirySeconds, String keyPrefix, RedisOperation redisOperation) {
        this.redisHost = redisHost;
        this.redisPort = redisPort;
        this.expirySeconds = expirySeconds;
        this.keyPrefix = keyPrefix;
        this.redisOperation = redisOperation;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        this.jedis = new Jedis(this.redisHost, this.redisPort);
        // this.jedis.auth("your password");
    }

    @Override
    public void invoke(UserVo vo, Context context) {
        this.redisOperation.execute(this.jedis, vo, this.keyPrefix, this.expirySeconds);
    }

    @Override
    public void close() throws Exception {
        if (this.jedis != null) {
            this.jedis.close();
        }
        super.close();
    }
}
````

最后，你可以在添加 sink 时，定义如何与 Redis 进行交互：

````java
    userStream.addSink(new RedisCustomSink("localhost", 6379, 10, "user_", 
    (jedis, vo, keyPrefix, expirySeconds) -> {
        jedis.hset(keyPrefix + vo.getUniqueId(), "username", vo.getUsername());
    }));
````