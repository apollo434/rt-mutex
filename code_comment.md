直接上中文吧，本来就是复杂，时间有限，就不拽英文了：）

**开始注释**

***rt_mute_lock***
```
/**
 * rt_mutex_lock - lock a rt_mutex
 *
 * @lock: the rt_mutex to be locked
 */
void __sched rt_mutex_lock(struct rt_mutex *lock)
{
	might_sleep();

  /* 快速路径一般需要硬件支持，同时owner == NULL，这里只看慢速路径 */
  /*
   * TASK_UNINTERRUPTIBLE 只能被wakeup唤醒
   * 慢速路径函数 rt_mutex_slowlock
   */
	rt_mutex_fastlock(lock, TASK_UNINTERRUPTIBLE, rt_mutex_slowlock);
}
EXPORT_SYMBOL_GPL(rt_mutex_lock);

```

***rt_mutex_slowlock***
```
/*
 * Slow path lock function:
 */
static int __sched
rt_mutex_slowlock(struct rt_mutex *lock, int state,
		  struct hrtimer_sleeper *timeout,
		  enum rtmutex_chainwalk chwalk)
{
  /*
   * 这里的waiter是局部变量，因为waiter只有在本进程成功获取rtmutex才有意义
   * 在https://www.kernel.org/doc/Documentation/locking/rt-mutex-design.txt 特意强调
   */
	struct rt_mutex_waiter waiter;
	int ret = 0;

  /*
   * 初始化waiter,同时初始化RB Tree
   */
	debug_rt_mutex_init_waiter(&waiter);
	RB_CLEAR_NODE(&waiter.pi_tree_entry);
	RB_CLEAR_NODE(&waiter.tree_entry);

  /*
   * 要对rtmutex进行操作，加自旋锁
   */
	raw_spin_lock(&lock->wait_lock);

	/* Try to acquire the lock again: */
  /*
   * 第一次尝试获取rtmutex，此函数在后面的__rt_mutex_slowlock中还会被调用
   */
	if (try_to_take_rt_mutex(lock, current, NULL)) {
		raw_spin_unlock(&lock->wait_lock);
		return 0;
	}

  /* 设置进程状态，之前是TASK_UNINTERRUPTIBLE */
	set_current_state(state);

	/* Setup the timer, when timeout != NULL */
  /* 如果有设置timeout的需求，则在当前进程设置，通过hrtimer完成,到期唤醒该进程 */
	if (unlikely(timeout))
		hrtimer_start_expires(&timeout->timer, HRTIMER_MODE_ABS);

  /*
   * 通过此函数来准备waiter，并调整PI Chain，该过程会多次检查，并尝试获取rtmutex lock。
   */
	ret = task_blocks_on_rt_mutex(lock, &waiter, current, chwalk);

  /*
   * 函数走到这边，waiter就绪（在lock 以及owner的pi_waiters RB树中）并且PI chain的优先级调整及死锁检查已经完毕。下面准备让当前进程sleep wait
   */
	if (likely(!ret))
		/* sleep on the mutex */
		ret = __rt_mutex_slowlock(lock, state, timeout, &waiter);

  /*
   * 如果返回值不为0的话，那么就是被信号打断或者timeout时间到了。不是被unlock唤醒的，所以需要在rt_mutex_lock中删除waiter.如果是被unlock唤醒的话，waiter的清理工作由unlock函数完成。
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
   * 因为之前设置了"Has waiter" bit，这里进行重置
   */
	fixup_rt_mutex_waiters(lock);

  /* 退临界区 */
	raw_spin_unlock(&lock->wait_lock);

	/* Remove pending timer: */
  /* 取消hrtimer定时器 */
	if (unlikely(timeout))
		hrtimer_cancel(&timeout->timer);

	debug_rt_mutex_free_waiter(&waiter);

	return ret;
}

```

***try_to_take_rt_mutex***

由于调用此函数的入参waiter为NULL,那么只有两种可能：

1）lock没有waiter，当前Task获取rtmutex lock，并返回1。

2）lock上有waiter，当前Task没有获取rtmutex lock，并返回0.

整个函数共分下面几种情况：
1. 如果当前lock有owner，获取失败
2. 当前lock没有owner

  1）但当前waiter不是top_waiter则获取失败。

  2）如果是top_waiter,则将waiter出队，准备获取lock
