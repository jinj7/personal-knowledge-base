URCU提供了五种flavors of RCU，使用最广的就是QSBR (quiescent-state-based RCU)。

# 1. rcu_register_thread

函数rcu_register_thread()在QSBR下的具体实现是函数urcu_qsbr_register_thread()。

```c
void urcu_qsbr_register_thread(void)
{
    URCU_TLS(urcu_qsbr_reader).tid = pthread_self();
    assert(URCU_TLS(urcu_qsbr_reader).ctr == 0);

    mutex_lock(&rcu_registry_lock);
    assert(!URCU_TLS(urcu_qsbr_reader).registered);
    URCU_TLS(urcu_qsbr_reader).registered = 1;
    cds_list_add(&URCU_TLS(urcu_qsbr_reader).node, &registry);
    mutex_unlock(&rcu_registry_lock);
    _urcu_qsbr_thread_online();
}

static inline void _urcu_qsbr_thread_online(void)
{
    urcu_assert(URCU_TLS(urcu_qsbr_reader).registered);
    cmm_barrier();  /* Ensure the compiler does not reorder us with mutex */
    _CMM_STORE_SHARED(URCU_TLS(urcu_qsbr_reader).ctr, CMM_LOAD_SHARED(urcu_qsbr_gp.ctr));
    cmm_smp_mb();
}
```

URCU_TLS(urcu_qsbr_reader)是个Thread-local storage (TLS)的变量，它的类型定义为：

```c
struct urcu_qsbr_reader {
    /* Data used by both reader and synchronize_rcu() */
    unsigned long ctr;
    /* Data used for registry */
    struct cds_list_head node __attribute__((aligned(CAA_CACHE_LINE_SIZE)));
    int waiting;
    pthread_t tid;
    /* Reader registered flag, for internal checks. */
    unsigned int registered:1;
};
```

函数urcu_qsbr_register_thread()的代码逻辑：

1. 赋值线程对应的URCU_TLS(urcu_qsbr_reader)；
2. 加锁（rcu_registry_lock）将线程对应的URCU_TLS(urcu_qsbr_reader)挂载到以registry为链头的全局链表中；
3. 调用函数_urcu_qsbr_thread_online()，使用计数器urcu_qsbr_gp.ctr来更新线程计数器URCU_TLS(urcu_qsbr_reader).ctr。

&nbsp;


# 2. rcu_quiescent_state

函数rcu_quiescent_state()在QSBR下的具体实现是函数_urcu_qsbr_quiescent_state()。

```c
static inline void _urcu_qsbr_quiescent_state(void)
{
    unsigned long gp_ctr;

    urcu_assert(URCU_TLS(urcu_qsbr_reader).registered);
    if ((gp_ctr = CMM_LOAD_SHARED(urcu_qsbr_gp.ctr)) == URCU_TLS(urcu_qsbr_reader).ctr)
        return;
    _urcu_qsbr_quiescent_state_update_and_wakeup(gp_ctr);
}

static inline void _urcu_qsbr_quiescent_state_update_and_wakeup(unsigned long gp_ctr)
{
    cmm_smp_mb();
    _CMM_STORE_SHARED(URCU_TLS(urcu_qsbr_reader).ctr, gp_ctr);
    cmm_smp_mb();   /* write URCU_TLS(urcu_qsbr_reader).ctr before read futex */
    urcu_qsbr_wake_up_gp();
    cmm_smp_mb();
}

static inline void urcu_qsbr_wake_up_gp(void)
{
    if (caa_unlikely(_CMM_LOAD_SHARED(URCU_TLS(urcu_qsbr_reader).waiting))) {
        _CMM_STORE_SHARED(URCU_TLS(urcu_qsbr_reader).waiting, 0);
        cmm_smp_mb();
        if (uatomic_read(&urcu_qsbr_gp.futex) != -1)
            return;
        uatomic_set(&urcu_qsbr_gp.futex, 0);
        /*
         * Ignoring return value until we can make this function
         * return something (because urcu_die() is not publicly
         * exposed).
         */
        (void) futex_noasync(&urcu_qsbr_gp.futex, FUTEX_WAKE, 1,
                NULL, NULL, 0);
    }
}
```

