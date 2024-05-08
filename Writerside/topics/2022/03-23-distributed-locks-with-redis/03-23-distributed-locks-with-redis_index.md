---
slug: distributed-locks-with-redis
title: Distributed locks with Redis
authors: [solidSpoon]
tags: []
---

# Distributed locks with Redis

> 原文链接：[https://redis.io/topics/distlock](https://redis.io/topics/distlock)

## Background

Distributed locks are a very useful primitive in many environments where different processes must operate with shared resources in a mutually exclusive way.

分布式锁在许多环境中是非常有用的原语，在这些环境中不同的进程必须以互斥的方式使用共享资源。

There are a number of libraries and blog posts describing how to implement a DLM (Distributed Lock Manager) with Redis, but every library uses a different approach, and many use a simple approach with lower guarantees compared to what can be achieved with slightly more complex designs.

有许多库和博客文章描述了如何使用 Redis 实现 DLM（分布式锁管理器），但是每个库都使用不同的方法，并且与稍微复杂一点的库相比，许多库使用的保证较低的简单方法设计。

This page is an attempt to provide a more canonical algorithm to implement distributed locks with Redis. We propose an algorithm, called **Redlock**, which implements a DLM which we believe to be safer than the vanilla single instance approach. We hope that the community will analyze it, provide feedback, and use it as a starting point for the implementations or more complex or alternative designs.

本页试图提供一种更规范的算法来使用 Redis 实现分布式锁。我们提出了一种称为 **Redlock** 的算法，它实现了一个我们认为比普通单实例方法更安全的 DLM。我们希望社区能够对其进行分析，提供反馈，并将其用作实施或更复杂或替代设计的起点。

## Implementations

Before describing the algorithm, here are a few links to implementations already available that can be used for reference.

在描述算法之前，这里有一些已经可用的实现链接，可供参考。

* [Redlock-rb](https://github.com/antirez/redlock-rb) (Ruby implementation). There is also a [fork of Redlock-rb](https://github.com/leandromoreira/redlock-rb) that adds a gem for easy distribution and perhaps more.
* [Redlock-py](https://github.com/SPSCommerce/redlock-py) (Python implementation).
* [Pottery](https://github.com/brainix/pottery#redlock) (Python implementation).
* [Aioredlock](https://github.com/joanvila/aioredlock) (Asyncio Python implementation).
* [Redlock-php](https://github.com/ronnylt/redlock-php) (PHP implementation).
* [PHPRedisMutex](https://github.com/malkusch/lock#phpredismutex) (further PHP implementation)
* [cheprasov/php-redis-lock](https://github.com/cheprasov/php-redis-lock) (PHP library for locks)
* [rtckit/react-redlock](https://github.com/rtckit/reactphp-redlock) (Async PHP implementation)
* [Redsync](https://github.com/go-redsync/redsync) (Go implementation).
* [Redisson](https://github.com/mrniko/redisson) (Java implementation).
* [Redis::DistLock](https://github.com/sbertrang/redis-distlock) (Perl implementation).
* [Redlock-cpp](https://github.com/jacket-code/redlock-cpp) (C++ implementation).
* [Redlock-cs](https://github.com/kidfashion/redlock-cs) (C#/.NET implementation).
* [RedLock.net](https://github.com/samcook/RedLock.net) (C#/.NET implementation). Includes async and lock extension support.
* [ScarletLock](https://github.com/psibernetic/scarletlock) (C# .NET implementation with configurable datastore)
* [Redlock4Net](https://github.com/LiZhenNet/Redlock4Net) (C# .NET implementation)
* [node-redlock](https://github.com/mike-marcacci/node-redlock) (NodeJS implementation). Includes support for lock extension.

## Safety and Liveness guarantees

We are going to model our design with just three properties that, from our point of view, are the minimum guarantees needed to use distributed locks in an effective way.

我们将仅使用三个属性对我们的设计进行建模，从我们的角度来看，这些属性是有效使用分布式锁所需的最低保证。

1. Safety property: Mutual exclusion. At any given moment, only one client can hold a lock.
2. Liveness property A: Deadlock free. Eventually it is always possible to acquire a lock, even if the client that locked a resource crashes or gets partitioned.
3. Liveness property B: Fault tolerance. As long as the majority of Redis nodes are up, clients are able to acquire and release locks.

4. 安全性：互斥。在任何给定时刻，只有一个客户端可以持有锁。
5. Liveness property A: 无死锁。最终，即使锁定资源的客户端崩溃或被分区，也总是可以获得锁。
6. Liveness property B: 容错。只要大多数 Redis 节点都启动，客户端就可以获取和释放锁。

## Why failover-based implementations are not enough

To understand what we want to improve, let’s analyze the current state of affairs with most Redis-based distributed lock libraries.

为了了解我们想要改进的地方，让我们分析一下大多数基于 Redis 的分布式锁库的现状。

The simplest way to use Redis to lock a resource is to create a key in an instance. The key is usually created with a limited **time to live**, using the Redis expires feature, so that eventually it will get released (property 2 in our list). When the client needs to release the resource, it deletes the key.

使用 Redis 锁定资源的最简单方法是在实例中创建 key。key 通常是使用 Redis 过期功能在有限的时间内创建的，因此最终它会被释放（我们列表中的属性 2）。当客户端需要释放资源时，它会删除 key。

Superficially this works well, but there is a problem: this is a single point of failure in our architecture. What happens if the Redis master goes down? Well, let’s add a replica! And use it if the master is unavailable. This is unfortunately not viable. By doing so we can’t implement our safety property of mutual exclusion, because Redis replication is asynchronous.

从表面上看，这很好用，但有一个问题：这是我们架构中的单点故障。如果 Redis master 宕机了怎么办？好吧，让我们添加一个副本！如果 master 不可用，请使用它。不幸的是，这是不可行的。这样做我们无法实现互斥的安全属性，因为 Redis 复制是异步的。

There is an obvious race condition with this model:

这个模型有一个明显的竞争条件：

1. Client A acquires the lock in the master.
2. The master crashes before the write to the key is transmitted to the replica.
3. The replica gets promoted to master.
4. Client B acquires the lock to the same resource A already holds a lock for. **SAFETY VIOLATION!**

5. 客户端 A 获取 master 中的锁。
6. master 在对 key 的写入传输到 replica 之前崩溃。
7. replica 被提升为 master。
8. 客户端 B 获取 A 已经持有锁的同一资源的锁。 **违反安全规定！**

Sometimes it is perfectly fine that under special circumstances, like during a failure, multiple clients can hold the lock at the same time. If this is the case, you can use your replication based solution. Otherwise we suggest to implement the solution described in this document.

有时在特殊情况下（例如在故障期间），多个客户端可以同时持有锁是完全可以的。如果是这种情况，您可以使用基于复制的解决方案。否则，我们建议实施本文档中描述的解决方案。

## Correct implementation with a single instance

Before trying to overcome the limitation of the single instance setup described above, let’s check how to do it correctly in this simple case, since this is actually a viable solution in applications where a race condition from time to time is acceptable, and because locking into a single instance is the foundation we’ll use for the distributed algorithm described here.

在尝试克服上述单实例设置的限制之前，让我们检查一下如何在这个简单的情况下正确地做到这一点，因为在不时出现竞争条件的应用程序中，这实际上是一个可行的解决方案，并且因为锁定到单个实例是我们将用于此处描述的分布式算法的基础。

To acquire the lock, the way to go is the following:

获取锁的方法如下：

```bash
SET resource_name my_random_value NX PX 30000
```

The command will set the key only if it does not already exist (NX option), with an expire of 30000 milliseconds (PX option). The key is set to a value “`my_random_value`”. This value must be unique across all clients and all lock requests.

该命令仅在密钥不存在时设置密钥（NX option），过期时间为 30000 毫秒（PX option）。密钥设置为值`my_random_value`。此值在所有客户端和所有锁定请求中必须是唯一的。

Basically the random value is used in order to release the lock in a safe way, with a script that tells Redis: remove the key only if it exists and the value stored at the key is exactly the one I expect to be. This is accomplished by the following Lua script:

基本上，随机值用于以安全的方式释放锁，脚本告诉 Redis：仅当密钥存在并且存储在密钥中的值正是我期望的值时才删除密钥。这是通过以下 Lua 脚本完成的：

```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

This is important in order to avoid removing a lock that was created by another client. For example a client may acquire the lock, get blocked in some operation for longer than the lock validity time (the time at which the key will expire), and later remove the lock, that was already acquired by some other client. Using just DEL is not safe as a client may remove the lock of another client. With the above script instead every lock is “signed” with a random string, so the lock will be removed only if it is still the one that was set by the client trying to remove it.

这对于避免删除由另一个客户端创建的锁很重要。例如，一个客户端可能获取了锁，在某个操作中被阻塞的时间超过了锁的有效时间（密钥过期的时间），然后移除了已经被其他客户端获取的锁。仅使用 DEL 是不安全的，因为客户端可能会删除另一个客户端的锁定。使用上面的脚本，每个锁都用一个随机字符串“签名”，所以只有当它仍然是客户端试图移除它时设置的锁才会被移除。

What should this random string be? I assume it’s 20 bytes from /dev/urandom, but you can find cheaper ways to make it unique enough for your tasks. For example a safe pick is to seed RC4 with /dev/urandom, and generate a pseudo random stream from that. A simpler solution is to use a combination of unix time with microseconds resolution, concatenating it with a client ID, it is not as safe, but probably up to the task in most environments.

这个随机字符串应该是什么？我假设它是 /dev/urandom 中的 20 个字节，但您可以找到更便宜的方法来使其对您的任务足够独特。例如，一个安全的选择是使用 /dev/urandom 作为 RC4 的种子，并从中生成一个伪随机流。一个更简单的解决方案是使用 unix 时间与微秒分辨率的组合，将其与客户端 ID 连接起来，它不是那么安全，但可能在大多数环境中都可以胜任。

The time we use as the key **time to live**, is called the “lock validity time”. It is both the auto release time, and the time the client has in order to perform the operation required before another client may be able to acquire the lock again, without technically violating the mutual exclusion guarantee, which is only limited to a given window of time from the moment the lock is acquired.

key 的过期时间，称为「锁有效时间」。它既是自动释放时间，也是客户端在另一个客户端可能能够再次获取锁之前执行所需操作的时间，而不会在技术上违反互斥保证，互斥保证仅限于从获得锁的那一刻起的给定时间窗口

So now we have a good way to acquire and release the lock. The system, reasoning about a non-distributed system composed of a single, always available, instance, is safe. Let’s extend the concept to a distributed system where we don’t have such guarantees.

所以现在我们有了一个获取和释放锁的好方法。该系统推理由一个始终可用的单个实例组成的非分布式系统是安全的。让我们将这个概念扩展到没有此类保证的分布式系统。

## The Redlock algorithm

In the distributed version of the algorithm we assume we have N Redis masters. Those nodes are totally independent, so we don’t use replication or any other implicit coordination system. We already described how to acquire and release the lock safely in a single instance. We take for granted that the algorithm will use this method to acquire and release the lock in a single instance. In our examples we set N=5, which is a reasonable value, so we need to run 5 Redis masters on different computers or virtual machines in order to ensure that they’ll fail in a mostly independent way.

在算法的分布式版本中，我们假设我们有 N 个 Redis master。这些节点是完全独立的，所以我们不使用复制或任何其他隐式协调系统。我们已经描述了如何在单个实例中安全地获取和释放锁。我们理所当然地认为算法会使用这种方法在单个实例中获取和释放锁。在我们的示例中，我们设置 N=5，这是一个合理的值，因此我们需要在不同的计算机或虚拟机上运行 5 个 Redis 主服务器，以确保它们以几乎独立的方式发生故障。

In order to acquire the lock, the client performs the following operations:

为了获取锁，客户端执行以下操作：

1. It gets the current time in milliseconds.
2. It tries to acquire the lock in all the N instances sequentially, using the same key name and random value in all the instances. During step 2, when setting the lock in each instance, the client uses a timeout which is small compared to the total lock auto-release time in order to acquire it. For example if the auto-release time is 10 seconds, the timeout could be in the ~ 5-50 milliseconds range. This prevents the client from remaining blocked for a long time trying to talk with a Redis node which is down: if an instance is not available, we should try to talk with the next instance ASAP.
3. The client computes how much time elapsed in order to acquire the lock, by subtracting from the current time the timestamp obtained in step 1. If and only if the client was able to acquire the lock in the majority of the instances (at least 3), and the total time elapsed to acquire the lock is less than lock validity time, the lock is considered to be acquired.
4. If the lock was acquired, its validity time is considered to be the initial validity time minus the time elapsed, as computed in step 3.
5. If the client failed to acquire the lock for some reason (either it was not able to lock N/2+1 instances or the validity time is negative), it will try to unlock all the instances (even the instances it believed it was not able to lock).

6. 它以毫秒为单位获取当前时间。
7. 它尝试顺序获取所有 N 个实例中的锁，在所有实例中使用相同的键名和随机值。在步骤 2 中，当在每个实例中设置锁时，客户端使用一个与锁自动释放总时间相比较小的 timeout 来获取它（防止单点阻塞）。例如，如果锁的自动释放时间为 10 秒，则 timeout 可能在 ~ 5-50 毫秒范围内。这可以防止客户端在尝试与已关闭的 Redis 节点通信时长时间保持阻塞：如果一个实例不可用，我们应该尽快尝试与下一个实例通信。
8. 客户端通过从当前时间中减去步骤 1 中获得的时间戳来计算获取锁所用的时间。当且仅当客户端能够在大多数实例中获取锁（至少 3 个） ，且获取锁的总时间小于锁的有效时间，则认为锁已被获取。
9. 如果获得了锁，则其有效时间被认为是初始有效时间减去经过的时间，如步骤 3 中计算的那样。
10. 如果客户端由于某种原因未能获得锁（它无法锁定 N/2+1 个实例或有效时间为负数），它将尝试解锁所有实例（甚至是它认为无法锁定的实例）。

## Is the algorithm asynchronous?

The algorithm relies on the assumption that while there is no synchronized clock across the processes, still the local time in every process flows approximately at the same rate, with an error which is small compared to the auto-release time of the lock. This assumption closely resembles a real-world computer: every computer has a local clock and we can usually rely on different computers to have a clock drift which is small.

该算法依赖于这样一个假设：虽然进程之间没有同步时钟，但每个进程中的本地时间仍然以大致相同的速率流动，与锁的自动释放时间相比，误差很小。这个假设非常类似于现实世界的计算机：每台计算机都有一个本地时钟，我们通常可以依靠不同的计算机来获得很小的时钟漂移。

At this point we need to better specify our mutual exclusion rule: it is guaranteed only as long as the client holding the lock will terminate its work within the lock validity time (as obtained in step 3), minus some time (just a few milliseconds in order to compensate for clock drift between processes).

此时我们需要更好地指定我们的互斥规则：只有持有锁的客户端会在锁的有效期内（如步骤 3 中获得）内终止其工作，减去一些时间（仅几毫秒，才能保证互斥规则为了补偿进程之间的时钟漂移）。

For more information about similar systems requiring a bound _clock drift_, this paper is an interesting reference: [Leases: an efficient fault-tolerant mechanism for distributed file cache consistency](http://dl.acm.org/citation.cfm?id=74870).

## Retry on failure

When a client is unable to acquire the lock, it should try again after a random delay in order to try to desynchronize multiple clients trying to acquire the lock for the same resource at the same time (this may result in a split brain condition where nobody wins). Also the faster a client tries to acquire the lock in the majority of Redis instances, the smaller the window for a split brain condition (and the need for a retry), so ideally the client should try to send the SET commands to the N instances at the same time using multiplexing.

当客户端无法获取锁时，它应该在随机延迟后再次尝试，这是为了尽可能同步多个客户端同时尝试获取同一资源的锁（这可能会导致没有人获胜的脑裂状态）。此外，客户端在大多数 Redis 实例中尝试获取锁的速度越快，裂脑条件的窗口就越小（以及重试的需要），因此理想情况下，客户端应该尝试将 SET 命令发送到 N 个实例同时使用多路复用。

It is worth stressing how important it is for clients that fail to acquire the majority of locks, to release the (partially) acquired locks ASAP, so that there is no need to wait for key expiry in order for the lock to be acquired again (however if a network partition happens and the client is no longer able to communicate with the Redis instances, there is an availability penalty to pay as it waits for key expiration).

值得强调的是，对于未能获得大部分锁的客户端来说，尽快释放（部分）获得的锁是多么重要，这样就无需等待密钥到期才能再次获得锁（但是，如果发生网络分区并且客户端不再能够与 Redis 实例通信，则需要承担「等待密钥到期」的性能损失）。

## Releasing the lock

Releasing the lock is simple and involves just releasing the lock in all instances, whether or not the client believes it was able to successfully lock a given instance.

释放锁很简单，只涉及在所有实例中释放锁，无论客户端是否相信它能够成功锁定给定实例。

## Safety arguments

Is the algorithm safe? We can try to understand what happens in different scenarios.

算法安全吗？我们可以尝试了解在不同情况下会发生什么。

To start let’s assume that a client is able to acquire the lock in the majority of instances. All the instances will contain a key with the same **time to live**. However, the key was set at different times, so the keys will also expire at different times. But if the first key was set at worst at time T1 (the time we sample before contacting the first server) and the last key was set at worst at time T2 (the time we obtained the reply from the last server), we are sure that the first key to expire in the set will exist for at least `MIN_VALIDITY=TTL-(T2-T1)-CLOCK_DRIFT`(TTL Time To Live). All the other keys will expire later, so we are sure that the keys will be simultaneously set for at least this time.

首先让我们假设客户端能够在大多数情况下获取锁。所有实例都将包含一个具有相同生存时间的 key。但是，key 是在不同的时间设置的，所以 key 也会在不同的时间过期。但是如果第一个密钥在时间 T1（我们在联系第一台服务器之前采样的时间）设置为最差，而最后一个密钥在时间 T2（我们从最后一个服务器获得回复的时间）设置为最差，我们确定集合中第一个过期的密钥将至少存在 `MIN_VALIDITY = TTL-(T2-T1)-CLOCK_DRIFT`。所有其他密钥都将在稍后过期，因此我们确信这些 key 将至少在段时间内同时设置。

During the time that the majority of keys are set, another client will not be able to acquire the lock, since N/2+1 SET NX operations can’t succeed if N/2+1 keys already exist. So if a lock was acquired, it is not possible to re-acquire it at the same time (violating the mutual exclusion property).

在设置大部分键的时间内，另一个客户端将无法获取锁，因为如果 N/2+1 个键已经存在，则 N/2+1 个 `SET NX` 操作将无法成功。所以如果获得了一个锁，就不可能同时重新获得它（违反互斥属性）。

However we want to also make sure that multiple clients trying to acquire the lock at the same time can’t simultaneously succeed.

但是，我们还想确保多个客户端同时尝试获取锁不能同时成功。

If a client locked the majority of instances using a time near, or greater, than the lock maximum validity time (the TTL we use for SET basically), it will consider the lock invalid and will unlock the instances, so we only need to consider the case where a client was able to lock the majority of instances in a time which is less than the validity time. In this case for the argument already expressed above, for `MIN_VALIDITY` no client should be able to re-acquire the lock. So multiple clients will be able to lock N/2+1 instances at the same time (with "time" being the end of Step 2) only when the time to lock the majority was greater than the TTL time, making the lock invalid.

如果客户端使用接近或大于锁最大有效时间（基本上就是我们给 SET 操作设置的的 TTL）的时间锁定大多数实例，它将认为锁无效并解锁实例，所以我们只需要考虑客户端能够在小于有效时间的时间内锁定大多数实例的情况。在这种情况下，对于上面已经表达的参数，对于“MIN_VALIDITY”，没有客户端应该能够重新获取锁。因此，只有当锁定多数的时间大于 TTL 时间时，多个客户端才能同时锁定 N/2+1 个实例（“时间”是步骤 2 的结束），如前文所述，锁定会被判定为无效。

Are you able to provide a formal proof of safety, point to existing algorithms that are similar, or find a bug? That would be greatly appreciated.

您是否能够提供正式的安全证明、指出现有的相似算法或发现错误？那将不胜感激。

## Liveness arguments

The system liveness is based on three main features:

系统活跃度基于三个主要特征：

1. The auto release of the lock (since keys expire): eventually keys are available again to be locked.
2. The fact that clients, usually, will cooperate removing the locks when the lock was not acquired, or when the lock was acquired and the work terminated, making it likely that we don’t have to wait for keys to expire to re-acquire the lock.
3. The fact that when a client needs to retry a lock, it waits a time which is comparably greater than the time needed to acquire the majority of locks, in order to probabilistically make split brain conditions during resource contention unlikely.

4. 锁的自动释放（因为 key 过期）：最终 key 可以再次被锁定。
5. 事实上，客户端通常会在未获取锁或获取锁但工作终止时合作移除锁，这使得我们可能不必等待密钥过期来重新获取锁。
6. 事实上，当客户端需要重试锁时，它等待的时间比获取大多数锁所需的时间要长得多，以便在资源争用期间不太可能出现脑裂情况。

However, we pay an availability penalty equal to TTL time on network partitions, so if there are continuous partitions, we can pay this penalty indefinitely. This happens every time a client acquires a lock and gets partitioned away before being able to remove the lock.

Basically if there are infinite continuous network partitions, the system may become not available for an infinite amount of time.

基本上，如果有无限连续的网络分区，系统可能会在无限长的时间内变得不可用。

## Performance, crash-recovery and fsync

Many users using Redis as a lock server need high performance in terms of both latency to acquire and release a lock, and number of acquire / release operations that it is possible to perform per second. In order to meet this requirement, the strategy to talk with the N Redis servers to reduce latency is definitely multiplexing (or poor man's multiplexing, which is, putting the socket in non-blocking mode, send all the commands, and read all the commands later, assuming that the RTT(Round-trip delay 往返延误) between the client and each instance is similar).

许多使用 Redis 作为锁服务器的用户在获取和释放锁的延迟以及每秒可以执行的获取/释放操作的数量方面都需要高性能。为了满足这个需求，与 N 台 Redis 服务器对话以减少延迟的策略肯定是多路复用（或者说穷人的多路复用，也就是将 socket 置于非阻塞模式，发送所有命令，稍后读取所有命令，假设客户端和每个实例之间的 RTT 是相似的）。

However there is another consideration to do about persistence if we want to target a crash-recovery system model.

然而，如果我们想要针对崩溃恢复系统模型，还有另一个关于持久性的考虑。

Basically to see the problem here, let’s assume we configure Redis without persistence at all. A client acquires the lock in 3 of 5 instances. One of the instances where the client was able to acquire the lock is restarted, at this point there are again 3 instances that we can lock for the same resource, and another client can lock it again, violating the safety property of exclusivity of lock.

基本上看这里的问题，让我们假设我们配置 Redis 时完全没有持久化。客户端在 5 个实例中的 3 个中获得了锁。其中一个客户端能够获得锁的实例被重启，此时我们又可以为同一个资源锁定 3 个实例，另一个客户端可以再次锁定它，违反了锁的排他性的安全属性。

If we enable AOF persistence, things will improve quite a bit. For example we can upgrade a server by sending SHUTDOWN and restarting it. Because Redis expires are semantically implemented so that virtually the time still elapses when the server is off, all our requirements are fine. However everything is fine as long as it is a clean shutdown. What about a power outage? If Redis is configured, as by default, to fsync on disk every second, it is possible that after a restart our key is missing. In theory, if we want to guarantee the lock safety in the face of any kind of instance restart, we need to enable fsync=always in the persistence setting. This in turn will totally ruin performances to the same level of CP systems that are traditionally used to implement distributed locks in a safe way.

如果我们启用 AOF 持久性，事情会改善很多。例如，我们可以通过发送 SHUTDOWN 并重新启动它来升级服务器。因为 Redis 过期是在语义上实现的，所以当服务器关闭时，实际上时间仍然过去，我们所有的要求都很好。但是，只要它是干净的关闭，一切都很好。停电怎么办？如果 Redis 默认配置为每秒在磁盘上 fsync 一次，那么重启后我们的 key 可能会丢失。理论上，如果我们想在任何类型的实例重启时保证锁的安全性，我们需要在持久化设置中启用 fsync=always。这反过来又会完全破坏与传统上用于以安全方式实现分布式锁的 CP 系统相同级别的性能。

> [Consistency](https://en.wikipedia.org/wiki/Consistency_model)
> Every read receives the most recent write or an error.
>
> [Availability](https://en.wikipedia.org/wiki/Availability)
> Every request receives a (non-error) response, without the guarantee that it contains the most recent write.
>
> [Partition tolerance](https://en.wikipedia.org/wiki/Network_partitioning)
> The system continues to operate despite(尽管) an arbitrary number of messages being dropped (or delayed) by the network between nodes.

However things are better than what they look like at a first glance. Basically the algorithm safety is retained as long as when an instance restarts after a crash, it no longer participates to any **currently active** lock, so that the set of currently active locks when the instance restarts, were all obtained by locking instances other than the one which is rejoining the system.

然而，事情比乍看之下要好。基本上算法安全性只要在实例崩溃后重启时，不再参与任何**当前活动的**锁，因此实例重启时当前活动的锁集合，都是通过锁定实例获得的除了重新加入系统的那个。

To guarantee this we just need to make an instance, after a crash, unavailable for at least a bit more than the max TTL we use, which is, the time needed for all the keys about the locks that existed when the instance crashed, to become invalid and be automatically released.

为了保证这一点，我们只需要创建一个实例，在崩溃之后，至少比我们使用的最大 TTL 多一点之后再变得可用，即所有关于锁的键所需的时间实例崩溃时存在的，变为无效并自动释放。

Using _delayed restarts_ it is basically possible to achieve safety even without any kind of Redis persistence available, however note that this may translate into an availability penalty. For example if a majority of instances crash, the system will become globally unavailable for TTL (here globally means that no resource at all will be lockable during this time).

使用 _delayed restarts_ 基本上可以实现安全，即使没有任何可用的 Redis 持久性，但是请注意，这可能会转化为可用性损失。例如，如果大多数实例崩溃，系统将在 TTL 期间全局不可用（这里全局意味着在此期间根本没有资源可锁定）。

## Making the algorithm more reliable: Extending the lock

If the work performed by clients is composed of small steps, it is possible to use smaller lock validity times by default, and extend the algorithm implementing a lock extension mechanism. Basically the client, if in the middle of the computation while the lock validity is approaching a low value, may extend the lock by sending a Lua script to all the instances that extends the TTL of the key if the key exists and its value is still the random value the client assigned when the lock was acquired.

如果客户端执行的工作由小步骤组成，则可以默认使用较小的锁有效时间，并扩展实现锁扩展机制的算法。基本上客户端，如果在计算过程中锁有效性接近一个低值（快过期了），可以通过向所有实例发送一个 Lua 脚本来扩展锁，条件是密钥存在并且它的值仍然是获取锁时客户端分配的随机值。

The client should only consider the lock re-acquired if it was able to extend the lock into the majority of instances, and within the validity time (basically the algorithm to use is very similar to the one used when acquiring the lock).

如果客户端能够将锁扩展到大多数实例，并且在有效时间内（基本上使用的算法与获取锁时使用的算法非常相似），客户端应该只考虑重新获取锁。

However this does not technically change the algorithm, so the maximum number of lock reacquisition attempts should be limited, otherwise one of the liveness properties is violated.

然而，这在技术上并没有改变算法，因此应该限制重新获取锁的最大尝试次数，否则会违反 liveness properties 之一。

## Want to help?

--------------------------------

If you are into distributed systems, it would be great to have your opinion / analysis. Also reference implementations in other languages could be great.

如果您进入分布式系统，那么有您的意见/分析会很棒。其他语言的参考实现也可能很棒。

Thanks in advance!