3. 当前lock没有owner，当前Task没有waiter，即task->pi_blocked_on is NULL。

  1）如果当前Task是top_waiter,获取lock。
  2）如果当前Task不是top_waiter,获取失败。

4. 当前lock没有owner，当前Task没有waiter，即task->pi_blocked_on is NULL。同时当前的lock还没有waiter，即无top_waiter,直接获取lock

```
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
	unsigned long flags;

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
   * 进来第一件事情，就是先将RT_MUTEX_HAS_WAITERS bit置1，这样，后续不会有其他Task进入快速路径，都强制进入慢速路径。
   * 1）如果没有获取lock，即lock确实有owner，通过调用fixup_rt_mutex_waiters(lock);来恢复。
   * 2）如果获取lock，则通过rt_mutex_set_owner(@task)设置owner
   */
	mark_rt_mutex_waiters(lock);

	/*
	 * If @lock has an owner, give up.
	 */
  /*
   * 如果当前lock有owner，获取失败
   */
	if (rt_mutex_owner(lock))
		return 0;

	/*
	 * If @waiter != NULL, @task has already enqueued the waiter
	 * into @lock waiter tree. If @waiter == NULL then this is a
	 * trylock attempt.
	 */
  /*
   * 如果有waiter
   * 1）但当前waiter不是top_waiter则获取失败
   * 2）如果是top_waiter,则将waiter出队，准备获取lock，后续会调用task->pi_blocked_on = NULL;设置当前Task没有被阻塞，
   * 同时调用rt_mutex_enqueue_pi来设置pi_waiter RB Tree,和rt_mutex_set_owner(lock, task)设置lock的owner
   */
	if (waiter) {
		/*
		 * If waiter is not the highest priority waiter of
		 * @lock, give up.
		 */
		if (waiter != rt_mutex_top_waiter(lock))
			return 0;

		/*
		 * We can acquire the lock. Remove the waiter from the
		 * lock waiters tree.
		 */
		rt_mutex_dequeue(lock, waiter);

	} else {
    /*
     * 如果waiter == NULL，即在rt_mutex_slowlock调用时传参
     * 当前lock没有owner（前面if (rt_mutex_owner(lock))已经判断）并且当前Task不是top_waiter,通过task->prio >= top_waiter->prio，来判断，prio越小，优先级越高，则当前
     * Task不是top_waiter，无法获取lock
     * bin
     */
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
			 * If @task->prio is greater than or equal to
			 * the top waiter priority (kernel view),
			 * @task lost.
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
       * 即没有waiter，又没有owner，而当前Task的waiter也是NULL，即task->pi_blocked_on is NULL。
       * 那还想什么，开心的获取lock吧
       */
			goto takeit;
		}
	}

	/*
	 * Clear @task->pi_blocked_on. Requires protection by
	 * @task->pi_lock. Redundant operation for the @waiter == NULL
	 * case, but conditionals are more expensive than a redundant
	 * store.
	 */
	raw_spin_lock_irqsave(&task->pi_lock, flags);
	task->pi_blocked_on = NULL;
	/*
	 * Finish the lock acquisition. @task is the new owner. If
	 * other waiters exist we have to insert the highest priority
	 * waiter into @task->pi_waiters tree.
	 */
	if (rt_mutex_has_waiters(lock))
		rt_mutex_enqueue_pi(task, rt_mutex_top_waiter(lock));
	raw_spin_unlock_irqrestore(&task->pi_lock, flags);

takeit:
	/* We got the lock. */
	debug_rt_mutex_lock(lock);

	/*
	 * This either preserves the RT_MUTEX_HAS_WAITERS bit if there
	 * are still waiters or clears it.
	 */
	rt_mutex_set_owner(lock, task);

	rt_mutex_deadlock_account_lock(lock, task);

	return 1;
}

```

***task_blocks_on_rt_mutex***

这个函数是rtmutex的核心思想所在，包含几个核心函数：

即PI Chain的实现：rt_mutex_adjust_prio_chain 优先级继承

本函数主要作用：

1）插入waiter链。

2）决定是否进行PI Chain


****本函数的大体操作过程:****

----

1. 修改当前Task的信息：

1）调整task->prio, 需要boost，从top_waiter中获取。

2）初始化Task对应的waiter->task/lock/prio.

3) 将Task对应的waiter，入要获取的lock的RB Tree.

