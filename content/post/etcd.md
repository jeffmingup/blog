---
title: "分布式锁理解及etcd实现方案"
date: 2021-09-17T15:56:49+08:00
draft: false
---
## 分布式锁基础

分布式锁，是控制分布式系统之间同步访问共享资源的一种方式。在分布式系统中，常常需要协调他们的动作。如果不同的系统或是同一个系统的不同主机之间共享了一个或一组资源，那么访问这些资源的时候，往往需要互斥来防止彼此干扰来保证一致性，在这种情况下，便需要使用到分布式锁。

### 分布式锁特点

 - 互斥性：任意时刻，同一个锁，只有一个进程能持有

 - 安全性：避免死锁，当进程没有主动释放锁（进程崩溃退出），保证其他进程能够加锁

 - 可用性：当提供锁的服务节点故障（宕机）时，热备节点能够接替故障的节点继续提供服务，并保证自身持有的数据与故障节点一致

- 对称性：对同一个锁，加锁和解锁必须是同一个进程，即某进程不能把其他进程持有的锁给释放了

 <!--互斥性-->

<!--这个没啥好说的，要满足这个特性，一般是多进程共同使用同一个分布式锁服务来管理。-->

 <!--安全性-->

<!--要满足进程崩溃之后，不会出现死锁，就需要给锁加上过期时间。进程崩溃了，锁过期依然会释放。如果出现进程还没处理完，锁就过期了也不行，所以需要守护进程来定时续约过期时间。-->

 <!--可用性-->

<!--这个需要锁服务本身是高可用，像redis主从，etcd（本身就是高可用）-->

 <!--对称性-->

<!--redis处理这个问题的方案是给key分配一个唯一值value，解锁的时候通过lua脚本来判断是不是自己的锁（自己设置的key-value）。-->

<!--etcd是根据key前缀的方式，具体方案看下方，也不会出现这个问题-->

## Redis分布式锁

### SET命令

- EX seconds ： 将键的过期时间设置为 seconds 秒。 执行 SET key value EX seconds 的效果等同于执行 SETEX key seconds value 。

- PX milliseconds ： 将键的过期时间设置为 milliseconds 毫秒。 执行 SET key value PX milliseconds 的效果等同于执行 PSETEX key milliseconds value 。

- NX ： 只在键不存在时， 才对键进行设置操作。 执行 SET key value NX 的效果等同于执行 SETNX key value 。

- XX ： 只在键已经存在时， 才对键进行设置操作。

### 实现流程

redis实现加锁的大致流程就是，使用NX参数多个进程设置同一个key，谁设置成功谁获得锁。

安全性靠过期时间来解决，为了加锁操作的原子性，需要使用set 命令同时加上NX和 EX（过期时间）参数。如果出现进程还没处理完，锁就过期了也不行，所以需要守护进程来定时续约过期时间。

对称性的话，每个进程为key设置一个value(唯一值)。解锁的时候通过lua脚本，先get出value和当前进程做对比，确认一致后才解锁。

### redis分布式锁的问题

其实就是可用性和一致性问题，redis基于内存通常用作缓存服务器，读写性能很高。但是redis的高可用不能保证完全一致性。

比如当前进程刚设置一个锁（set命令），这条set命令还没有同步到redis从服务器上，然后redis主服务器挂了。在redis从服务器升级为主服务器的之后没有这条set命令。就会有别的进程重复获得锁，这样就有问题了。

网上似乎也有一些方案，但是redis本身的一致性并不是它的强项，读写性能高才是。所以类似etcd这种本身就是分布式并具备一致性的服务就成了分布式锁的最佳方案。

## Etcd 特性