全局变量urcu_qsbr_gp的数据类型为：

```c
struct urcu_gp {
    /*
     * Global grace period counter.
     * Contains the current URCU_GP_CTR_PHASE.
     * Also has a URCU_GP_COUNT of 1, to accelerate the reader fast path.
     * Written to only by writer with mutex taken.
     * Read by both writer and readers.
     */
    unsigned long ctr;

    int32_t futex;
} __attribute__((aligned(CAA_CACHE_LINE_SIZE)));
```

函数_urcu_qsbr_quiescent_state()的代码逻辑为：

1. 读取计数器urcu_qsbr_gp.ctr的当前值，与线程计数器URCU_TLS(urcu_qsbr_reader).ctr相比较，如果相等，则立即返回；
2. 否则，线程计数器URCU_TLS(urcu_qsbr_reader).ctr同步更新为计数器urcu_qsbr_gp.ctr；
3. 调用函数urcu_qsbr_wake_up_gp()。
   1. 如果URCU_TLS(urcu_qsbr_reader).waiting当前值为1
      1. URCU_TLS(urcu_qsbr_reader).waiting复位为0；
      2. 如果urcu_qsbr_gp.futex当前值不等于-1，则立即返回；
      3. 否则，urcu_qsbr_gp.futex当前值为-1，复位为0；
      4. 唤醒关联urcu_qsbr_gp.futex的一个writer线程。

&nbsp;


# 3. synchronize_rcu

函数synchronize_rcu()在QSBR下的具体实现是函数urcu_qsbr_synchronize_rcu()。

```c
void urcu_qsbr_synchronize_rcu(void)
{
    CDS_LIST_HEAD(qsreaders);
    unsigned long was_online;
    DEFINE_URCU_WAIT_NODE(wait, URCU_WAIT_WAITING);
    struct urcu_waiters waiters;

    was_online = urcu_qsbr_read_ongoing();

    /*
     * Mark the writer thread offline to make sure we don't wait for
     * our own quiescent state. This allows using synchronize_rcu()
     * in threads registered as readers.
     */
    if (was_online)
        urcu_qsbr_thread_offline();
    else
        cmm_smp_mb();

    /*
     * Add ourself to gp_waiters queue of threads awaiting to wait
     * for a grace period. Proceed to perform the grace period only
     * if we are the first thread added into the queue.
     */
    if (urcu_wait_add(&gp_waiters, &wait) != 0) {
        /* Not first in queue: will be awakened by another thread. */
        urcu_adaptative_busy_wait(&wait);
        goto gp_end;
    }
    /* We won't need to wake ourself up */
    urcu_wait_set_state(&wait, URCU_WAIT_RUNNING);

    mutex_lock(&rcu_gp_lock);

    /*
     * Move all waiters into our local queue.
     */
    urcu_move_waiters(&waiters, &gp_waiters);

    mutex_lock(&rcu_registry_lock);

    if (cds_list_empty(&registry))
        goto out;

    /* Increment current G.P. */
    CMM_STORE_SHARED(urcu_qsbr_gp.ctr, urcu_qsbr_gp.ctr + URCU_QSBR_GP_CTR);

    /*
     * Must commit urcu_qsbr_gp.ctr update to memory before waiting for
     * quiescent state. Failure to do so could result in the writer
     * waiting forever while new readers are always accessing data
     * (no progress). Enforce compiler-order of store to urcu_qsbr_gp.ctr
     * before load URCU_TLS(urcu_qsbr_reader).ctr.
     */
    cmm_barrier();

    /*
     * Adding a cmm_smp_mb() which is _not_ formally required, but makes the
     * model easier to understand. It does not have a big performance impact
     * anyway, given this is the write-side.
     */
    cmm_smp_mb();

    /*
     * Wait for readers to observe new count of be quiescent.
     * wait_for_readers() can release and grab again rcu_registry_lock
     * interally.
     */
    wait_for_readers(&registry, NULL, &qsreaders);

    /*
     * Put quiescent reader list back into registry.
     */
    cds_list_splice(&qsreaders, &registry);
out:
    mutex_unlock(&rcu_registry_lock);
    mutex_unlock(&rcu_gp_lock);
    urcu_wake_all_waiters(&waiters);
gp_end:
    if (was_online)
        urcu_qsbr_thread_online();
    else
        cmm_smp_mb();
}
```