4) 将Task：task->pi_blocked_on = waiter

2. 如果要获取的lock的owner为空，则直接获取当前lock。

3. 如果lock的owner不为空，调整owner的信息：

3.1）如果当前waiter成为lock的top waiter的话：

3.1.1）调整owner的PI_waiter： owner->waiter

3.1.2）调整owner的prio：owner->waiters_leftmost->prio

3.1.3) 判断在对owner操作的过程中，是否当前owner被新的Task给block住，owner->pi_blocked_on == NULL or not?

3.1.3.1) 如果被block住了，则可能需要进行PI Chain.

3.1.3.2) 获取阻塞owner的lock，即next_lock。

3.1.3.3）如果3.1.3.1 & 3.1.3.2 同时成立，则正式进行PI Chain.

4. 调用rt_mutex_adjust_prio_chain，进行PI Chain

```
/*
 * Task blocks on lock.
 *
 * Prepare waiter and propagate pi chain
 *
 * This must be called with lock->wait_lock held.
 */
static int task_blocks_on_rt_mutex(struct rt_mutex *lock,
				   struct rt_mutex_waiter *waiter,
				   struct task_struct *task,
				   enum rtmutex_chainwalk chwalk)
{
	struct task_struct *owner = rt_mutex_owner(lock);
  /*
   * 此处waiter一直是局部变量，牢记！
   */
	struct rt_mutex_waiter *top_waiter = waiter;
	struct rt_mutex *next_lock;
	int chain_walk = 0, res;
	unsigned long flags;

	/*
	 * Early deadlock detection. We really don't want the task to
	 * enqueue on itself just to untangle the mess later. It's not
	 * only an optimization. We drop the locks, so another waiter
	 * can come in before the chain walk detects the deadlock. So
	 * the other will detect the deadlock and return -EDEADLOCK,
	 * which is wrong, as the other waiter is not in a deadlock
	 * situation.
	 */
  /*
   * 如果当前Task已经是owner了，死锁,比如：重复调用rt_mutex_lock
   */
	if (owner == task)
		return -EDEADLK;

  /*
   * 进入临界区，需要修改task的内容，调用task->pi_lock
   */
	raw_spin_lock_irqsave(&task->pi_lock, flags);

  /*
   * Boost当前Task的prio，并将本Task对应的waiter进行初始化
   */
	__rt_mutex_adjust_prio(task);
	waiter->task = task;
	waiter->lock = lock;
	waiter->prio = task->prio;

  /*
   * 如果要获取的lock有waiter，则获取top_waiter(暂存,后面用),然后讲当前Task的waiter入lock的waiter队
   */
	/* Get the top priority waiter on the lock */
	if (rt_mutex_has_waiters(lock))
		top_waiter = rt_mutex_top_waiter(lock);
	rt_mutex_enqueue(lock, waiter);

  /*
   * 当前Task被对应的waiter block住了，所以记录task->pi_blocked_on。
   * 如果后面获取lock成功的话，这个pi_blocked_on会被重新设置为NULL
   */
	task->pi_blocked_on = waiter;

	raw_spin_unlock_irqrestore(&task->pi_lock, flags);

  /*
   * 如果要获取的lock的owner为空，则获取lock
   */
	if (!owner)
		return 0;

  /*
   * 进入当前owner的临界区，调用owner->pi_lock
   */
	raw_spin_lock_irqsave(&owner->pi_lock, flags);
  /*
   * 如果当前waiter成为lock的top waiter的话，调整owner的PI_waiter.
   * 因为进程的pi_waiters里面链入的是lock的top waiter
   */
	if (waiter == rt_mutex_top_waiter(lock)) {
		rt_mutex_dequeue_pi(owner, top_waiter);
		rt_mutex_enqueue_pi(owner, waiter);

   /*
    * 因为top_waiter更改，所以，需要调整owner的prio
    */
		__rt_mutex_adjust_prio(owner);

    /*
     * 如果在进行对waiter插入各个RB Tree的过程中，要获取lock
     * 的owner被block了，这个有意思了，需要调用那个PI Chain了，
     * 因为此刻有更高优先级的Task来了。
     */
		if (owner->pi_blocked_on)
			chain_walk = 1;
	} else if (rt_mutex_cond_detect_deadlock(waiter, chwalk)) {
    /*
     * 当前waiter不是top_waiter，那么做一次死锁检查
     */
		chain_walk = 1;
	}

	/* Store the lock on which owner is blocked or NULL */
  /*
   * 获取阻塞owner的lock。如果next_lock存在的话，
   * 那么需要一步一步往下去调整整个pi chain
   */
	next_lock = task_blocked_on_lock(owner);

	raw_spin_unlock_irqrestore(&owner->pi_lock, flags);
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
  /*
   * 为什么要get呢？因为owner是局部变量
   */
	get_task_struct(owner);

	raw_spin_unlock(&lock->wait_lock);

  /*
   * 啥也不说了，看到快吐血的PI Chain函数来了，切记！切记！切记！
   * 重要的事情说三句，这里的入参有两个rt_mutex lock 和 两个 task_struct
   * 一定要把这些参数放到一个显眼的地方看着，要是你不晕，我竟你是条汉子！！！！！！！
   */
	res = rt_mutex_adjust_prio_chain(owner, chwalk, lock,
					 next_lock, waiter, task);

	raw_spin_lock(&lock->wait_lock);

	return res;
}

```