这些年，随着 [Raft 协议](https://raft.github.io/) 的广泛应用，涌现出很多的基于此协议实现的高可用分布式系统，比较知名的有 [Etcd](https://github.com/etcd-io/etcd)、[Consul](https://github.com/hashicorp/consul) 等，由于其内部实现了 [分布式一致性](https://zh.wikipedia.org/wiki/CAP定理)，所以非常适合做分布式锁。本片文章就以 Etcd 为例，来介绍下构建分布式锁的方法。值得注意的是，Etcd 分布式锁的实现是在客户端做的。

Etcd 支持以下功能，正是依赖这些功能来实现分布式锁的：

- Lease 机制：即租约机制（TTL，Time To Live），Etcd 可以为存储的 Key-Value 对设置租约，当租约到期，Key-Value 将失效删除；同时也支持续约续期（KeepAlive）
- Revision 机制:  Etcd有个全局的Revision值，Etcd 每进行一次事务对应的全局 Revision 值都会加一 。每个 key 带有一个 CreateRevision属性值就是创建key的时候的全局Revision值，还有对应的ModRevision(修改时候的全局Revision值 )。因此每个 Key 对应的 CreateRevision（ModRevision）属性值都是全局唯一的。(还有一个Version对应当前key的共修改了几次，put相同的值也算修改)。通过比较 CreateRevision的大小就可以知道进行写操作的顺序。在实现分布式锁时，多个程序同时抢锁，根据 CreateRevision值大小依次获得锁，可以避免惊群效应，实现公平锁。
- Prefix 机制：即前缀机制（或目录机制）。可以根据前缀（目录）获取该目录下所有的 Key 及对应的属性（包括 Key、Value 以及 Revision 等）
- Watch 机制：即监听机制，Watch 机制支持 Watch 某个固定的 Key，也支持 Watch 一个目录前缀（前缀机制），当被 Watch 的 Key 或目录发生变化，客户端将收到通知

## 官方Concurrency 包分析

https://github.com/etcd-io/etcd/tree/main/client/v3/concurrency

EtcdV3 版本的 Concurrency 包 提供了实现 分布式锁 及 分布式选主 的接口，这两个概念间是有共性的，譬如，谁抢到了锁谁就是 Master。这次主要说分布式锁，下面是锁的几个方法

```go
type Mutex
	func NewMutex(s *Session, pfx string) *Mutex
	func (m *Mutex) Header() *pb.ResponseHeader
	func (m *Mutex) IsOwner() v3.Cmp
	func (m *Mutex) Key() string
	func (m *Mutex) Lock(ctx context.Context) error
	func (m *Mutex) TryLock(ctx context.Context) error
	func (m *Mutex) Unlock(ctx context.Context) error
```

先看一下Mutex结构体

```go
// Mutex implements the sync Locker interface with etcd
type Mutex struct {
	s *Session //每个进程使用分布式锁的时候，都会和etcd建立一个连接，session就是这次连接的指针

	pfx   string // 锁的共同前缀 pfx，如 "/service/lock/"
	myKey string // 当前持有锁的客户端的 key 值（完整 Key 的组成为 pfx+"/"+leaseid）
	myRev int64  // 当前持有锁的 CreateRevision 
	hdr   *pb.ResponseHeader // 当前put/get命令响应值
}
```

初始化 Mutex，可以看到在 NewMutex 方法中，并非直接拿传进来的 pfx 作为 Prefix 的，而且在后面加了个 /，在 Etcd 开发项目中，定义或查找 Prefix 或 Suffix 时都建议加上分隔符 /，这是个好习惯，也可以避免出现问题。

```go
func NewMutex(s *Session, pfx string) *Mutex {
   //Etcd 这里默认将 / path 的 key 结构转换为一个目录结构 /path/
   return &Mutex{s, pfx + "/", "", -1, nil}
}
```

然后看主要的Lock(),TryLock()方法，

这两个方法的区别：Lock()是阻塞等待的，会一直阻塞到获取锁；TryLock()则是立即返回，锁被占用的时候返回ErrLocked

这两个方法都是先调用了tryAcquire()方法

```go
func (m *Mutex) tryAcquire(ctx context.Context) (*v3.TxnResponse, error) {
	s := m.s
	client := m.s.Client()
	// 下面的 m.pfx 就是 prefix，是传进来的前缀，后面的 s.Lease() 会返回一个租约，是一个 int64 的整数，和 session 有关系
	//mykex 先理解为是 / prefix/leaseid 这样的结构
	m.myKey = fmt.Sprintf("%s%x", m.pfx, s.Lease())
  	/* 这里定义一个 cmp 方法，
  	比较上面 m.myKey 的 CreateRevision 是否为 0 等于 0 表示目前不存在该 key，
  	需要执行 Put 操作 不等于 0 表示对应的 key 已经创建了，需要执行 Get 操作 。
  	
  	简单理解就是，看这个key是否已经存在；如果不存在就put新建，已存在就get获取。
  	已经存在的情况是，一个进程中同一个session重复调用了lock()方法也没关系，这是幂等的。
  	*/
	cmp := v3.Compare(v3.CreateRevision(m.myKey), "=", 0)
  
  
	// put self in lock waiters via myKey; oldest waiter holds lock
	put := v3.OpPut(m.myKey, "", v3.WithLease(s.Lease()))
	// reuse key in case this session already holds the lock
	get := v3.OpGet(m.myKey)
	// fetch current holder to complete uncontended path with only one RPC
  // 获取当前锁的真正持有者
	getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)
  //Txn 事务，判断 cmp 的条件是否成立，成立执行 Then(put新建)，不成立执行 Else(获取get)，最终执行 Commit()
	resp, err := client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()
	if err != nil {
		return nil, err
	}
  //If(cmp) 为ture,执行 Then(put,getOwner)，myRev是当前事务返回的Header.Revisions属性
	m.myRev = resp.Header.Revision
  //If(cmp) 为false, 执行 Else(get,getOwner)，myRev则是get操作返回的CreateRevision属性
	if !resp.Succeeded {
    //resp.Responses[0] 表示事务Then()/Else()方法中的第一个操作的响应值
    //resp.Responses[1] 表示事务Then()/Else()方法中的第二个操作的响应值，以此类推
		m.myRev = resp.Responses[0].GetResponseRange().Kvs[0].CreateRevision
	}
	return resp, nil
}
```

在 tryAcquire 的 TXN 事务中，不管是执行 Put 还是 Get，最终都有个 getOwner 的过程，看一下这个 getOwner 的逻辑，这里有个有趣的选项，getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...) 中的 options 模式里有个 v3.WithFirstCreate() 函数调用，这是一个和 prefix 相关的函数，看看它的实现：

