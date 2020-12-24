---
layout: post
title: redis进阶之分布式锁
date: 2020-12-26 20:00:00
tags: 
- redis
categories:
- redis
---

---

写过一个单节点版的redis分布式锁，源码在这里 https://github.com/IBM/distributed-lock-spring-boot-starter

---

官方文档直接给了答案 https://redis.io/topics/distlock

## 单节点redis上实现分布式锁

其实很简单，上锁就是`set nx px`, 解锁就是get+del现在一个lua脚本里就行。

这里给出java版本的代码

```java
    // 加锁
    @Override
    public boolean initValWithTtl(String key, String value, long ttl, TimeUnit timeUnit) {
        try {
            return lockRedisTemplate.execute(
                    (RedisCallback<Boolean>) connection -> (
                            (StringRedisConnection) connection).set(key, value, Expiration.from(ttl, timeUnit), RedisStringCommands.SetOption.SET_IF_ABSENT
                    )
            );
        } catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    // 解锁
    @Override
    public boolean del(String key, String value) {
        try {
            String del = "if (redis.call('get', KEYS[1]) == ARGV[1]) "
                    + " then "
                    + " redis.call('del', KEYS[1]) "
                    + " return true; "
                    + " else "
                    + " return false; "
                    + " end; ";
            DefaultRedisScript<Boolean> redisScript = new DefaultRedisScript<>(del);
            redisScript.setResultType(Boolean.class);
            return lockRedisTemplate.execute(redisScript, Collections.singletonList(key), value);
        } catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    // 锁续约
    @Override
    public boolean extendTtl(String key, String value, long ttl, TimeUnit timeUnit){
        try {
            String extendTtl = "if (redis.call('get', KEYS[1]) == ARGV[1]) "
                    + " then "
                    + " redis.call('pexpire', KEYS[1], ARGV[2]); "
                    + " return true;"
                    + " else "
                    + " return false; "
                    + " end;";
            DefaultRedisScript<Boolean> redisScript = new DefaultRedisScript<>(extendTtl);
            redisScript.setResultType(Boolean.class);
            return lockRedisTemplate.execute(redisScript, Collections.singletonList(key), value, String.valueOf(timeUnit.toMillis(ttl)));
        } catch (Exception e){
            if (!(e.getCause().getCause() instanceof InterruptedException)){
                e.printStackTrace();
            }
            return false;
        }
    }
```

需要注意的几点：

- 上锁的key必须是全局唯一的，value也一样，而且value必须是线程独有的： key可以锁名称，value可以ip加线程id加时间戳（其实value设一个随机值也可以，但你得在threadlocal里保存这个随机值）
- 锁有过期时间，为了防止死锁
- 过期时间到了但是需要上锁的业务代码还没跑完怎么办，加一个租约机制，用一个守护线程去不断的续约（ScheduledThreadPoolExecutor）
- 代码可以直接套用Java的aqs，但注意：
  单实例的aqs，头节点永远是占有锁的节点，所以它的后驱节点永远在自旋。其他节点都可以放心休息。而多实例aqs，肯定会有实例的等待队列的头节点得不到锁，从而错误的进入了休息，这样这个队列就永远不会被唤醒。因此，要解决问题，只需要保持每个实例的队列的头节点即使不是占有锁的节点，也得保证其后驱节点永远在自旋即可。
  简单改一下aqs中的一行就能实现：
    ```java
    final boolean acquireQueued(final Node node, int arg) {
            boolean failed = true;
            try {
                boolean interrupted = false;
                for (;;) {
                    final Node p = node.predecessor();
                    if (p == head && tryAcquire(arg)) {
                        setHead(node);
                        p.next = null;
                        failed = false;
                        return interrupted;
                    }
                    // 就是这一行
                    if (p != head && shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()) {
                        interrupted = true;
                    }
                }
            } finally {
                if (failed) {
                    cancelAcquire(node);
                }
            }
        }
    ```

## 分布式redis环境的分布式锁

### 为什么基于主从复制的分布式锁是无效的

redis的主从复制是异步的，会产生竞态条件：

- 客户端A获取 master 中的锁。
- 在将锁（一个redis的key）写入传输到 slave 之前，master 崩溃。
- slave 晋升为 master。
- 客户端B希望获取对相同资源A的锁定，尽管该资源A已经持有了锁，但这个锁资源并没有被复制到 slave（new master）上，出现了多个线程拿锁的情况，不互斥了！

### 解决方案redlock

#### 算法

假设我们用 5 个 master 节点，分布在不同的机房尽量保证可用性。为了获得锁，客户端会进行如下操作：