***rt_mutex_adjust_prio_chain***

根据之前的记录，这部分函数分成四个部分，分别进行分析：

1. 判断是否还需要PI Chain.

2. 死锁检测

3. 调整rtmutex的lock对应的RB Tree等相关信息.

4. 调整Task对应的RB Tree等相关信息.

**NOTE1**

对PI Chain进行walk时，就是遍历PI Chain来调整优先级，直到某个owner并没有被阻塞为止

**NOTE2**

为什么要不阻塞为止？因为向上遍历，在此再次给出一个chain的例子：

```

Example:

   Process:  A, B, C, D, E
   Mutexes:  L1, L2, L3, L4

   A owns: L1
           B blocked on L1
           B owns L2
                  C blocked on L2
                  C owns L3
                         D blocked on L3
                         D owns L4
                                E blocked on L4

The chain would be:

   E->L4->D->L3->C->L2->B->L1->A
```

**NOTE3**

对于PI Chain的walk可能是从底开始，也可能是从中间开始

**第一部分：判断是否还需要PI Chain**
```
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
static int rt_mutex_adjust_prio_chain(struct task_struct *task,
				      enum rtmutex_chainwalk chwalk,
				      struct rt_mutex *orig_lock,
				      struct rt_mutex *next_lock,
				      struct rt_mutex_waiter *orig_waiter,
				      struct task_struct *top_task)
{
	struct rt_mutex_waiter *waiter, *top_waiter = orig_waiter;
	struct rt_mutex_waiter *prerequeue_top_waiter;
	int ret = 0, depth = 0;
	struct rt_mutex *lock;
	bool detect_deadlock;
	unsigned long flags;
	bool requeue = true;

	detect_deadlock = rt_mutex_cond_detect_deadlock(orig_waiter, chwalk);

	/*
	 * The (de)boosting is a step by step approach with a lot of
	 * pitfalls. We want this to be preemptible and we want hold a
	 * maximum of two locks per step. So we have to check
	 * carefully whether things change under us.
	 */
 again:
	/*
	 * We limit the lock chain length for each invocation.
	 */
  /*
   * 该函数会遍历PI Chain，但也不是无限制的遍历，对于遍历的深度是有限制的，
   * 为1024 (int max_lock_depth = 1024;）
   */
	if (++depth > max_lock_depth) {
		static int prev_max;

    /*
     * 如果超过限制则为死锁
     */
		/*
		 * Print this only once. If the admin changes the limit,
		 * print a new message when reaching the limit again.
		 */
		if (prev_max != max_lock_depth) {
			prev_max = max_lock_depth;
			printk(KERN_WARNING "Maximum lock depth %d reached "
			       "task: %s (%d)\n", max_lock_depth,
			       top_task->comm, task_pid_nr(top_task));
		}

    /*
     * 这里为什么要put task呢？因为前一个函数里面是get的局部变量，需要释放
     */
		put_task_struct(task);

		return -EDEADLK;
	}

  /*
   * OK，从这里开始就是函数的第一部分，各种判断是否需要进行PI Chain的判断
   */
	/*
	 * We are fully preemptible here and only hold the refcount on
	 * @task. So everything can have changed under us since the
	 * caller or our own code below (goto retry/again) dropped all
	 * locks.
	 */
 retry:
	/*
	 * [1] Task cannot go away as we did a get_task() before !
	 */
	raw_spin_lock_irqsave(&task->pi_lock, flags);

  /*
   * 获取Task对应的waiter，注意这个Task是orig_waiter的owner,
   * 为了明确我把上一个函数中对本函数调用时的入参放到这里：
   * rt_mutex_adjust_prio_chain(owner, chwalk, lock,next_lock, waiter, task);
   * 再次做一个对应：
   * 本函数入参 ============> 对应上一个函数的内容
   * Task ============> owner（current要获取的lock的owner）
   * orig_lock ============> lock (current要获取的lock)
   * next_lock ============> 阻塞owner的lock
   * orig_waiter ============> waiter(current对应的waiter)
   * top_task ============> task(即current)
   */
  /*
   * 下面是本函数出现的局部变量：
   * 本函数局部变量 ============> 对应的内容
   * top_waiter ============> current对应的waiter
   * waiter ============> current要获取的lock的owner，阻塞它的waiter
   * lock ============> current要获取的lock的owner中的waiter记录的阻塞这个owner的lock
   */
  	/*
	 * [2] Get the waiter on which @task is blocked on.
	 */

	waiter = task->pi_blocked_on;

	/*
	 * [3] check_exit_conditions_1() protected by task->pi_lock.
	 */

	/*
	 * Check whether the end of the boosting chain has been
	 * reached or the state of the chain has changed while we
	 * dropped the locks.
	 */
   /*
    * 如果current要获取的lock的owner中的waiter不存在，
    * 则说明owner已经是top，没有其他Task阻塞它，不需PI Chain
    */
	if (!waiter)
		goto out_unlock_pi;

	/*
	 * Check the orig_waiter state. After we dropped the locks,
	 * the previous owner of the lock might have released the lock.
	 */
  /*
   * 如果current对应的waiter存在，同时current要获取的lock的owner却不存在，即当前
   * 的current不在被owner所阻塞，所以，无需PI Chain，而是要尝试获取lock了
   */
	if (orig_waiter && !rt_mutex_owner(orig_lock))
		goto out_unlock_pi;

	/*
	 * We dropped all locks after taking a refcount on @task, so
	 * the task might have moved on in the lock chain or even left
	 * the chain completely and blocks now on an unrelated lock or
	 * on @orig_lock.
	 *
	 * We stored the lock on which @task was blocked in @next_lock,
	 * so we can detect the chain change.
	 */
  /*
   * 如果阻塞owner的lock 不等于 current要获取的lock的owner中的waiter的lock，
   * 本来两个lock是一个，却不想等，很可能此时owner不再被阻塞，或者，阻塞它的锁从
   * 一个锁变成另一个锁，那么就没必要继续进行PI Chain了
   */
	if (next_lock != waiter->lock)
		goto out_unlock_pi;

	/*
	 * Drop out, when the task has no waiters. Note,
	 * top_waiter can be NULL, when we are in the deboosting
	 * mode!
	 */
	if (top_waiter) {
    /*
     * 如果 current对应的waiter依然存在，但current要获取的lock的owner的
     * pi waiter却不存在，这就意味着owner没有RB Tree了，那么owner不再
     * 阻塞current，无需进行PI Chain
     */
		if (!task_has_pi_waiters(task))
			goto out_unlock_pi;
		/*
		 * If deadlock detection is off, we stop here if we
		 * are not the top pi waiter of the task. If deadlock
		 * detection is enabled we continue, but stop the
		 * requeueing in the chain walk.
		 */
		if (top_waiter != task_top_pi_waiter(task)) {
			if (!detect_deadlock)
      /*
       * 如果top_waiter即current对应的waiter 不等于 current要
       * 获取的lock的owner的top waiter，即最left的waiter，如果
       * 开启了死锁检测，则记录下requeue = falese，如果没有开启，
       * 则无需进行PI Chain，直接返回
       */
				goto out_unlock_pi;
			else
				requeue = false;
		}
	}

	/*
	 * If the waiter priority is the same as the task priority
	 * then there is no further priority adjustment necessary.  If
	 * deadlock detection is off, we stop the chain walk. If its
	 * enabled we continue, but stop the requeueing in the chain
	 * walk.
	 */
  /*
   * 当 （current要获取的lock的owner，阻塞它的waiter）对应的prio
   * 与current要获取的lock的owner的prio相等，那么说明在没有开启死
   * 锁检测的情况下，是无需进行PI Chain，因为优先级没变化，如果开启
   * 了死锁检测则requeue = false
   */
	if (waiter->prio == task->prio) {
		if (!detect_deadlock)
			goto out_unlock_pi;
		else
			requeue = false;
	}

	/*
	 * [4] Get the next lock
	 */
	lock = waiter->lock;
	/*
	 * [5] We need to trylock here as we are holding task->pi_lock,
	 * which is the reverse lock order versus the other rtmutex
	 * operations.
	 */
	if (!raw_spin_trylock(&lock->wait_lock)) {
		raw_spin_unlock_irqrestore(&task->pi_lock, flags);
		cpu_relax();
		goto retry;
	}

```


