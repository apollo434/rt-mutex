#### Comment key code


**rt_mutex_slowlock**

1. 初始化rt_mutex的RB树。
2. try_to_take_rt_mutex,由于此处waiter参数为NULL，只有两种返回值：
1）lock没有waiter，当前进程获取锁。返回 1.2）lock上有waiter，当前进程没获得，所以返回0
3. 设置当前进程状态，如果有timeout的话，设置高精度定时器，并启动。
4. 调用task_blocks_on_rt_mutex来准备waiter，并调整PI chain。在这个过程中会多次检查时候能够获得锁：1）如果返回值为0，waiter就绪（在lock 以及owner的pi_waiters RB树中）并且PI chain的优先级调整及死锁检查已经完毕。下面准备让当前进程sleep wait。调用关键函数__rt_mutex_slowlock。 2）如果返回值为1，那么就是被信号打断或者timeout时间到了。不是被unlock唤醒的，所以需要在rt_mutex_lock中删除waiter.
5. 调用fixup_rt_mutex_waiters来设置task的owner，waiter的bit位，来进行后续调整（有一个race这里，后面介绍）


```
/*
 * Slow path lock function: 伪代码！！
 */
static int __sched
rt_mutex_slowlock(struct rt_mutex *lock, int state,
		  struct hrtimer_sleeper *timeout,
		  enum rtmutex_chainwalk chwalk)
{

  /* 初始化waiter
   * 初始化RB树
   */
  debug_rt_mutex_init_waiter(&waiter);
	RB_CLEAR_NODE(&waiter.pi_tree_entry);
	RB_CLEAR_NODE(&waiter.tree_entry);

  /*
   * 进入临界区，rtmutex的自旋锁保护
   */
  raw_spin_lock(&lock->wait_lock);


  /* Try to acquire the lock again: */
  /*
   * 第一次尝试获取rt_mutex
   * 调用try_to_wake_rt_mutex尝试获得rtmutex。该函数在后面__rt_mutex_slowlock还会被调用
   */
  if (try_to_take_rt_mutex(lock, current, NULL)) {
    raw_spin_unlock(&lock->wait_lock);
    return 0;
  }

  /* Setup the timer, when timeout != NULL */
  /*  
   * 如果设置了timeout值，调用内核高精度定时器，到期唤醒该进程
   */
	if (unlikely(timeout))
		hrtimer_start_expires(&timeout->timer, HRTIMER_MODE_ABS);

    /*
     * 调用task_blocks_on_rt_mutex来准备waiter，并调整PI chain。在这个过程中会多次检查时候能够获得锁
     */
    ret = task_blocks_on_rt_mutex(lock, &waiter, current, chwalk);

    if (likely(!ret))
  		/* sleep on the mutex */
      /*
       * 函数走到这边，waiter就绪（在lock 以及owner的pi_waiters RB树中）并且PI chain的优先级调整及死锁检查已经完毕。下面准备让当前进程sleep wait
       */
  		ret = __rt_mutex_slowlock(lock, state, timeout, &waiter);

    /*
     *如果返回值不为0的话，那么就是被信号打断或者timeout时间到了。不是被unlock唤醒的，所以需要在rt_mutex_lock中删除waiter.
     * 如果是被unlock唤醒的话，waiter的清理工作由unlock函数完成
     */
    if (unlikely(ret)) {
  		__set_current_state(TASK_RUNNING);
    		if (rt_mutex_has_waiters(lock))
    			remove_waiter(lock, &waiter);
    		rt_mutex_handle_deadlock(ret, chwalk, &waiter);
    	}

      /*
    	 * try_to_take_rt_mutex() sets the waiter bit
    	 * unconditionally. We might have to fix that up.
    	 */
       /*
        * 设置task的owner waiter的bit位
        */
    	fixup_rt_mutex_waiters(lock);

}

```

**try_to_take_rt_mutex**