- 它以毫秒为单位获取当前时间。
- 尝试顺序地在 5 个实例上申请锁，当然需要使用相同的 key 和 random value，这里客户端需要合理锁的 ttl 大小 以及 获得一个节点上的锁的等待时间，避免长时间和一个 fail 了的节点浪费时间，比如：TTL为5s,设置获取锁最多用1s，所以如果一秒内无法获取锁，就放弃获取这个锁，从而尝试获取下个锁
- 当客户端在大于等于 3 个 master 上成功申请到锁的时候，且它会计算申请锁消耗了多少时间，这部分消耗的时间，采用获得锁的当下时间减去第一步获得的时间戳得到，如果这个时间小于锁的ttl，那么锁就真正获取到了。
- 如果锁申请到了，则锁的真正有效时间是 TTL减去第三步的时间差 的时间
- 如果客户端由于某些原因获取锁失败，便会开始解锁所有redis实例；因为可能已经获取了小于3个锁，必须释放，否则影响其他client获取锁

#### RedLock算法是否是异步算法？

可以看成是同步算法；因为 即使进程间（多个电脑间）没有同步时钟，但是每个进程时间流速大致相同；并且时钟漂移相对于TTL叫小，可以忽略，所以可以看成同步算法；（不够严谨，算法上要算上时钟漂移，因为如果两个电脑在地球两端，则时钟漂移非常大）

#### RedLock失败重试

当客户端不能获取锁时，应该在随机时间后重试获取锁；并且最好在同一时刻并发的把set命令发送给所有redis实例；而且对于已经获取锁的客户端在完成任务后要及时释放锁，这是为了节省时间；

#### RedLock释放锁

由于释放锁时会判断这个锁的value是不是自己设置的，如果是才删除；所以在释放锁时非常简单，只要向所有实例都发出释放锁的命令，不用考虑能否成功释放锁；

#### RedLock注意点（Safety arguments）

1. 先假设客户端获取所有实例，所有实例包含相同的key和过期时间(TTL) ,但每个实例set命令时间不同导致不能同时过期，第一个set命令之前是T1,最后一个set命令后为T2, 则此client有效获取锁的最小时间为TTL-(T2-T1)-时钟漂移;

2. 对于以N/2+ 1(也就是一半以 上)的方式判断获取锁成功，是因为如果小于一半判断为成功的话，有可能出现多个client都成功获取锁的情况， 从而使锁失效

3. 一个客户端锁定一个实例耗费的时间大于或接近锁的过期时间，就认为锁无效，并且解锁这个redis实例(不执行业务) ; 只要在TTL时间内成功获取一半以上的锁便是有效锁;否则无效

 
#### 系统有活性的三个特征

1. 能够自动释放锁

2. 在获取锁失败（不到一半以上），或任务完成后 能够自动释放锁，不用等到其自动过期

3. 在客户端重试获取锁前（第一次失败到第二次重试时间间隔）大于第一次获取锁消耗的时间；

4. 重试获取锁要有一定次数限制

 
#### RedLock性能及崩溃恢复的相关解决方法

1. 如果redis没有持久化功能，在客户端A获取锁成功后，所有redis重启，客户端B能够再次获取到锁，这样违法了锁的排他互斥性;

2. 如果启动AOF永久化存储，事情会好些， 举例:当我们重启redis后，由于redis过期机制是按照unix时间戳走的，所以在重启后，然后会按照规定的时间过期，不影响业务;但是由于AOF同步到磁盘的方式默认是每秒-次，如果在一秒内断电，会导致数据丢失，立即重启会造成锁互斥性失效; 但如果同步磁盘方式使用Always(每一个写命令都同步到硬盘)造成性能急剧下降; 所以在锁完全有效性和性能方面要有所取舍; 

3. 既保证锁完全有效性，又性能高效，即使断电情况也有效 的方法是redis同步到磁盘方式保持 默认的每秒，在redis无论因为什么原因停掉后要等待TTL时间后再重启(延迟重启); 缺点是 在TTL时间内服务相当于暂停状态;

 
### 总结：

1. TTL时长 要大于正常业务执行的时间+获取所有redis服务消耗时间+时钟漂移

2. 获取redis所有服务消耗时间要 远小于TTL时间，并且获取成功的锁个数要 在总数的一般以上:N/2+1

3. 尝试获取每个redis实例锁时的时间要 远小于TTL时间

4. 尝试获取所有锁失败后 重新尝试一定要有一定次数限制

5. 在redis崩溃后（无论一个还是所有），要延迟TTL时间重启redis

6. 在实现多redis节点时要结合单节点分布式锁算法 共同实现