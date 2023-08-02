## 并发控制(Concurrency control)-互斥

### 共享内存上的互斥

实现互斥的根本困难：不能同时读/写共享内存

- load (环顾四周) 的时候不能写，只能 “看一眼就把眼睛闭上”
  看到的东西马上就过时了
- store (改变物理世界状态) 的时候不能读，只能 “闭着眼睛动手”
  也不知道把什么改成了什么
- 这是简单、粗暴 (稳定)、有效的《操作系统》课

### 自旋锁(Spin Lock)

假设硬件能为我们提供一条 “瞬间完成” 的读 + 写指令

- 请所有人闭上眼睛，看一眼 (load)，然后贴上标签 (store)

  - 如果多人同时请求，硬件选出一个 “胜者”
  - “败者” 要等 “胜者” 完成后才能继续执行

#### x86 原子操作：LOCK 指令前缀

例子：sum-atomic.c

    sum = 200000000

Atomic exchange (load + store)

        int xchg(volatile int *addr, int newval) {
          int result;
          asm volatile ("lock xchg %0, %1"
            : "+m"(*addr), "=a"(result) : "1"(newval));
          return result;
        }

#### 用 xchg 实现互斥

如何协调宿舍若干位同学上厕所问题？

    在厕所门口放一个桌子 (共享变量)
    初始时，桌上是 🔑
    实现互斥的协议

    想上厕所的同学 (一条 xchg 指令)
    天黑请闭眼
    看一眼桌子上有什么 (🔑 或 🔞)
    把 🔞 放到桌上 (覆盖之前有的任何东西)
    天亮请睁眼；看到 🔑 才可以进厕所哦
    出厕所的同学
    把 🔑 放到桌上

#### 实现互斥：自旋锁

    int table = YES;

    void lock() {
    retry:
      int got = xchg(&table, NOPE);
      if (got == NOPE)
        goto retry;
      assert(got == YES);
    }

    void unlock() {
      xchg(&table, YES)
    }
    int locked = 0;
    void lock() { while (xchg(&locked, 1)) ; }
    void unlock() { xchg(&locked, 0); }

并发编程：千万小心

- 做详尽的测试 (在此省略，你们做 Labs 就知道了)
- 尽可能地证明 (model-checker.py 和 spinlock.py)

原子指令的模型

- 保证之前的 store 都写入内存
- 保证 load/store 不与原子指令乱序

#### 原子指令的诞生：Bus Lock (80486)

486 (20-50MHz) 就支持 dual-socket 了

#### Lock 指令的现代实现

在 L1 cache 层保持一致性 (ring/mesh bus)

- 相当于每个 cache line 有分别的锁
- store(x) 进入 L1 缓存即保证对其他处理器可见
  - 但要小心 store buffer 和乱序执行

L1 cache line 根据状态进行协调

- M (Modified), 脏值
- E (Exclusive), 独占访问
- S (Shared), 只读共享
- I (Invalid), 不拥有 cache line

#### RISC-V: 另一种原子操作的设计

考虑常见的原子操作：

- atomic test-and-set
- reg = load(x); if (reg == XX) { store(x, YY); }
- lock xchg
- reg = load(x); store(x, XX);
- lock add
- t = load(x); t++; store(x, t);

它们的本质都是：

- load
- exec (处理器本地寄存器的运算)
- store

#### Load-Reserved/Store-Conditional (LR/SC)

LR: 在内存上标记 reserved (盯上你了)，中断、其他处理器写入都会导致标记消除

    lr.w rd, (rs1)
      rd = M[rs1]
      reserve M[rs1]

SC: 如果 “盯上” 未被解除，则写入

    sc.w rd, rs2, (rs1)
      if still reserved:
        M[rs1] = rs2
        rd = 0
      else:
        rd = nonzero

#### Compare-and-Swap 的 LR/SC 实现

    int cas(int *addr, int cmp_val, int new_val) {
      int old_val = *addr;
      if (old_val == cmp_val) {
        *addr = new_val; return 0;
      } else { return 1; }
    }

    cas:
      lr.w  t0, (a0)       # Load original value.
      bne   t0, a1, fail   # Doesn’t match, so fail.
      sc.w  t0, a2, (a0)   # Try to update.
      bnez  t0, cas        # Retry if store-conditional failed.
      li a0, 0             # Set return to success.
      jr ra                # Return.
    fail:
      li a0, 1             # Set return to failure.
      jr ra                # Return

#### LR/SC 的硬件实现

BOOM (Berkeley Out-of-Order Processor)