```go
// WithFirstCreate gets the key with the oldest creation revision in the request range.
func WithFirstCreate() []OpOption { return withTop(SortByCreateRevision, SortAscend) }

// withTop gets the first key over the get's prefix given a sort order
func withTop(target SortTarget, order SortOrder) []OpOption {
   return []OpOption{WithPrefix(), WithSort(target, order), WithLimit(1)}
}

// WithPrefix enables 'Get', 'Delete', or 'Watch' requests to operate
// on the keys with matching prefix. For example, 'Get(foo, WithPrefix())'
// can return 'foo1', 'foo2', and so on.
func WithPrefix() OpOption {
	// 返回所有满足 prefix 匹配的 key-value，和 etcdctl get key --prefix 功能一样
   return func(op *Op) {
      if len(op.key) == 0 {
         op.key, op.end = []byte{0}, []byte{0}
         return
      }
      op.end = getPrefix(op.key)
   }
}
```

看到上面的代码，WithPrefix/WithSort，所以 getOwner 的具体执行效果是会把所有有以 lockkey 开头的 Key-Value 都拿到，且按照 CreateRevision 升序排列，并取第一个值，这个意思就很明白了，就是要拿到当前以 lockkey 为 Prefix 的且 CreatereVision 最小的那个 Key，就是目前已经拿到锁的 Key;

TryLock/Lock 上层调用：

```go
// Lock locks the mutex with a cancelable context. If the context is canceled
// while trying to acquire the lock, the mutex tries to clean its stale lock entry.
func (m *Mutex) Lock(ctx context.Context) error {
	// 先调用 tryAcquire 尝试获取锁
	resp, err := m.tryAcquire(ctx)
	if err != nil {
		return err
	}
	// if no key on prefix / the minimum rev is key, already hold the lock
	ownerKey := resp.Responses[1].GetResponseRange().Kvs
  // 当前以 lockkey 为 Prefix下没有值，则表示当前没有人获得锁（第一次场景），
  //这个操作有疑问，之前不是有put或者get操作吗？而且是一个事务，一定会有值的啊，可能是习惯问题，使用slice的时候，先判断len()长度。但我以为确实是无用功。。。
 // 或者锁的 owner 的 CreateRevision 等于当前的 kv 的 Revision，
 // 表示自己是当前以 lockkey 为 Prefix 中createrVision中最小的 则表示已成功获得锁，就可以退出了
	if len(ownerKey) == 0 || ownerKey[0].CreateRevision == m.myRev {
		// 拿当前进程的 Revision 和 Owner 的 CreateVision 比较，如果相等，就是拿到了锁,直接返回
		m.hdr = resp.Header
		return nil
	}
	client := m.s.Client()
	// wait for deletion revisions prior to myKey
	// 走到这里代表没有获得锁，需要等待之前的锁被释放，即 revision 小于当前 revision 的 kv 被删除
	hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
	// 阻塞等待 Owner 释放锁，下面分析 waitDeletes
	// release lock key if wait failed
	if werr != nil {
		m.Unlock(client.Ctx())
	} else {
		// 成功获取锁
		m.hdr = hdr
	}
	return werr
}
```