函数urcu_qsbr_synchronize_rcu()的代码逻辑为：

1. 调用函数urcu_qsbr_read_ongoing()，获取线程计数器URCU_TLS(urcu_qsbr_reader).ctr的当前值；

2. 如果线程计数器URCU_TLS(urcu_qsbr_reader).ctr当前值不为0，i.e. 当前线程既是writer，又是reader；

   1. 调用函数urcu_qsbr_thread_offline()，将当前线程设置为offline状态，保证不会等待自身的quiescent state：
      1. 线程计数器URCU_TLS(urcu_qsbr_reader).ctr复位为0；
      2. 调用函数urcu_qsbr_wake_up_gp()。

3. 调用函数urcu_wait_add() --> cds_wfs_push() --> _cds_wfs_push()，将代表当前线程的wait node，添加到到全局变量gp_waiters所定义的wait queue中：

   ```c
   /*
    * Add ourself atomically to a wait queue. Return 0 if queue was
    * previously empty, else return 1.
    * A full memory barrier is issued before being added to the wait queue.
    */
   static inline
   bool urcu_wait_add(struct urcu_wait_queue *queue,
           struct urcu_wait_node *node)
   {
       return cds_wfs_push(&queue->stack, &node->node);
   }
   
   /*
    * cds_wfs_push: push a node into the stack.
    *
    * Issues a full memory barrier before push. No mutual exclusion is
    * required.
    *
    * Returns 0 if the stack was empty prior to adding the node.
    * Returns non-zero otherwise.
    */
   static inline
   int _cds_wfs_push(cds_wfs_stack_ptr_t u_stack, struct cds_wfs_node *node)
   {
       struct __cds_wfs_stack *s = u_stack._s;
       struct cds_wfs_head *old_head, *new_head;
   
       assert(node->next == NULL);
       new_head = caa_container_of(node, struct cds_wfs_head, node);
       /*
        * uatomic_xchg() implicit memory barrier orders earlier stores
        * to node (setting it to NULL) before publication.
        */
       old_head = uatomic_xchg(&s->head, new_head);
       /*
        * At this point, dequeuers see a NULL node->next, they should
        * busy-wait until node->next is set to old_head.
        */
       CMM_STORE_SHARED(node->next, &old_head->node);
       return !___cds_wfs_end(old_head);
   }
   ```

   cds_wfs_stack_ptr_t用于遍历stack，stack是用链表来实现的。

   1. 栈顶指针u_stack._s->head指向cds_wfs_node{}结构（i.e. 入栈），变量old_head保存了更新前的栈顶节点；
   2. 新栈顶节点重新挂载到stack链表中；
   3. 如果更新前原始stack链表是empty，函数返回0；否则，返回1。

4. 如果函数urcu_wait_add()的返回值不为0，说明当前线程并不是第一个被添加到全局wait queue队列中的线程；

   1. 调用函数urcu_adaptative_busy_wait()；
   2. 直接跳转到Label gp_end执行。

5. 代码执行到这里，说明当前线程是第一个被添加到wait queue队列中的线程；

6. 调用函数urcu_wait_set_state()：设置当前线程对应的urcu_wait_node{}结构状态为URCU_WAIT_RUNNING；