**第二部分：这部分是死锁检查**

```
	/*
	 * [6] check_exit_conditions_2() protected by task->pi_lock and
	 * lock->wait_lock.
	 *
	 * Deadlock detection. If the lock is the same as the original
	 * lock which caused us to walk the lock chain or if the
	 * current lock is owned by the task which initiated the chain
	 * walk, we detected a deadlock.
	 */
	if (lock == orig_lock || rt_mutex_owner(lock) == top_task) {
		debug_rt_mutex_deadlock(chwalk, orig_waiter, lock);
		raw_spin_unlock(&lock->wait_lock);
		ret = -EDEADLK;
		goto out_unlock_pi;
	}


```

**第二部分后续：死锁检查善后工作**

```
	/*
	 * If we just follow the lock chain for deadlock detection, no
	 * need to do all the requeue operations. To avoid a truckload
	 * of conditionals around the various places below, just do the
	 * minimum chain walk checks.
	 */
  /*
   * 如果我们只是按照lock chain进行死锁检测，就不需要执行所有的重新排队操作。
   * 为了避免在下面的不同地方出现一卡车的条件语句，只需要做最少的链式检查
   */
	if (!requeue) {
		/*
		 * No requeue[7] here. Just release @task [8]
		 */
		raw_spin_unlock_irqrestore(&task->pi_lock, flags);
		put_task_struct(task);

    /*
     * 如果current要获取的lock的owner对应的waiter记录的阻塞这个
     * owner的lock的owner，是不存在的，则已经到了PI Chain的结尾，结束，打完收工
     */
		/*
		 * [9] check_exit_conditions_3 protected by lock->wait_lock.
		 * If there is no owner of the lock, end of chain.
		 */
		if (!rt_mutex_owner(lock)) {
			raw_spin_unlock(&lock->wait_lock);
			return 0;
		}

    /*
     * 如果存在current要获取的lock的owner对应的waiter记录的
     * 阻塞这个owner的lock的owner，那么获取这个owner，并进行下一步PI Chain操作
     */
		/* [10] Grab the next task, i.e. owner of @lock */
		task = rt_mutex_owner(lock);
		get_task_struct(task);
		raw_spin_lock_irqsave(&task->pi_lock, flags);

    /*
     * 获取current要获取的lock的owner对应的waiter记录的阻塞
     * 这个owner的lock的owner中的阻塞它的lock（唉。。。生无可恋）
     */
		/*
		 * No requeue [11] here. We just do deadlock detection.
		 *
		 * [12] Store whether owner is blocked
		 * itself. Decision is made after dropping the locks
		 */
		next_lock = task_blocked_on_lock(task);

    /*
     * 获取current要获取的lock的owner对应的waiter记录的阻塞这个
     * owner的lock中的top_waiter.
     */
		/*
		 * Get the top waiter for the next iteration
		 */
		top_waiter = rt_mutex_top_waiter(lock);

		/* [13] Drop locks */
		raw_spin_unlock_irqrestore(&task->pi_lock, flags);
		raw_spin_unlock(&lock->wait_lock);

    /*
     * 如果next_lock == NULL,那么就已经到PI Chain的尾部了，
     * 直接释放刚刚的task局部变量，然后返回
     * 如果next_lock 不为NULL，那么继续来一遍PI Chain
     */
		/* If owner is not blocked, end of chain. */
		if (!next_lock)
			goto out_put_task;
		goto again;
	}

```