再看看 waitDeletes函数的行为, waitDeletes模拟了一种公平的先来后到的排队逻辑，只循环等待比当前 Key 的 CreateRevision 值小的Key 被删除后，锁释放后才返回 ，注意上面的关键字：循环等待。此处正是利用了 Revision 是单调递增的特性来实现。

很多blog里面都解释到，只需要watch比当前CreateRevision小的第一个key值就可以了，可以避免惊群效应，那这里为啥还要for循环来watch呢？

我的理解：就像排队，你前面排了很多人，但是你前面的这个大哥排到一半提前走了（进程崩了），但你还是要继续排队。。。这种情况还是少数，大都是办完事才走的

```go
// 注意 waitDeletes 的调用方式：hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
func waitDeletes(ctx context.Context, client *v3.Client, pfx string, maxCreateRev int64) (*pb.ResponseHeader, error) {
   // 注意 WithLastCreate 和 WithMaxCreateRev 这两个特性
   getOpts := append(v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev))
   for {
      // 循环看前面没有没人排队
      resp, err := client.Get(ctx, pfx, getOpts...)
      if err != nil {
         return nil, err
      }
      if len(resp.Kvs) == 0 {
         // 表示在 maxCreateRev 前面已经没有比本进程优先级更高的客户端了，本客户端可以获得锁
         return resp.Header, nil
      }
      lastKey := string(resp.Kvs[0].Key)
      // 阻塞，waitDelete 等待 lastKey+resp.Header.Revision 被成功删除
      //resp.Header.Revision 是当前最小的
      if err = waitDelete(ctx, client, lastKey, resp.Header.Revision); err != nil {
         return nil, err
      }
   }
}

//
func waitDelete(ctx context.Context, client *v3.Client, key string, rev int64) error {
   cctx, cancel := context.WithCancel(ctx)
   defer cancel()

   var wr v3.WatchResponse
   // wch 是个 channel，key 被删除后会往这个 chan 发数据
   wch := client.Watch(cctx, key, v3.WithRev(rev))
   for wr = range wch {
      for _, ev := range wr.Events {
         if ev.Type == mvccpb.DELETE {
            return nil
         }
      }
   }
   if err := wr.Err(); err != nil {
      return err
   }
   if err := ctx.Err(); err != nil {
      return err
   }
   return fmt.Errorf("lost watcher waiting for delete")
}
```

使用 Etcd 实现分布式锁的架构图如下：

![etcd-d-lock](https://raw.githubusercontent.com/jeffmingup/image/master/img/etcd_lock.png)

## 总结步骤

分析完 `concurrency` 包的核心逻辑，这里总结下基于 Etcd 构造（公平式长期）分布式锁的一般流程如下：

1. 假设分布式锁的 Name 为 `/root/lockname`，用来控制某个共享资源，`concurrency` 会自动将其转换为目录形式：`/root/lockname/`
2. 客户端 A 连接 Etcd，创建一个租约 Leaseid_A，并设置 TTL（以业务逻辑来定 TTL 的时间）, 以 `/root/lockname` 为前缀创建全局唯一的 Key，该 Key 的组织形式为 `/root/lockname/{leaseid_A}`，客户端 A 将此 Key 绑定租约写入 Etcd，同时调用 TXN 事务查询写入的情况和具有相同前缀 `/root/lockname/` 的 `Revision` 的排序情况
3. 客户端 A 判断自己是否获得锁，以前缀 `/root/lockname/` 读取 keyValue 列表（keyValue 中带有 Key 对应的 `Revision`），判断自己 Key 的 `Revision` 是否为当前列表中最小的，如果是则认为获得锁；否则阻塞监听列表中前一个 `Revision` 比自己小的 Key 的删除事件，一旦监听到删除事件或者因租约失效而删除的事件，则自己获得锁
4. 执行业务逻辑，操作共享资源
5. 释放分布式锁，现网的程序逻辑需要实现在正常和异常条件下的释放锁的策略，如捕获 `SIGTERM` 后执行 `Unlock`，或者异常退出时，有完善的监控和及时删除 Etcd 中的 Key 的异步机制。
6. 当客户端持有锁期间，其它客户端只能等待，为了避免等待期间租约失效，客户端需创建一个定时任务进行续约续期。如果持有锁期间客户端崩溃，心跳停止，Key 将因租约到期而被删除，从而锁释放，避免死锁。

参考资料

- [Etcd 应用开发之分布式锁](https://pandaychen.github.io/2019/10/24/ETCD-DISTRIBUTED-LOCK/)

- [Etcdv3.3.10：concurrency](https://github.com/etcd-io/etcd/tree/v3.3.10/clientv3/concurrency)