7. 调用函数urcu_move_waiters()，将全局wait queue中的所有节点移动到当前线程local wait queue -- waiters中

   ```c
   /*
    * Atomically move all waiters from wait queue into our local struct
    * urcu_waiters.
    */
   static inline
   void urcu_move_waiters(struct urcu_waiters *waiters,
           struct urcu_wait_queue *queue)
   {
       waiters->head = __cds_wfs_pop_all(&queue->stack);
   }
   
   static inline
   struct cds_wfs_head *
   ___cds_wfs_pop_all(cds_wfs_stack_ptr_t u_stack)
   {
       struct __cds_wfs_stack *s = u_stack._s;
       struct cds_wfs_head *head;
   
       /*
        * Implicit memory barrier after uatomic_xchg() matches implicit
        * memory barrier before uatomic_xchg() in cds_wfs_push. It
        * ensures that all nodes of the returned list are consistent.
        * There is no need to issue memory barriers when iterating on
        * the returned list, because the full memory barrier issued
        * prior to each uatomic_cmpxchg, which each write to head, are
        * taking care to order writes to each node prior to the full
        * memory barrier after this uatomic_xchg().
        */
       head = uatomic_xchg(&s->head, CDS_WFS_END);
       if (___cds_wfs_end(head))
           return NULL;
       return head;
   }
   ```

   1. 栈顶指针u_stack._s->head指向CDS_WFS_END；
   2. 原来的stack栈顶节点保存在urcu_waiters{}->head中。

8. 如果以registry为链头的全局链表为空，直接跳转到Label out执行；

9. 计数器urcu_qsbr_gp.ctr递增2；

10. 调用函数wait_for_readers()，等待grace period结束，最终将所有reader线程的urcu_qsbr_reader{}结构节点，从registry链表挪动到qsreaders链表中；

11. 调用函数cds_list_splice()，将所有reader线程的urcu_qsbr_reader{}结构节点，重新挪回到registry链表中，等待下一次同步；

12. Label out

13. 调用函数urcu_wake_all_waiters()，唤醒wait queue队列上等待的其它线程；

14. Label gp_end

15. 呼应step 2，如果当前线程既是writer，又是reader；

    1. 调用函数urcu_qsbr_thread_online()，将当前线程设置为online状态；
       1. 线程计数器URCU_TLS(urcu_qsbr_reader).ctr，更新为计数器urcu_qsbr_gp.ctr当前值。

## 3.1 wait_for_readers

```c
static void wait_for_readers(struct cds_list_head *input_readers,
            struct cds_list_head *cur_snap_readers,
            struct cds_list_head *qsreaders)
{
    unsigned int wait_loops = 0;
    struct urcu_qsbr_reader *index, *tmp;

    /*
     * Wait for each thread URCU_TLS(urcu_qsbr_reader).ctr to either
     * indicate quiescence (offline), or for them to observe the
     * current urcu_qsbr_gp.ctr value.
     */
    for (;;) {
        if (wait_loops < RCU_QS_ACTIVE_ATTEMPTS)
            wait_loops++;
        if (wait_loops >= RCU_QS_ACTIVE_ATTEMPTS) {
            uatomic_set(&urcu_qsbr_gp.futex, -1);
            /*
             * Write futex before write waiting (the other side
             * reads them in the opposite order).
             */
            cmm_smp_wmb();
            cds_list_for_each_entry(index, input_readers, node) {
                _CMM_STORE_SHARED(index->waiting, 1);
            }
            /* Write futex before read reader_gp */
            cmm_smp_mb();
        }
        cds_list_for_each_entry_safe(index, tmp, input_readers, node) {
            switch (urcu_qsbr_reader_state(&index->ctr)) {
            case URCU_READER_ACTIVE_CURRENT:
                if (cur_snap_readers) {
                    cds_list_move(&index->node,
                        cur_snap_readers);
                    break;
                }
                /* Fall-through */
            case URCU_READER_INACTIVE:
                cds_list_move(&index->node, qsreaders);
                break;
            case URCU_READER_ACTIVE_OLD:
                /*
                 * Old snapshot. Leaving node in
                 * input_readers will make us busy-loop
                 * until the snapshot becomes current or
                 * the reader becomes inactive.
                 */
                break;
            }
        }

        if (cds_list_empty(input_readers)) {
            if (wait_loops >= RCU_QS_ACTIVE_ATTEMPTS) {
                /* Read reader_gp before write futex */
                cmm_smp_mb();
                uatomic_set(&urcu_qsbr_gp.futex, 0);
            }
            break;
        } else {
            /* Temporarily unlock the registry lock. */
            mutex_unlock(&rcu_registry_lock);
            if (wait_loops >= RCU_QS_ACTIVE_ATTEMPTS) {
                wait_gp();
            } else {
#ifndef HAS_INCOHERENT_CACHES
                caa_cpu_relax();
#else /* #ifndef HAS_INCOHERENT_CACHES */
                cmm_smp_mb();
#endif /* #else #ifndef HAS_INCOHERENT_CACHES */
            }
            /* Re-lock the registry lock before the next loop. */
            mutex_lock(&rcu_registry_lock);
        }
    }
}
```

