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

  /*
   * 调整current的优先级
  */
  __rt_mutex_adjust_prio(task);
	waiter->task = task;
	waiter->lock = lock;
	waiter->prio = task->prio;

  /* Get the top priority waiter on the lock */
	if (rt_mutex_has_waiters(lock))
		top_waiter = rt_mutex_top_waiter(lock);
  /*
   * 将waiter插入到lock的RB树中
   */
	rt_mutex_enqueue(lock, waiter);

  /*
   * 设置当前进程被阻塞在waiter上，如果后面获取锁成功的话，这个pi_blocked_on会被重新置为NULL
   */
	task->pi_blocked_on = waiter;    

>>>>>>>>>>>> P >>>>>>>>>>>>>

  if (!owner)
    retuen 0;

>>>>>>>>>>>> no irq >>>>>>>>>>>>>
/*
 * 如果当前waiter成为lock的top waiter的话，调整owner的PI_waiter.因为进程的pi_waiters里面链入的是lock的top waiter
 */
if (waiter == rt_mutex_top_waiter(lock)) {
  rt_mutex_dequeue_pi(owner, top_waiter);
  rt_mutex_enqueue_pi(owner, waiter);

  /*
   * 调整owner的优先级，因为一个高优先级的waiter出现了。
   */
  __rt_mutex_adjust_prio(owner);

  /*
   * 如果owner同样被另外一把lock阻塞的话。那么需要遍历chain，调整优先级
   */
  if (owner->pi_blocked_on)
    chain_walk = 1;
} else if (rt_mutex_cond_detect_deadlock(waiter, chwalk)) {
  chain_walk = 1;
}

  /* Store the lock on which owner is blocked or NULL */
  /*
   * 获取阻塞owner的lock。如果next_lock存在的话，那么需要一步一步往下去调整整个pi chain
   */
  next_lock = task_blocked_on_lock(owner);
>>>>>>>>>>>> no irq >>>>>>>>>>>>>


  /*
  * Even if full deadlock detection is on, if the owner is not
  * blocked itself, we can avoid finding this out in the chain
  * walk.
  */
  if (!chain_walk || !next_lock)
    return 0;

    /*
    * The owner can't disappear while holding a lock,
    * so the owner struct is protected by wait_lock.
    * Gets dropped in rt_mutex_adjust_prio_chain()!
    */
    get_task_struct(owner);

>>>>>>>>>>>> P >>>>>>>>>>>>>

    /*
     * 遍历PI chain，调整优先级。关于pi chain，建议先看一下rt-mutex-design.txt大体了解一下。看了一百遍也没看透，shit，看代码继续缕吧。。。。。
     */
    res = rt_mutex_adjust_prio_chain(owner, chwalk, lock,
         next_lock, waiter, task);
}

```

### 最难懂的部分，看到吐血。。。 rt_mutex_adjust_prio_chain ####



```
保存下入参
res = rt_mutex_adjust_prio_chain(owner, chwalk, lock,
         next_lock, waiter, task);
/*
 * Adjust the priority chain. Also used for deadlock detection.
 * Decreases task's usage by one - may thus free the task.
 *
 * @task:	the task owning the mutex (owner) for which a chain walk is
 *		probably needed
 * @chwalk:	do we have to carry out deadlock detection?
 * @orig_lock:	the mutex (can be NULL if we are walking the chain to recheck
 *		things for a task that has just got its priority adjusted, and
 *		is waiting on a mutex)
 * @next_lock:	the mutex on which the owner of @orig_lock was blocked before
 *		we dropped its pi_lock. Is never dereferenced, only used for
 *		comparison to detect lock chain changes.
 * @orig_waiter: rt_mutex_waiter struct for the task that has just donated
 *		its priority to the mutex owner (can be NULL in the case
 *		depicted above or if the top waiter is gone away and we are
 *		actually deboosting the owner)
 * @top_task:	the current top waiter
 *
 * Returns 0 or -EDEADLK.
 *
 * Chain walk basics and protection scope
 *
 * [R] refcount on task
 * [P] task->pi_lock held
 * [L] rtmutex->wait_lock held
 *
 * Step	Description				Protected by
 *	function arguments:
 *	@task					[R]
 *	@orig_lock if != NULL			@top_task is blocked on it
 *	@next_lock				Unprotected. Cannot be
 *						dereferenced. Only used for
 *						comparison.
 *	@orig_waiter if != NULL			@top_task is blocked on it
 *	@top_task				current, or in case of proxy
 *						locking protected by calling
 *						code
 *	again:
 *	  loop_sanity_check();
 *	retry:
 * [1]	  lock(task->pi_lock);			[R] acquire [P]
 * [2]	  waiter = task->pi_blocked_on;		[P]
 * [3]	  check_exit_conditions_1();		[P]
 * [4]	  lock = waiter->lock;			[P]
 * [5]	  if (!try_lock(lock->wait_lock)) {	[P] try to acquire [L]
 *	    unlock(task->pi_lock);		release [P]
 *	    goto retry;
 *	  }
 * [6]	  check_exit_conditions_2();		[P] + [L]
 * [7]	  requeue_lock_waiter(lock, waiter);	[P] + [L]
 * [8]	  unlock(task->pi_lock);		release [P]
 *	  put_task_struct(task);		release [R]
 * [9]	  check_exit_conditions_3();		[L]
 * [10]	  task = owner(lock);			[L]
 *	  get_task_struct(task);		[L] acquire [R]
 *	  lock(task->pi_lock);			[L] acquire [P]
 * [11]	  requeue_pi_waiter(tsk, waiters(lock));[P] + [L]
 * [12]	  check_exit_conditions_4();		[P] + [L]
 * [13]	  unlock(task->pi_lock);		release [P]
 *	  unlock(lock->wait_lock);		release [L]
 *	  goto again;
 */
static int rt_mutex_adjust_prio_chain(struct task_struct *task,//orig_lock的owner
				      enum rtmutex_chainwalk chwalk,
				      struct rt_mutex *orig_lock,//当前进程尝试要获取的lock
				      struct rt_mutex *next_lock,//owner尝试要获取的lock
				      struct rt_mutex_waiter *orig_waiter,//当前进程的waiter
				      struct task_struct *top_task)//当前进程
{


}

```
