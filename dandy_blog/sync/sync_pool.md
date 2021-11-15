

# sync.Pool对象池源码解析

### 何为对象池

在开发过程中，代码里头我们经常会创建和销毁同一类对象。而频繁的创建和销毁开销还是很大的，常见的优化手段就是创建`对象池`。**对象池**就是提前创建很多对象，使用过的对象不销毁保存起来，等下次请求在重复使用该对象。我们今天分析的主角就是：Go 标准库`sync.Pool`(go1.16.2)。但是`sync.Pool`中的对象会被GC所清理掉。

### sync.pool使用

```go
func main() {
  count:=0
  pool := sync.Pool{New:func() interface{} {
		count++
		return count
  }}
	v1 := pool.Get()
	fmt.Printf("value 1: %v\n", v1)
	pool.Put(10)
	pool.Put(11)
	pool.Put(12)
	v2 := pool.Get()
	fmt.Printf("value 2: %v\n", v2)
  // 第一次gc
	runtime.GC()
	v3 := pool.Get()
	fmt.Printf("value 3: %v\n", v3)
	runtime.GC()
  // 第二次gc
	v4 := pool.Get()
	fmt.Printf("value 4: %v\n", v4)
	pool.New = nil
	v5 := pool.Get()
	fmt.Printf("value 5: %v\n", v5)
}
```

指定New回调，当Get没有获取到的时候，就会调用该方法。这里调用GC，目的是验证sync.pool会被GC给清理掉，并且我们会看到第二次GC后，才会正真意义的释放

### Pool源码分析

```go
type Pool struct {
   // go vet检测使用
   noCopy noCopy
	 // 指向数组实际的类型[P]poolLoca
   local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
   // 数组大小
   localSize uintptr        // size of the local array
   // 受害者缓存 类似分代垃圾回收的思想
   victim     unsafe.Pointer // local from previous cycle
   victimSize uintptr        // size of victims array
   // get获取不到的创建新的对象使用
   New func() interface{}
}

type poolLocal struct {
    poolLocalInternal
    // 占位使用，防止cpu cache line上发生false sharing
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
type poolLocalInternal struct {
	private interface{} // Can be used only by the respective P.
	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}
```

### Put保存对象

```go
func (p *Pool) Put(x interface{}) {
   // 数据为nil不保存
   if x == nil {
      return
   }
   // data race
   // 绑定g和p，禁止当前P被抢占
   l, _ := p.pin()
   // 将x赋值给private
   if l.private == nil {
      l.private = x
      x = nil
   }
   // private已经赋值了，那只能放本地的队列中了
   if x != nil {
      l.shared.pushHead(x)
   }
   // 解除抢占
   runtime_procUnpin()
   // data race
}
```

此处省略data race校验相关的代码。

- 调用pin()绑定p上锁，防止当前的p被其他的g抢占。
- 保存数据，先存private，再存shared队列中的。`pushHead`我们后面分解
- runtime_procUnpin可以理解为解锁，解除抢占

#### Get获取对象

```go
func (p *Pool) Get() interface{} {
   // .... data race
   // 绑定g和p，禁止当前P被抢占 
   l, pid := p.pin()
   x := l.private
   l.private = nil
   if x == nil {
      // Try to pop the head of the local shard. We prefer
      // the head over the tail for temporal locality of
      // reuse.
      // 本地的用pophead不用popTail,popTail会浪费空间
      x, _ = l.shared.popHead()
      if x == nil {
        // 从其他的P获取队列,只偷一次，不确定会不会出现P没偷完的情况。(还需恶补GMP模型)
         x = p.getSlow(pid)
      }
   }
   // 解除抢占
   runtime_procUnpin()
	 // .... data race
   if x == nil && p.New != nil {
      x = p.New()
   }
   return x
}
```

此处省略data race校验相关的代码。

1. 调用pin()绑定p上锁，防止当前的p被其他的g抢占
2. 取出private赋值给x，如果x为空，继续取队列shared.popHead()的值。如果获取不到在从其他的p获取一个poolLocal。
3. getSlow中先从其他P获取，如果其他p获取不到在从cache获取数据，最后都获取不到，就调用New生成

**详细看pin()怎么获取poolLocal**

```go
func (p *Pool) pin() (*poolLocal, int) {
   // GMP会获取到对应的p。我们可以理解跟进程pid类似
   pid := runtime_procPin()
   // 获取size大小
   s := runtime_LoadAcquintptr(&p.localSize) // load-acquire
   l := p.local                              // load-consume
   if uintptr(pid) < s {
      // 根据pid直接获取对应的poolLocal
      return indexLocal(l, pid), pid
   }
   // 第一次localSize=0的时候或者被gc的时候。
   return p.pinSlow()
}
// runtime_procPin最终调用，获取到GMP中P的id
func procPin() int {
	_g_ := getg()
	mp := _g_.m
	mp.locks++
	return int(mp.p.ptr().id)
}
```