大循环的代码逻辑是：

1. 局部变量wait_loops记录循环次数，依次递增，直到上限RCU_QS_ACTIVE_ATTEMPTS；

2. 如果循环次数wait_loops到达上限RCU_QS_ACTIVE_ATTEMPTS；

   1. urcu_qsbr_gp.futex设置为-1；
   2. 遍历以输入参数input_readers为链头的urcu_qsbr_reader{}结构节点链表；
      1. 每个reader线程的urcu_qsbr_reader{}->waiting设置为1。

3. 遍历以输入参数input_readers为链头的urcu_qsbr_reader{}结构节点链表（每个节点就是reader线程对应的URCU_TLS(urcu_qsbr_reader)）；

   1. 调用函数urcu_qsbr_reader_state()，根据线程计数器urcu_qsbr_reader{}->ctr的当前值来判断reader线程的状态；
      1. 0值：返回URCU_READER_INACTIVE，不参与RCU同步；
      2. 与计数器urcu_qsbr_gp.ctr当前值相等：返回URCU_READER_ACTIVE_CURRENT，说明reader线程已进入quiescent state；
      3. 与计数器urcu_qsbr_gp.ctr当前值不相等：返回URCU_READER_ACTIVE_OLD，说明reader线程还在RCU read-side critical section。
   2. 根据获取的reader线程当前的不同状态，分类进行处理；
      1. URCU_READER_INACTIVE和URCU_READER_ACTIVE_CURRENT（输入参数cur_snap_readers为NULL）：将reader线程对应的urcu_qsbr_reader{}结构节点，移动到以输入参数qsreaders为链头的链表上；
      2. URCU_READER_ACTIVE_OLD：nop，i.e. reader线程对应的urcu_qsbr_reader{}结构节点继续保留在以输入参数input_readers为链头的链表上。

4. 如果以输入参数input_readers为链头的urcu_qsbr_reader{}结构节点链表变为空，i.e. grace period已结束；

   1. 如果循环次数wait_loops到达上限RCU_QS_ACTIVE_ATTEMPTS，urcu_qsbr_gp.futex复位为0；
   2. 结束大循环。

