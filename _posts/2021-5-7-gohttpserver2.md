---
title: 改进Go server的性能
categories:
- Go
---


#### 1. 对于服务器的改进

上一篇实现的简易服务器，在数据量和qps较大的时候，会导致数据库宕机，无法正常工作。为此，我们作出以下改动：
1. 建立合适的索引，保证查询的高效。
2. 在读操作里引入redis。
3. 对于写操作用InnoDB的行锁机制来保证并发，这里我们目前的数据库操作都使用了索引的id，所以会使用行锁而非表锁。

我们选用了go-redis作为library。

```Go
import(
    "github.com/go-redis/redis/v8"
)
```


#### 2. 代码实现改动

对于读操作，我们首先通过`redis.Nil`检查Redis中是否存在缓存，如果存在则直接从redis中读取。否则从数据库中读取并且写入redis。

```Go
    rdb := connectRedisClient()
	defer rdb.Close()
	val, err := rdb.Get(ctx, params["id"]).Result()

	if(err == redis.Nil) { // not cached in redis
		db := connectDB()
		// ...
		// do something at db side

		// write into redis
		rdb.Set(ctx, params["id"], json, time.Minute)
		return
	} else if (err != nil) {
		fmt.Println(err)
		return
	} else { // read from redis
		w.WriteHeader(http.StatusOK)
		w.Write([]btye(val))
		return
	}
```

对于更新（包括update和delete），我们也要检查redis中是否存在该行的缓存，并且进行更新。
```Go
    params := mux.Vars(r)
	id, _ := strconv.Atoi(params["id"])
	val, err := rdb.Get(ctx, params["id"]).Result()
	if(err == nil) {
		rdb.Set(ctx, params["id"], json, time.Minute)
	} else if (err != redis.Nil) {
		fmt.Println(err)
	}
```

#### 3.测试

参考了[这个stackoverflow回答](https://stackoverflow.com/questions/33550878/jmeter-pull-paths-from-file-at-random)，先生成一个包含随机路径的文件，然后在jmeter里进行随机路径的访问压测。经测试，在表达到1000万行的情况下，读操作可以在3-5s的短时间内经受2000qps，写操作可以经受1000qps。但如果延长测试的时间到30s以上，则只能支持300-500qps左右。



#### 4.思考
##### 4.1 redis跟普通memcached的区别
1. redis支持多种数据类型，memcached则只有键值对的形式，且redis的键值对可以存储更大的数据
2. redis支持数据的持久化，也可以恢复(snapshot/AOF)
3. redis支持pub/sub，pipelining，Lua scripting等等

当然memcache仍然是简单强大的选择，只是在很多情况下redis更优。

##### 4.2 缓存和数据不一致

要解决缓存和db的数据不一致主要有以下方法：
1. cache-aside: 仅仅通过代码来操作数据库，在对数据库进行更新的时候，同步更新缓存。同时也可以对缓存的数据设置过期时间等等，因为缓存并不直接参与，所以叫做cache-aside。

2. write-through: 直接经过缓存进行操作，写数据的时候同时对缓存和db进行更新，两者在同一个事务里。同样还有read-through，读数据的时候直接通过缓存，让缓存决定是否访问db。类似地还有write-behind，这种策略在更新缓存之后，异步把更新的数据写入数据库，加长了不一致的时间，但是提高了性能。

3. 使用2-phase commit等一致性协议，缺点是比较复杂，维护成本也比较高。

以上的方法很多都只能保证eventual consistency，如果要求strict consistency，那么缓存原本的提升性能意义似乎就不大了。Consistency comes with costs.