procPin()这里mp.locks加锁，禁止其他g抢占p。之后获取原子性的获取p.localsize，这里只有gc的时候，才会修改这个值。为什么加上原子性的操作? 如果本地有直接indexLocal获取，`indexLocal`（取p.local首地址+当前pid*size大小偏移）获取。 如果本地也获取不到，调用p.pinSlow()。从其他的p获取。第一次或对象被gc的时候，也会调用pinSlow初始化。

#### pinSlow获取存储poolLocal

![image-20210715200935699](/Users/11126518/knowledge/dandy_blog/image/image-20210715200935699.png)

```go
func (p *Pool) pinSlow() (*poolLocal, int) {
   // 先释放抢占
   runtime_procUnpin()
   // 上锁，全局大锁，上锁更耗时吧，所以上面先释放了
   allPoolsMu.Lock()
   defer allPoolsMu.Unlock()
   // 这里应该会获取到别的p
   pid := runtime_procPin()
   // poolCleanup won't be called while we are pinned.
   s := p.localSize
   l := p.local
   // 在判断一把，自身p没有，也可能自身的p被其他的G抢占了。设置了pid有数据
   if uintptr(pid) < s {
      return indexLocal(l, pid), pid
   }
   // 开始的时候，gc后，获取别的p的时候，local肯定都是nil
   if p.local == nil {
      allPools = append(allPools, p)
   }
   // 获取跟CPU核数相同的P
   size := runtime.GOMAXPROCS(0)
   local := make([]poolLocal, size)
   // 将切片存到p.local，这里就是为什么我们get的时候，从local获取
   atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
   // 在存localSize大小
   runtime_StoreReluintptr(&p.localSize, uintptr(size))     // store-release
   // 返回对应的poolLocal
   return &local[pid], pid
}
```

