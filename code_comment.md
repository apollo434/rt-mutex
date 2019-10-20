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

	if (likely(!ret))
		/* sleep on the mutex */
		ret = __rt_mutex_slowlock(lock, state, timeout, &waiter);

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
	fixup_rt_mutex_waiters(lock);

	raw_spin_unlock(&lock->wait_lock);

	/* Remove pending timer: */
	if (unlikely(timeout))
		hrtimer_cancel(&timeout->timer);

	debug_rt_mutex_free_waiter(&waiter);

	return ret;
}

```