5. 否则，reader线程对应的urcu_qsbr_reader{}结构节点并没有完全从input_readers链表，移动到qsreaders链表中，i.e. 仍然有reader线程还在RCU read-side critical section；

   1. 如果循环次数wait_loops到达上限RCU_QS_ACTIVE_ATTEMPTS，调用函数wait_gp()：

      ```c
      /*
       * synchronize_rcu() waiting. Single thread.
       */
      static void wait_gp(void)
      {
          /* Read reader_gp before read futex */
          cmm_smp_rmb();
          if (uatomic_read(&urcu_qsbr_gp.futex) != -1)
              return;
          while (futex_noasync(&urcu_qsbr_gp.futex, FUTEX_WAIT, -1,
                  NULL, NULL, 0)) {
              switch (errno) {
              case EWOULDBLOCK:
                  /* Value already changed. */
                  return;
              case EINTR:
                  /* Retry if interrupted by signal. */
                  break;  /* Get out of switch. */
              default:
                  /* Unexpected error. */
                  urcu_die(errno);
              }
          }
      }
      ```

      1. 如果urcu_qsbr_gp.futex当前值不等于-1，则立即返回；
      2. 如果urcu_qsbr_gp.futex当前值仍然是-1，当前线程睡眠，等待被其它reader线程唤醒。

## 3.2 urcu_adaptative_busy_wait

```c
/*
 * Caller must initialize "value" to URCU_WAIT_WAITING before passing its
 * memory to waker thread.
 */
static inline
void urcu_adaptative_busy_wait(struct urcu_wait_node *wait)
{
    unsigned int i;

    /* Load and test condition before read state */
    cmm_smp_rmb();
    for (i = 0; i < URCU_WAIT_ATTEMPTS; i++) {
        if (uatomic_read(&wait->state) != URCU_WAIT_WAITING)
            goto skip_futex_wait;
        caa_cpu_relax();
    }
    while (futex_noasync(&wait->state, FUTEX_WAIT, URCU_WAIT_WAITING,
            NULL, NULL, 0)) {
        switch (errno) {
        case EWOULDBLOCK:
            /* Value already changed. */
            goto skip_futex_wait;
        case EINTR:
            /* Retry if interrupted by signal. */
            break;  /* Get out of switch. */
        default:
            /* Unexpected error. */
            urcu_die(errno);
        }
    }
skip_futex_wait:

    /* Tell waker thread than we are running. */
    uatomic_or(&wait->state, URCU_WAIT_RUNNING);

    /*
     * Wait until waker thread lets us know it's ok to tear down
     * memory allocated for struct urcu_wait.
     */
    for (i = 0; i < URCU_WAIT_ATTEMPTS; i++) {
        if (uatomic_read(&wait->state) & URCU_WAIT_TEARDOWN)
            break;
        caa_cpu_relax();
    }
    while (!(uatomic_read(&wait->state) & URCU_WAIT_TEARDOWN))
        poll(NULL, 0, 10);
    assert(uatomic_read(&wait->state) & URCU_WAIT_TEARDOWN);
}
```

wait node初始状态设置为URCU_WAIT_WAITING。

1. 使用futex等待被其它线程唤醒前，先busy-loop（最多尝试URCU_WAIT_ATTEMPTS次）等待wait node的当前状态变为非URCU_WAIT_WAITING值，如果成功直接跳转到Label skip_futex_wait执行；
2. 如果wait node的当前状态仍然是URCU_WAIT_WAITING，当前线程睡眠，等待被其它线程唤醒；
3. Label skip_futex_wait
4. wait node当前状态添加URCU_WAIT_RUNNING标志；
5. 继续busy-loop（最多尝试URCU_WAIT_ATTEMPTS次）等待wait node的当前状态被设置URCU_WAIT_TEARDOWN标志；
6. 使用poll睡眠一直等待wait node的当前状态被设置URCU_WAIT_TEARDOWN标志。

## 3.3 urcu_wake_all_waiters

```c
static inline
void urcu_wake_all_waiters(struct urcu_waiters *waiters)
{
    struct cds_wfs_node *iter, *iter_n;

    /* Wake all waiters in our stack head */
    cds_wfs_for_each_blocking_safe(waiters->head, iter, iter_n) {
        struct urcu_wait_node *wait_node =
            caa_container_of(iter, struct urcu_wait_node, node);

        /* Don't wake already running threads */
        if (wait_node->state & URCU_WAIT_RUNNING)
            continue;
        urcu_adaptative_wake_up(wait_node);
    }
}
```