**第三部分：调整rtmutex的lock对应的RB Tree等相关信息**

```
  /*
   * 获取current要获取的lock的owner对应的waiter记录的
   * 阻塞这个owner的lock中的top_waiter，起个名字叫prerequeue_top_waiter
   * 这个top_waiter在这里只是用来预存的，后面会用到
   */
	/*
	 * Store the current top waiter before doing the requeue
	 * operation on @lock. We need it for the boost/deboost
	 * decision below.
	 */
	prerequeue_top_waiter = rt_mutex_top_waiter(lock);

  /*
   * 走到这里，说明已经存在一个current要获取的lock的owner对应
   * 的waiter记录的阻塞这个owner的lock存在，并且prio还不一样
   * （翻看前面的第一部分的判断），那么需要对rtmutex lock的RB Tree进行更新。
   */
	/* [7] Requeue the waiter in the lock waiter tree. */
	rt_mutex_dequeue(lock, waiter);
	waiter->prio = task->prio;
	rt_mutex_enqueue(lock, waiter);

	/* [8] Release the task */
	raw_spin_unlock_irqrestore(&task->pi_lock, flags);
	put_task_struct(task);

  /*
   * current要获取的lock的owner中的waiter记录的阻塞这个owner
   * 的lock的owner，这个时候不存在了，那么说明这个lock已经释放了，
   * 那么唤醒这个lock的top waiter
   */
	/*
	 * [9] check_exit_conditions_3 protected by lock->wait_lock.
	 *
	 * We must abort the chain walk if there is no lock owner even
	 * in the dead lock detection case, as we have nothing to
	 * follow here. This is the end of the chain we are walking.
	 */
	if (!rt_mutex_owner(lock)) {
		/*
		 * If the requeue [7] above changed the top waiter,
		 * then we need to wake the new top waiter up to try
		 * to get the lock.
		 */
		if (prerequeue_top_waiter != rt_mutex_top_waiter(lock))
			wake_up_process(rt_mutex_top_waiter(lock)->task);
		raw_spin_unlock(&lock->wait_lock);
		return 0;
	}


```