- riscv-boom
  - lsu/dcache.scala
  - 留意 s2_sc_fail 的条件
    - s2 是流水线 Stage 2
  - (yzh 扒出的代码)

#### 自旋锁的缺陷

性能问题 (0)

- 自旋 (共享变量) 会触发处理器间的缓存同步，延迟增加

性能问题 (1)

- 除了进入临界区的线程，其他处理器上的线程都在空转
- 争抢锁的处理器越多，利用率越低

性能问题 (2)

- 获得自旋锁的线程可能被操作系统切换出去
  - 操作系统不 “感知” 线程在做什么
  - (但为什么不能呢？)
- 实现 100% 的资源浪费

#### Scalability: 性能的新维度

同一份计算任务，时间 (CPU cycles) 和空间 (mapped memory) 会随处理器数量的增长而变化。

- sum-scalability.c
- thread-sync.h
  - 严谨的统计很难
    - CPU 动态功耗
    - 系统中的其他进程
    - ……
  - Benchmarking crimes

#### 自旋锁的使用场景

1. 临界区几乎不 “拥堵”
2. 持有自旋锁时禁止执行流切换

使用场景：操作系统内核的并发数据结构 (短临界区)

- 操作系统可以关闭中断和抢占
  - 保证锁的持有者在很短的时间内可以释放锁
- (如果是虚拟机呢...😂)
  - PAUSE 指令会触发 VM Exit
- 但依旧很难做好
  - An analysis of Linux scalability to many cores (OSDI'10)

### 互斥锁(Mutex Lock)

#### 实现线程 + 长临界区的互斥

> 作业那么多，与其干等 Online Judge 发布，不如把自己 (CPU) 让给其他作业 (线程) 执行？

“让” 不是 C 语言代码可以做到的 (C 代码只能计算)

- 把锁的实现放到操作系统里就好啦！
  - syscall(SYSCALL_lock, &lk);
    - 试图获得 lk，但如果失败，就切换到其他线程
  - syscall(SYSCALL_unlock, &lk);
    - 释放 lk，如果有等待锁的线程就唤醒

#### 实现线程 + 长临界区的互斥 (cont'd)

操作系统 = 更衣室管理员

- 先到的人 (线程)
  - 成功获得手环，进入游泳馆
  - \*lk = 🔒，系统调用直接返回
- 后到的人 (线程)
  - 不能进入游泳馆，排队等待
  - 线程放入等待队列，执行线程切换 (yield)
- 洗完澡出来的人 (线程)
  - 交还手环给管理员；管理员把手环再交给排队的人
  - 如果等待队列不空，从等待队列中取出一个线程允许执行
  - 如果等待队列为空，\*lk = ✅
- 管理员 (OS) 使用自旋锁确保自己处理手环的过程是原子的

### Futex = Spin + Mutex

#### 关于互斥的一些分析

自旋锁 (线程直接共享 locked)

- 更快的 fast path
  - xchg 成功 → 立即进入临界区，开销很小
- 更慢的 slow path
  - xchg 失败 → 浪费 CPU 自旋等待

睡眠锁 (通过系统调用访问 locked)

- 更快的 slow path
  - 上锁失败线程不再占用 CPU
- 更慢的 fast path
  - 即便上锁成功也需要进出内核 (syscall)

#### Futex: Fast Userspace muTexes

> 小孩子才做选择。我当然是全都要啦！

- Fast path: 一条原子指令，上锁成功立即返回
- Slow path: 上锁失败，执行系统调用睡眠
  - 性能优化的最常见技巧
    - 看 average (frequent) case 而不是 worst case

POSIX 线程库中的互斥锁 (pthread_mutex)

- sum-scalability.c，换成 mutex
  - 观察系统调用 (strace)
  - gdb 调试
    - set scheduler-locking on, info threads, thread X

#### Futex: Fast Userspace muTexes (cont'd)

先在用户空间自旋

- 如果获得锁，直接进入
- 未能获得锁，系统调用
- 解锁以后也需要系统调用
  - futex.py
  - 更好的设计可以在 fast-path 不进行系统调用

RTFM (劝退)

- futex (7), futex (2)
- A futex overview and update (LWN)
- Futexes are tricky (论 model checker 的重要性)
- (我们不讲并发算法)

### 总结

本次课回答的问题

- Q: 如何在多处理器系统上实现互斥？

Take-away message

- 软件不够，硬件来凑 (自旋锁)
- 用户不够，内核来凑 (互斥锁)
  - 找到你依赖的假设，并大胆地打破它
- Fast/slow paths: 性能优化的重要途径