```
/*
 * 依然是伪代码，只贴最重要的部分
 */
/*
 * Try to take an rt-mutex
 *
 * Must be called with lock->wait_lock held.
 *
 * @lock:   The lock to be acquired.
 * @task:   The task which wants to acquire the lock
 * @waiter: The waiter that is queued to the lock's wait tree if the
 *	    callsite called task_blocked_on_lock(), otherwise NULL
 */
static int try_to_take_rt_mutex(struct rt_mutex *lock, struct task_struct *task,
				struct rt_mutex_waiter *waiter)
{
  /*
	 * Before testing whether we can acquire @lock, we set the
	 * RT_MUTEX_HAS_WAITERS bit in @lock->owner. This forces all
	 * other tasks which try to modify @lock into the slow path
	 * and they serialize on @lock->wait_lock.
	 *
	 * The RT_MUTEX_HAS_WAITERS bit can have a transitional state
	 * as explained at the top of this file if and only if:
	 *
	 * - There is a lock owner. The caller must fixup the
	 *   transient state if it does a trylock or leaves the lock
	 *   function due to a signal or timeout.
	 *
	 * - @task acquires the lock and there are no other
	 *   waiters. This is undone in rt_mutex_set_owner(@task) at
	 *   the end of this function.
	 */
  /*
   * 此处很重要，具体可以参考 Documentation/locking/rt-mutex.txt
   */
	mark_rt_mutex_waiters(lock);

  if (rt_mutex_owner(lock)) //如果当前锁已经有了owner，获取失败
          return 0;

   /*
    * 如果有waiter，task已经讲waiter放到@lock waiter RB树里面，如果没有waiter，那么这是一次尝试获取lock的机会
    */
    /*
  	 * If @waiter != NULL, @task has already enqueued the waiter
  	 * into @lock waiter tree. If @waiter == NULL then this is a
  	 * trylock attempt.
  	 */    
   if (waiter) {
     /*
      * 如果waiter不是@lock树中的最高优先级，give up
      */
      /*
       * If waiter is not the highest priority waiter of @lock, give up.
       *
       */
      if (waiter != rt_mutex_top_waiter(lock))
          return 0;
     /*
      * 如果waiter是top priority，说明我们可以获取lock，讲waiter 出队。
      */
     /*
      * We can acquire the lock. Remove the waiter from the
      * lock waiters tree.
      */
      rt_mutex_dequeue(lock, waiter);
   } else {

     /*
 		 * If the lock has waiters already we check whether @task is
 		 * eligible to take over the lock.
 		 *
 		 * If there are no other waiters, @task can acquire
 		 * the lock.  @task->pi_blocked_on is NULL, so it does
 		 * not need to be dequeued.
 		 */
     if (rt_mutex_has_waiters(lock)) {

       /*
        * 如果lock没有owner，并且当前进程并不是top waiter。获取失败
        */
       if (task->prio >= rt_mutex_top_waiter(lock)->prio)
          return 0;

       /*
 			 * The current top waiter stays enqueued. We
 			 * don't have to change anything in the lock
 			 * waiters order.
 			 */
     } else {
       /*
 			 * No waiters. Take the lock without the
 			 * pi_lock dance.@task->pi_blocked_on is NULL
 			 * and we have no waiters to enqueue in @task
 			 * pi waiters tree.
 			 */
       /*
        * 如果没有waiter，
        */
        goto takeit;
   }

   /*
    * 下面就不具体分析了，已经暂时在纸上做了流程图，后续画图
    */


}

```

**task_blocks_on_rt_mutex**

这个函数是RTmutex思想的核心，即优先级继承。

```
/*
 * Task blocks on lock.
 *
 * Prepare waiter and propagate pi chain
 *
 * This must be called with lock->wait_lock held.
 */
/*
 * 这里强调下入参：
 * ret = task_blocks_on_rt_mutex(lock, &waiter, current, chwalk)
 */
static int task_blocks_on_rt_mutex(struct rt_mutex *lock,
				   struct rt_mutex_waiter *waiter,
				   struct task_struct *task,
				   enum rtmutex_chainwalk chwalk)
{
  /*
   * 如果当前进程已经是owner了，死锁!
   * 比如代码重复两次lock同一个锁。 rt_mutex_lock(&lock); rt_mutex_lock(&lock)；这样的代码
   */
  if (owner == task)
    return -EDEADLK;

>>>>>>>>>>>> P >>>>>>>>>>>>>

  raw_spin_lock_irqsave(&task->pi_lock, flags);
  __rt_mutex_adjust_prio(task);
	waiter->task = task;
	waiter->lock = lock;
	waiter->prio = task->prio;

  /* Get the top priority waiter on the lock */
	if (rt_mutex_has_waiters(lock))
		top_waiter = rt_mutex_top_waiter(lock);
	rt_mutex_enqueue(lock, waiter);

  /*
   * 将waiter插入到lock的RB树中
   */
	task->pi_blocked_on = waiter;    

>>>>>>>>>>>> P >>>>>>>>>>>>>

}

```