**第四部分：调整Task对应的RB Tree等相关信息**

```
  /*
   * 前面更新了rtmutex lock的RB Tree等信息，那么下面是更新Task的信息，
   * 获取current要获取的lock的owner中的waiter记录的阻塞这个owner的lock的owner
   */
	/* [10] Grab the next task, i.e. the owner of @lock */
	task = rt_mutex_owner(lock);
	get_task_struct(task);
	raw_spin_lock_irqsave(&task->pi_lock, flags);

  /*
   * 如果（current要获取的lock的owner中的waiter记录的阻塞这个owner的
   * lock的top waiter） == （current要获取的lock的owner，阻塞它的waiter），
   * 那么说明当前的waiter才是真正的top waiter，需要更新Task里面的RB Tree，
   * 将之前的prerequeue_top_waiter移除，将这个waiter移入，这个属于Boost
   * the owner，优先级提高了
   */
	/* [11] requeue the pi waiters if necessary */
	if (waiter == rt_mutex_top_waiter(lock)) {
		/*
		 * The waiter became the new top (highest priority)
		 * waiter on the lock. Replace the previous top waiter
		 * in the owner tasks pi waiters tree with this waiter
		 * and adjust the priority of the owner.
		 */
		rt_mutex_dequeue_pi(task, prerequeue_top_waiter);
		rt_mutex_enqueue_pi(task, waiter);
		__rt_mutex_adjust_prio(task);

    /*
     * 如果（current要获取的lock的owner，阻塞它的waiter） ==
     * （current要获取的lock的owner对应的waiter记录的阻塞这个owner
     * 的lock中的top_waiter）.
     * 说明什么呢？说明该current要获取的lock的owner，阻塞它的waiter，
     * 它本来就是top waiter，但很巧，这个top waiter获得了阻塞它的lock，
     * 那么需就需要重新调整优先级，这个属于Deboost the owner，优先级降低了
     */
	} else if (prerequeue_top_waiter == waiter) {
		/*
		 * The waiter was the top waiter on the lock, but is
		 * no longer the top prority waiter. Replace waiter in
		 * the owner tasks pi waiters tree with the new top
		 * (highest priority) waiter and adjust the priority
		 * of the owner.
		 * The new top waiter is stored in @waiter so that
		 * @waiter == @top_waiter evaluates to true below and
		 * we continue to deboost the rest of the chain.
		 */
		rt_mutex_dequeue_pi(task, waiter);
		waiter = rt_mutex_top_waiter(lock);
		rt_mutex_enqueue_pi(task, waiter);
		__rt_mutex_adjust_prio(task);
	} else {
		/*
		 * Nothing changed. No need to do any priority
		 * adjustment.
		 */
	}

  /*
   * 获取next_lock， 这个lock是什么呢？是阻塞current要获取的lock的
   * owner中的waiter记录的阻塞这个owner的lock的owner <==== 阻塞这个owner的lock
   */
	/*
	 * [12] check_exit_conditions_4() protected by task->pi_lock
	 * and lock->wait_lock. The actual decisions are made after we
	 * dropped the locks.
	 *
	 * Check whether the task which owns the current lock is pi
	 * blocked itself. If yes we store a pointer to the lock for
	 * the lock chain change detection above. After we dropped
	 * task->pi_lock next_lock cannot be dereferenced anymore.
	 */
	next_lock = task_blocked_on_lock(task);

  /*
   * 获取current要获取的lock的owner中的waiter记录的阻塞这个owner的
   * lock中的top waiter
   */
	/*
	 * Store the top waiter of @lock for the end of chain walk
	 * decision below.
	 */
	top_waiter = rt_mutex_top_waiter(lock);

	/* [13] Drop the locks */
	raw_spin_unlock_irqrestore(&task->pi_lock, flags);
	raw_spin_unlock(&lock->wait_lock);

  /*
   * 下面进行下一次PI Chain的walk
   */
	/*
	 * Make the actual exit decisions [12], based on the stored
	 * values.
	 *
	 * We reached the end of the lock chain. Stop right here. No
	 * point to go back just to figure that out.
	 */
	if (!next_lock)
		goto out_put_task;

	/*
	 * If the current waiter is not the top waiter on the lock,
	 * then we can stop the chain walk here if we are not in full
	 * deadlock detection mode.
	 */
	if (!detect_deadlock && waiter != top_waiter)
		goto out_put_task;

	goto again;

 out_unlock_pi:
	raw_spin_unlock_irqrestore(&task->pi_lock, flags);
 out_put_task:
	put_task_struct(task);

	return ret;
}

```