执行cds_wfs_for_each_blocking_safe()循环遍历wait queue上的所有wait node：

1. 如果wait node的当前状态已经变为URCU_WAIT_RUNNING，继续处理后面的wait node（调用函数urcu_qsbr_synchronize_rcu()时，step 6已经将当前线程对应的wait node状态设置为URCU_WAIT_RUNNING）；
2. 调用函数urcu_adaptative_wake_up()；
   1. wait node的当前状态被设置为URCU_WAIT_WAKEUP；
   2. 如果wait node的当前状态没有设置URCU_WAIT_RUNNING标志；
      1. 唤醒关联wait node的当前状态的一个其它线程。
   3. wait node的当前状态被设置URCU_WAIT_TEARDOWN标志。

&nbsp;


# 4. Summary

## 4.1 Main Procedure

1. [reader] [rcu_register_thread()]，挂载registry链表，更新线程计数器URCU_TLS(urcu_qsbr_reader).ctr；
2. [writer] [urcu_qsbr_synchronize_rcu()]，计数器urcu_qsbr_gp.ctr被更新；
3. [reader] [rcu_quiescent_state()]，某些reader线程较快完成RCU read-side critical section的访问时，线程计数器URCU_TLS(urcu_qsbr_reader).ctr同步更新为计数器urcu_qsbr_gp.ctr；
4. [writer] [wait_for_readers()]，最多尝试RCU_QS_ACTIVE_ATTEMPTS次，将线程计数器urcu_qsbr_reader{}->ctr已完成更新的reader线程，从registry链表挪动到qsreaders链表中；
5. [writer] [wait_for_readers()]，如果registry链表变为空，说明grace period已结束；
6. [writer] [wait_for_readers()]，如果尝试次数达到了RCU_QS_ACTIVE_ATTEMPTS，先将urcu_qsbr_gp.futex设置为-1，再将未同步的所有reader线程的urcu_qsbr_reader{}->waiting设置为1；
7. [writer] [wait_for_readers()]，继续检查reader线程的同步情况（同step 4）；
8. [writer] [wait_gp()]，如果urcu_qsbr_gp.futex当前值仍然是-1，writer线程睡眠，等待被其它reader线程唤醒；
9. [reader] [rcu_quiescent_state()]，某些reader线程较慢完成RCU read-side critical section的访问时，线程计数器URCU_TLS(urcu_qsbr_reader).ctr才同步更新为计数器urcu_qsbr_gp.ctr；
10. [reader] [urcu_qsbr_wake_up_gp()]，reader线程URCU_TLS(urcu_qsbr_reader).waiting复位为0，urcu_qsbr_gp.futex从-1复位为0，唤醒关联urcu_qsbr_gp.futex的一个writer线程；
11. [writer] [wait_for_readers()]，writer线程被唤醒，urcu_qsbr_gp.futex重新设置为-1，刚已同步的reader线程和仍未同步的reader线程，对应的urcu_qsbr_reader{}->waiting仍然设置为1（同step 6）；
12. [writer] [wait_for_readers()]，继续检查reader线程的同步情况，刚已同步的reader线程，从registry链表挪动到qsreaders链表中（同step 4、7）；
13. [writer] [wait_for_readers()]，重复执行step 8、9、10等，直至registry链表变为空，说明grace period已结束，urcu_qsbr_gp.futex复位为0；
14. [writer] [urcu_qsbr_synchronize_rcu()]，将所有reader线程的urcu_qsbr_reader{}结构节点，重新挪回到registry链表中，等待下一次同步；
15. 同步过程结束。

## 4.2 Wait Node / Wait Queue

- 使用场景：两个writer同时调用了函数synchronize_rcu()，等待grace period结束；
- 第一个writer线程来完成等待grace period结束的工作，最后调用函数urcu_wake_all_waiters()来唤醒其它writer线程；
- 其它writer线程调用函数urcu_adaptative_busy_wait()，等待第一个writer线程完成工作后来唤醒它们。