1. 全局大锁，阻塞等待上锁的时间，比较持久。借用曹大的分析[几个 Go 系统可能遇到的锁问题](https://xargin.com/lock-contention-in-go/),如果每次请求都new一个sync.Pool，那么每次请求将是灾难性的全局锁加锁。因为新的pool一定会走到pinSlow去创建poolLocal。所以我们需要先`runtime_procUnpin`释放抢占

2. 再次获取P，如果其他的P之前已经设置过了。那么local就会有数据了，所以需要在判断一把。
3. 之后都是初始化操作，给对应的p.local和p.localSize赋值(`代码注释`)，这里为什么需要原子操作，可能gc的时候，会操作。需要在查些资料可能会清晰一些，大家可以一起探讨

#### 数据存储shared.pushHead()

![image-20210715200935699](/Users/11126518/knowledge/dandy_blog/image/push_pool.gif)

```go
func (c *poolChain) pushHead(val interface{}) {
   d := c.head
   if d == nil {
      // 必须是2的倍数？应该跟后续双倍扩容，dequeueLimit判断。不浪费最后的空间
      const initSize = 8 // Must be a power of 2
      d = new(poolChainElt)
      // 只存8个数据。
      d.vals = make([]eface, initSize)
      c.head = d
     // 因为如果你不传二级指针(指针的指针)，那么就是指针的副本赋值了，这样tail其实本身没有赋值
      storePoolChainElt(&c.tail, d)
   }
	 // 都是对poolDequeue赋值，操作。
   if d.pushHead(val) {
      return
   }
   // 队列满时，双倍扩容
   newSize := len(d.vals) * 2
   // 边界判断一把
   if newSize >= dequeueLimit {
      // Can't make it any bigger.
      newSize = dequeueLimit
   }
	 // 双向链表链接prev即指向d->c.head->上一个的自身d2
   d2 := &poolChainElt{prev: d}
   d2.vals = make([]eface, newSize)
   // 把头节点始终指向最新对象
   c.head = d2
   // 并且刚才的下一个指针指向刚创建的对象。如图
   storePoolChainElt(&d.next, d2)
  // 都是对poolDequeue赋值，操作。
   d2.pushHead(val)
}
```

- 看图，这里有多个pin获取到pid，并且获取到相应的poolLocal,这时候前置private已经存储。那么就将数据存储到队列中。
- 当前切片vals存储满时，就扩容poolChainElt。并且增加里头的存储值vals扩容两倍。之后拼接链表
- 这里双端循环列表的操作思想，和我们当时分析过[redigo连接池源码解析](https://www.jianshu.com/p/b8bd4f3d11b4)连接池的思想都是一样的。这里sync.pool的很多无锁操作思想，还是值得大家思考和学习的。

#### poolDequeue.pushHead存储具体的val的值

```go
// 存储vals和headTail值
func (d *poolDequeue) pushHead(val interface{}) bool {
  // popTail会抢其他的p来修改headTail值.
	ptrs := atomic.LoadUint64(&d.headTail)
  // 高32位赋给head、低32位赋给tail
	head, tail := d.unpack(ptrs)
  // len(d.vals)大小固定，head每次++。相等就说明队列满了
	if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
		return false
	}
  // 这里因为tail只会++，所以head大小会大于len大小的情况
	slot := &d.vals[head&uint32(len(d.vals)-1)]
	// slot会被其他的p popTail掉，导致队列还是满的？？？所以才要原子操作，没看懂
	typ := atomic.LoadPointer(&slot.typ)
	if typ != nil {
		return false
	}
	// 基础校验
	if val == nil {
		val = dequeueNil(nil)
	}
  // 这里unsafe.Pointer存储，指针操作内存块赋值
	*(*interface{})(unsafe.Pointer(slot)) = val
	// 这里就是给head高32位+1
	atomic.AddUint64(&d.headTail, 1<<dequeueBits)
	return true
}
```

- 注意其他p会`抢占`当前执行的队列。gmp里头，偷其他对列的思想。这里前提必须是自身的队列都取完了，后续的popTail我们会分析
- unpack取高32位给head、低32位赋值给tail
- 可以学习一下unsafe.Pointer这种存储操作。`c艹`基操,(\*interface{})看成(void\*)

### 先从本地获取数据shared.popHead()

```go
func (c *poolChain) popHead() (interface{}, bool) {
	d := c.head
  // 一直取，直到当前的队列获取不到数据，这里每个p put和get每次只能一个拥有pin()类似给p加锁
	for d != nil {
    // 从数组中获取数据
		if val, ok := d.popHead(); ok {
			return val, ok
		}
		// 后进先出
		d = loadPoolChainElt(&d.prev)
	}
	return nil, false
}
```

![image-20210715200935699](/Users/11126518/knowledge/dandy_blog/image/popHead.gif)

- 当c.head取完，就取下一个的d.pre如图
- 这里d.next没有清空，没有关系，因为新插入的时候，d.next会指向新的对象

### 队头取值

```go
func (d *poolDequeue) popHead() (interface{}, bool) {
   var slot *eface
   for {
      // 取headtail
      ptrs := atomic.LoadUint64(&d.headTail)
      head, tail := d.unpack(ptrs)
      // 校验队列是否为空
      if tail == head {
         return nil, false
      }

      // pop出后，head--
      head--
      ptrs2 := d.pack(head, tail)
      // cas比较当前headTail是否有其他的P修改过
      if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
         // 没人修改取出slot
         slot = &d.vals[head&uint32(len(d.vals)-1)]
         break
      }
   }
	 // 取出值
   val := *(*interface{})(unsafe.Pointer(slot))
   if val == dequeueNil(nil) {
      val = nil
   }
   // Zero the slot. Unlike popTail, this isn't racing with
   // pushHead, so we don't need to be careful here.
   // 这里说这里是没有race的，其实我也没搞明白为什么没有race
   *slot = eface{}
   return val, true
}
```

- 常见的无锁cas操作，因为当前的p的和其他偷过来的p，会同时操作headTail，并且当前的p中head--和偷的p使用popTail() tail--会操作同一个值

  ![image-20210715200935699](/Users/11126518/knowledge/dandy_blog/image/cas.gif)

- 这里slot直接赋值为空，官方注释说，这里没有race现象，可以直接清0

### 队尾取值

```go
func (d *poolDequeue) popTail() (interface{}, bool) {
   var slot *eface
   // 和队头取值cas思路一致，只是tail++
   for {
      ptrs := atomic.LoadUint64(&d.headTail)
      head, tail := d.unpack(ptrs)
      if tail == head {
         // Queue is empty.
         return nil, false
      }
      ptrs2 := d.pack(head, tail+1)
      if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
         // tail也会有大于len的情况.因为每次tail只++
         slot = &d.vals[tail&uint32(len(d.vals)-1)]
         break
      }
   }

   // 取值
   val := *(*interface{})(unsafe.Pointer(slot))
   if val == dequeueNil(nil) {
      val = nil
   }

   // 告知pushHead这个slot已经让出？？？没明白
   slot.val = nil
   atomic.StorePointer(&slot.typ, nil)
   // At this point pushHead owns the slot.
   return val, true
```

- 对比popHead，一个使用*slot = eface{}，一个使用atomic.StorePointer。都是清空，都是赋值nil，好像没啥并发问题。 这里存在疑惑？？？有清楚的小伙伴，欢迎评论区留言，到时候请教yyds-曹大

### 多线程下，如何获取下一个队列

```go
func (c *poolChain) popTail() (interface{}, bool) {
   // 队尾为空，返回
   d := loadPoolChainElt(&c.tail)
   if d == nil {
      return nil, false
   }

   for {
      // 先取出下一个节点，因为当前d节点获取的时候为空，执行的时候，另外一个p又插入的数据
      d2 := loadPoolChainElt(&d.next)
      // 获取vals切片数据 
      if val, ok := d.popTail(); ok {
         return val, ok
      }
      // 校验d2是否为空
      if d2 == nil {
         return nil, false
      }

      // 原子性cas对比d的值中途是否有改变
      if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&c.tail)), unsafe.Pointer(d), unsafe.Pointer(d2)) {
         // popTail为空，并且d2非空。 则清空上一个数据
         storePoolChainElt(&d2.prev, nil)
      }
      d = d2
   }
}
```

如何做到无锁线程安全的，还是很值得学习的：

1. 如果先d.popTail取出切片的数据，判断为空。 之后取下一个数据d.next。因为获取d2之前，另外一个P瞬间把d切片写满。 这时候d2不为空。接下去就会把d删除。会造成误删
2. 必须在poptail前，获取d2，并且d2非空。 popTail也为空，才能证明tail是永久性的为空。然后清除
3. 会不会出现即使先取出d2非空，然后popTail为空，但是一瞬间，popTail被写满呢。 不会，因为pushHead都是从头部插入，如果d2非空了，那么插入肯定是从d2当前节点(未写满)或者下一个节点d3开始节点，而不会写d这个节点。这里还是很难理解的。大家要结合pushHead只从头插入，并且每次都是递增的插入。如果可以队尾插入，是做不到无锁的。

### 从其他P偷队列处理，获取数据

```go
func (p *Pool) getSlow(pid int) interface{} {
   // 取出poolLocal数组大小
   size := runtime_LoadAcquintptr(&p.localSize) // load-acquire
   // 切片地址
   locals := p.local                            // load-consume
   // 遍历所有的poolLocal
   for i := 0; i < int(size); i++ {
      // pid+1去取poolLocal,就是当前的p可以取到其他poolLocal
      l := indexLocal(locals, (pid+i+1)%int(size))
      // 偷来的队列，只能从队尾读取数据
      if x, _ := l.shared.popTail(); x != nil {
         return x
      }
   }

   // 在从cache里取数据
   size = atomic.LoadUintptr(&p.victimSize)
   if uintptr(pid) >= size {
      return nil
   }
   locals = p.victim
   l := indexLocal(locals, pid)
   if x := l.private; x != nil {
      l.private = nil
      return x
   }
   for i := 0; i < int(size); i++ {
      l := indexLocal(locals, (pid+i)%int(size))
      if x, _ := l.shared.popTail(); x != nil {
         return x
      }
   }
   // Mark the victim cache as empty for future gets don't bother
   // with it.
   atomic.StoreUintptr(&p.victimSize, 0)

   return nil
}
```

- 偷取的队列，只能从队尾获取数据
- 如果队列数据都为空，则从cache中获取数据

### 总结

- Put只能从对头插入，并且每个p只能插入自己的队列，所以可以理解为线程安全的。
- Get获取先从private -> shared -> 循环其他P -> victim cache -> New
- poolLocal和GMP中的P大小一致，GMP中的思想，尽可能的提高性能。
- 多个地方使用无锁思想。这里特别强调为什么每个p来说，是单个生产者，多个消费者模式。并且只能队头插入，如果对尾也可以插入，这种做法是无法做到无锁操作的。
- 引入victim cache，gc中新老年代、java中COWLIST等思想都是类似，降低 GC 压力的同时提高命中率。

### 疑问

- 切片清空的时候，popHead和popTail中，一个使用*slot = eface{}、一个使用atomic.StorePointer。都是赋值为空。还不清楚为什么，虽然说popTail是非线程安全的，但是为什么这么做。

还是有一些优秀的内存池如，[groupcache](https://github.com/golang/groupcache)、[bigcache](https://github.com/allegro/bigcache)等后续值得学习

### 大家可以添加我的微信一起探讨

我是一个爱扣源码细节的dandy，码字不易，点个小赞，只希望大家能更加明白

![image-20210715200935699](/Users/11126518/knowledge/dandy_blog/image/dandyhuang.jpeg)

### Reference

[golang的对象池sync.pool源码解读](https://zhuanlan.zhihu.com/p/99710992)

[深度解密Go语言之sync.pool](https://www.cnblogs.com/qcrao-2018/p/12736031.html)

[Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms](https://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf)