**__rt_mutex_slowlock**

如同注释中说的那样，本函数就是完成:wait-wake-try-to-take loop

Perform the wait-wake-try-to-take loop

```
/**
 * __rt_mutex_slowlock() - Perform the wait-wake-try-to-take loop
 * @lock:		 the rt_mutex to take
 * @state:		 the state the task should block in (TASK_INTERRUPTIBLE
 * 			 or TASK_UNINTERRUPTIBLE)
 * @timeout:		 the pre-initialized and started timer, or NULL for none
 * @waiter:		 the pre-initialized rt_mutex_waiter
 *
 * lock->wait_lock must be held by the caller.
 */
static int __sched
__rt_mutex_slowlock(struct rt_mutex *lock, int state,
		    struct hrtimer_sleeper *timeout,
		    struct rt_mutex_waiter *waiter)
{
	int ret = 0;


  /*
   * 当代码进到这里后，说明task_blocks_on_rt_mutex执行完后并没有获得锁
   */
  /*
   * 这里会再次调用try_to_take_rt_mutex,但这次waiter不是NULL
   */
	for (;;) {
		/* Try to acquire the lock: */
		if (try_to_take_rt_mutex(lock, current, waiter))
			break;

    /*
     * 进程是TASK_INTERRUPTIBLE状态的话，即代表可以倍信号打断，
     * 这里会检查是否有为处理的信号，如果设置了hrtime，则查看是否超时
     */
		/*
		 * TASK_INTERRUPTIBLE checks for signals and
		 * timeout. Ignored otherwise.
		 */
		if (unlikely(state == TASK_INTERRUPTIBLE)) {

			/* Signal pending? */
			if (signal_pending(current))
				ret = -EINTR;
			if (timeout && !timeout->task)
				ret = -ETIMEDOUT;
			if (ret)
				break;
		}

		raw_spin_unlock(&lock->wait_lock);

		debug_rt_mutex_print_deadlock(waiter);

    /*
     * schedule 反复进入循环体，知道获取lock才break
     */
		schedule();

		raw_spin_lock(&lock->wait_lock);
		set_current_state(state);
	}

	__set_current_state(TASK_RUNNING);
	return ret;
}

```
