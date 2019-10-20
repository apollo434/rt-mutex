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
   * 如果当前Task已经是owner了，死锁
   */
	if (owner == task)
		return -EDEADLK;

  /*
   * 进入临界区，需要修改task的内容，调用task->pi_lock
   */
	raw_spin_lock_irqsave(&task->pi_lock, flags);
	__rt_mutex_adjust_prio(task);
	waiter->task = task;
	waiter->lock = lock;
	waiter->prio = task->prio;

	/* Get the top priority waiter on the lock */
	if (rt_mutex_has_waiters(lock))
		top_waiter = rt_mutex_top_waiter(lock);
	rt_mutex_enqueue(lock, waiter);

	task->pi_blocked_on = waiter;

	raw_spin_unlock_irqrestore(&task->pi_lock, flags);

	if (!owner)
		return 0;

	raw_spin_lock_irqsave(&owner->pi_lock, flags);
	if (waiter == rt_mutex_top_waiter(lock)) {
		rt_mutex_dequeue_pi(owner, top_waiter);
		rt_mutex_enqueue_pi(owner, waiter);

		__rt_mutex_adjust_prio(owner);
		if (owner->pi_blocked_on)
			chain_walk = 1;
	} else if (rt_mutex_cond_detect_deadlock(waiter, chwalk)) {
		chain_walk = 1;
	}

	/* Store the lock on which owner is blocked or NULL */
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
	get_task_struct(owner);

	raw_spin_unlock(&lock->wait_lock);

	res = rt_mutex_adjust_prio_chain(owner, chwalk, lock,
					 next_lock, waiter, task);

	raw_spin_lock(&lock->wait_lock);

	return res;
}

```
