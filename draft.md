
```
update_process_times
  rcu_check_callbacks
    rcu_pending
      __rcu_pending
        check_cpu_stall
          print_cpu_stall

```


```
005 linux2.6.25.4-rt/kernel/rtmutex.c
 分类： LINUX2008-10-06 14:22:32
2008-6-16   

 一个低优先级的rtmutex owner会继承高优先级的waiter的优先级, 谓之PI.如果此进程还block到其他的rtmutex,这个优先级还要
传播到这个rtmutx的owner.


struct rt_mutex {
	raw_spinlock_t		wait_lock;
	struct plist_head	wait_list; /*按优先级排序,同优先级采用FIFO order*/
	struct task_struct	*owner;  /*owner以及状态,见下表*/
#ifdef CONFIG_DEBUG_RT_MUTEXES
	int			save_state;
	const char 		*name, *file;
	int			line;
	void			*magic;
#endif
};

owner状态:
 owner		bit1	bit0
 NULL		0	0	mutex is free (fast acquire possible)
 NULL		0	1	invalid state
 NULL		1	0	Transitional state*
 NULL		1	1	invalid state
 taskpointer	0	0	mutex is held (fast release possible)
 taskpointer	0	1	task is pending owner
 taskpointer	1	0	mutex is held and has waiters
 taskpointer	1	1	task is pending owner and mutex has waiters


当rt_mutex设置owner后, 不一定代表立即拥有此lock,有可能被steal掉,见try_to_steal_lock和对应的wakeup_next_waiter
(此函数唤醒一个进程时,设置最可能的owenr,并mark成pending状态,代表还没有最后获取到lock).参考文档就是rt-mutex.txt:
    当mutex释放时,owenrship指定给当前最高优先级的任务,并唤醒他,然后他开始运行,然后他可以清除pending bit正式获取到这个
mutex,但有高优先级的任务竞争时,mutex会被搞优先级的任务steal. 就是try_to_steal_lock干的事情. steal在下面的情况下很
有效果:一个高优先级的任务有个low优先级的watier,如果没有steal,当高优先级任务释放的时候会吧owner指定给低优先级的任务,这
样就推迟了一次可以马上抢占的机会.

Transitional state*:
(rtmutex.txt中对此解释有点老,看代码里的解释,比较新,rtmutex.c的开头和函数 do_try_to_take_rt_mutex )
***: RT_MUTEX_HAS_WAITERS 本意是说有waiter,不能做fast release(见rt_spin_lock_fastunlock, cmpxchg会失败从而
走入slow path). 这里的关键是:
    owner在进行fast unlock时不持有lock->wait_lock, 这样当我们do_try_to_take_rt_mutex在进行try_to_steal_lock
等一系列检查时,当前的owner可能已经释放了这个锁(还可能被被人抢掉),所以do_try_to_take_rt_mutex先设置上have watier然后
再去做一些列的工作, 这样slow path不通只能走slow path, slow path的锁(lock->wait_lock)已经被do_try_to_take_rt_
mutex 获取了,这就避免了不同步的现象.
    过渡状态就是这样出现的: 如果根本没有竞争(我们锁lock->wait_lock的时候,原来的owner释放了这个锁),就出现了过渡状态.


rt-mutex-design.txt:
------------------------------------------------------------
top waiter - The highest priority process waiting on a specific mutex.
                mutex的waiter队列中最高优先级的进程.

top pi waiter-The highest priority process waiting on one of the mutexes that a specific process owns.
               一个进程拥有的所有mutexes的watier中最高优先级的进程.

PI chain:
------------------------------------------------------------
       A owns: L1
           B blocked on L1
           B owns L2
                C block on L2
表示为:
      C->L2->B->L1->A

  一个进程不能同时block到多个mutex,但是可以拥有多个mutexes,所以这个链表只会merge,不会分叉.复杂点的例子如下:
 E->L4->D->L3->C-+
                      +->L2-+
                      |       |
                   G-+       +->B->L1->A
                              |
                     F->L5-+
链表最右边的, 或称之为top, 其优先级应该等于或高于其左边的任意进程, 就是PI.
--------------------------------------------------------------------------
Plist: 按照优先级排序的链表, 链表头是list head, 节点是list node.
--------------------------------------------------------------------------


Mutex Waiter List
-------------------------

struct rt_mutex {
	raw_spinlock_t		wait_lock;  /*无中断竞争*/
	struct plist_head	wait_list; /*按优先级排序,同优先级采用FIFO order,Mutex Waiter List*/
	struct task_struct	*owner;  /*owner以及状态,见下表*/
#ifdef CONFIG_DEBUG_RT_MUTEXES
	int			save_state;
	const char 		*name, *file;
	int			line;
	void			*magic;
#endif
};


Task PI List
------------
struct task_struct {
..........
	/* Protection of the PI data structures: */
	raw_spinlock_t pi_lock;  /*可能从中断访问, 需要spin_lock_irq*/

#ifdef CONFIG_RT_MUTEXES
	/* PI waiters blocked on a rt_mutex held by this task */
	struct plist_head pi_waiters;/*进程拥有的每个mutex的最高优先级的任务在此队列,此任务的PI优先级应等于TOP*/
	/* Deadlock detection and priority inheritance handling */
	struct rt_mutex_waiter *pi_blocked_on; /*只能block到一个mutex上*/
#endif

#ifdef CONFIG_DEBUG_MUTEXES
	/* mutex deadlock detection */
	struct mutex_waiter *blocked_on;
#endif
.......
}



Depth of the PI Chain
-------------------------
一个PI chain中的进程数目

Mutex owner and flags
---------------------
上面有了详细讨论了已经.


cmpxchg Tricks
-----------------
著名的lock free 基本元素之一.


Priority adjustments
--------------------
pi_waiters 的第一个元素之进程的优先级就是mutex所应该有的优先级, rt_mutex_adjust_prio 考虑了更多的情况取下面各个优先
级的最小值(优先级最高):
  p->normal_prio, get_rcu_prio(task),rt_mutex_get_readers_prio(task, prio),task_top_pi_waiter


High level overview of the PI chain walk
----------------------------------------
rt_mutex_adjust_prio_chain : 当一个进程block/leave一个mutex, 或者进程被修改优先级后,需要用此函数把优先级改变传播
到整个PI chain.(task_blocks_on_rt_mutex,remove_waiter,rt_mutex_adjust_pi),rt_mutex_adjust_prio 已经
执行.

为方便读此函数,先看看数据结构的一张简图, 这个图显式A->L1->B这个PI chain如何通过waiter这个结构组织起来



以一个任务block到一个mutex为例,读下代码:
我们设想一个场景,如下图,图中标识了各个变量的位置,使代码阅读起来非常的容易:


task_blocks_on_rt_mutex(struct rt_mutex *lock, struct rt_mutex_waiter *waiter,  int detect_deadlock,
 unsigned long flags)
{
	struct task_struct *owner = rt_mutex_owner(lock);
	struct rt_mutex_waiter *top_waiter = waiter;
	int chain_walk = 0, res;

	spin_lock(¤t->pi_lock);
	__rt_mutex_adjust_prio(current);
        /*初始化current进程关联的waiter*/
	waiter->task = current;
	waiter->lock = lock;
	plist_node_init(&waiter->list_entry, current->prio);
	plist_node_init(&waiter->pi_list_entry, current->prio);

	/* 得到Lock1的top waiter,就是A对应的waiter */
	if (rt_mutex_has_waiters(lock))
		top_waiter = rt_mutex_top_waiter(lock);
        /*将当前进程的waiter加入lock的wait_list*/
	plist_add(&waiter->list_entry, &lock->wait_list);
        /*当前进程block到新创建的waiter*/
	current->pi_blocked_on = waiter;

	spin_unlock(¤t->pi_lock);

	if (waiter == rt_mutex_top_waiter(lock)) {/*当前进程变成lock L1的top waiter*/
		/* readers are handled differently */
		if (task_is_reader(owner)) { /*暂时忽略*/
			res = rt_mutex_adjust_readers(lock, waiter,
						      current, lock, 0);
			return res;
		}
                /*调整L1的owner的Pi_waiters队列,重新设定其优先级,这是pi chain调整的一部分*/
		spin_lock(&owner->pi_lock);
		plist_del(&top_waiter->pi_list_entry, &owner->pi_waiters);
		plist_add(&waiter->pi_list_entry, &owner->pi_waiters);

		__rt_mutex_adjust_prio(owner);
		if (owner->pi_blocked_on) /*如果owner(B)依然block on到另一个lock(L2),需要walk PI chian*/
			chain_walk = 1;
		spin_unlock(&owner->pi_lock);
	}
	else if (debug_rt_mutex_detect_deadlock(waiter, detect_deadlock))
		chain_walk = 1;

	if (!chain_walk || task_is_reader(owner))
		return 0;

	/*
	 * The owner can't disappear while holding a lock,
	 * so the owner struct is protected by wait_lock.
	 * Gets dropped in rt_mutex_adjust_prio_chain()!
	 */
	get_task_struct(owner);

	spin_unlock_irqrestore(&lock->wait_lock, flags);

	res = rt_mutex_adjust_prio_chain(owner, detect_deadlock, lock, waiter,
					 current, 0);

	spin_lock_irq(&lock->wait_lock);

	return res;
}


再给出一幅图描述下各个变量位置,对看清楚代码很有好处:



static int rt_mutex_adjust_prio_chain(struct task_struct *task,int deadlock_detect,
				      struct rt_mutex *orig_lock,
				      struct rt_mutex_waiter *orig_waiter,
				      struct task_struct *top_task,
				      int recursion_depth)

{
	struct rt_mutex *lock;
	struct rt_mutex_waiter *waiter, *top_waiter = orig_waiter;
	int detect_deadlock, ret = 0, depth = 0;
	unsigned long flags;

	detect_deadlock = debug_rt_mutex_detect_deadlock(orig_waiter,
							 deadlock_detect);

	/*
	 * The (de)boosting is a step by step approach with a lot of
	 * pitfalls. We want this to be preemptible and we want hold a
	 * maximum of two locks per step. So we have to check
	 * carefully whether things change under us.
	 */
 again:
	if (++depth > max_lock_depth) {/*不能超过最大深度*/
		static int prev_max;

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
		put_task_struct(task);

		return deadlock_detect ? -EDEADLK : 0;
	}
  retry:
	/*
	 * Task can not go away as we did a get_task() before !
	 */
	spin_lock_irqsave(&task->pi_lock, flags);

	waiter = task->pi_blocked_on;
	/*
	 * Check whether the end of the boosting chain has been
	 * reached or the state of the chain has changed while we
	 * dropped the locks. (此task为list top,或者chain已经变化)
	 */
	if (!waiter || !waiter->task)
		goto out_unlock_pi;

	/*
	 * Check the orig_waiter state. After we dropped the locks,
	 * the previous owner of the lock might have released the lock
	 * and made us the pending owner:(最初的task可能已经不再等待而成为pending owner)
	 */
	if (orig_waiter && !orig_waiter->task)
		goto out_unlock_pi;

	/*
	 * Drop out, when the task has no waiters. Note,
	 * top_waiter can be NULL, when we are in the deboosting
	 * mode!
	 */
	if (top_waiter && (!task_has_pi_waiters(task) ||/*chain change*/
			top_waiter != task_top_pi_waiter(task)))/*如果top waiter不是tsk的top pi_waiter
                      他就不会影响更上游的PI chain 节点了*/
		goto out_unlock_pi;

	/*
	 * When deadlock detection is off then we check, if further
	 * priority adjustment is necessary.
	 */
	if (!detect_deadlock && waiter->list_entry.prio == task->prio)
		goto out_unlock_pi;/*waiter的优先级等于进程优先级,无须更新(task,即owner优先级已经调整)*/

	lock = waiter->lock; /*task block on的 lockL2*/
	if (!spin_trylock(&lock->wait_lock)) {
		spin_unlock_irqrestore(&task->pi_lock, flags);
		cpu_relax();
		goto retry; /**/
	}

	/* Deadlock detection */
	if (lock == orig_lock || rt_mutex_owner(lock) == top_task) {
		debug_rt_mutex_deadlock(deadlock_detect, orig_waiter, lock);
		spin_unlock(&lock->wait_lock);
		ret = deadlock_detect ? -EDEADLK : 0;
		goto out_unlock_pi;
	}

        /*推进top_waiter,即L2的top waiter,(这个情景下就是waiter),下图是推进后的示意图*/


	top_waiter = rt_mutex_top_waiter(lock);

	/* Requeue the waiter */
	plist_del(&waiter->list_entry, &lock->wait_list);
	waiter->list_entry.prio = task->prio;
	plist_add(&waiter->list_entry, &lock->wait_list);

	/* Release the task */
	spin_unlock(&task->pi_lock);
	put_task_struct(task);

	/* Grab the next task :D*/
	task = rt_mutex_owner(lock);

	/*
	 * Readers are special. We may need to boost more than one owner.
	 */
	if (task_is_reader(task)) {
		ret = rt_mutex_adjust_readers(orig_lock, orig_waiter,
					      top_task, lock,
					      recursion_depth);
		spin_unlock_irqrestore(&lock->wait_lock, flags);
		goto out;
	}

	get_task_struct(task);
	spin_lock(&task->pi_lock);

	if (waiter == rt_mutex_top_waiter(lock)) {/*Requeue waiter后,waiter成为top waiter*/
		/* Boost the owner */
		plist_del(&top_waiter->pi_list_entry, &task->pi_waiters); /*原来的top_waiter不再是
                                                                                         pi_waiter*/
		waiter->pi_list_entry.prio = waiter->list_entry.prio;
		plist_add(&waiter->pi_list_entry, &task->pi_waiters);
		__rt_mutex_adjust_prio(task);

	} else if (top_waiter == waiter) { /*原来的top_waiter是waiter*/
		/* Deboost the owner */
		plist_del(&waiter->pi_list_entry, &task->pi_waiters);/*waiter不再具备pi_waiter的资格了*/
		waiter = rt_mutex_top_waiter(lock);
		waiter->pi_list_entry.prio = waiter->list_entry.prio;
		plist_add(&waiter->pi_list_entry, &task->pi_waiters);
		__rt_mutex_adjust_prio(task);
	}

	spin_unlock(&task->pi_lock);

	top_waiter = rt_mutex_top_waiter(lock); /*更新topwaiter,进行下一轮的boost*/
	spin_unlock_irqrestore(&lock->wait_lock, flags);

	if (!detect_deadlock && waiter != top_waiter)
		goto out_put_task;

	goto again;

 out_unlock_pi:
	spin_unlock_irqrestore(&task->pi_lock, flags);
 out_put_task:
	put_task_struct(task);
 out:
	return ret;
}

PI chain walk的总结:
  1)top_task, orig_lock, orig_waiter永远是原始的哪个进程和它相关(block on)的pi_waiter,lock.
  2) top_waiter,task是叠代变量
  3) lock和waiter是中间辅助变量

rt_mutex_adjust_prio_chain的任务描述:
static int rt_mutex_adjust_prio_chain(struct task_struct *task,int deadlock_detect,
				      struct rt_mutex *orig_lock,
				      struct rt_mutex_waiter *orig_waiter,
				      struct task_struct *top_task,
				      int recursion_depth)
前提: task由于某种原因,已经进行了优先级调整
功能: 把task优先级的变化沿着PI chain向右传播
        1.task 所block_on的waiter的优先级或许要调整
        2. 同时或许需要调整waiter在lock->wait_list内的位置
        3. 如果调整影响了lock->owner的pi_waiter,调整之
        简言之, 对task的优先级变化做响应, 调整task block_on 的waiter在两个队列内的位置.

注:检测waiter->task是否为NULL的意义,waiter对应的task正在成为pending owner
   owner side 需要唤醒一个top_waiter的时候(见wakeup_next_waiter),把top waiter从两个链表摘除,然后把waitr->task置
空,然后再唤醒top_waiter.(waiter是等待进程栈上分配的变量,这个顺序保证不出问题)
  我们传播一个优先级变化的时候,上述事件可以同时发生,因为我们只锁task->pi_lock,而waiter->task被置空只锁lock->
wait_lock. 检测waiter->task是否为NULL可以在第一时间获取task的状态变化(task要被唤醒了,已经成为pending owner), 不管
最后task是否能够获取到lock(可能被steal)我们都没有必要传播这个优先级变化了:成为owener不用说了, 如果被steal, 执行steal
的进程优先级肯定高于task, 也不用再传播下去了.
  另一个问题是检查waiter->task没有保护(我们锁的是pi_lock, 但是wakeup的时候修改这个变量时锁的是lock->wait_lock.我们最
后尝试锁lock->wait_lock的时候用spin_trylock, 如果没有成功我们得重新来过.
        if (!spin_trylock(&lock->wait_lock)) {
		spin_unlock_irqrestore(&task->pi_lock, flags);
		cpu_relax();
		goto retry; /**/
	}
 问题是从我们检查到trylock成功,释放锁的过程是否会产生竞争从而使判断失效? 看看wakeup_next_waiter的简略逻辑:
/* Called with lock->wait_lock held */
static void wakeup_next_waiter(struct rt_mutex *lock, int savestate)
{
	spin_lock(¤t->pi_lock);

	waiter = rt_mutex_top_waiter(lock);
	plist_del(&waiter->list_entry, &lock->wait_list);

	pendowner = waiter->task;
	waiter->task = NULL;

	if (!savestate)
		wake_up_process(pendowner);
	else {
		smp_mb();

		/* If !RUNNING && !RUNNING_MUTEX */
		if (pendowner->state & ~TASK_RUNNING_MUTEX)
			wake_up_process_mutex(pendowner);
	}

	rt_mutex_set_owner(lock, pendowner, RT_MUTEX_OWNER_PENDING);

	spin_unlock(¤t->pi_lock);

	spin_lock(&pendowner->pi_lock);  /* 这是个关键地方,如果有冲突,比方说PI walk到try lock时,这里正好改变了
           waiter->task 的值, 这个释放lock的进程就会spin在这个pendowner->pi_lock,因为PI walk已持有这个锁...
           从而保证pi walk必定失败,从而能够感知状态改变*/

	pendowner->pi_blocked_on = NULL;

	if (next)
		plist_add(&next->pi_list_entry, &pendowner->pi_waiters);

	spin_unlock(&pendowner->pi_lock);
}

 就是说, wakeup过程会一直持有lock->wait_lcok 直到pi chain walk释放pi_lock,从而pi chain walk可以感知状态的改变(表
明有冲突存在).




Pending Owners and Lock stealing
--------------------------------
上面在解释Transitional state的时候提到了 lock stealing.当锁的owner在释放锁的时候,需要唤醒waiter, 但是为了更好的响应
速度,即让更高优先级的进程获取到这个lock,并不是马上让top watier获取到这个锁,而是先标记其为pending owner,给恰在此时尝试获
取锁的更高优先级的任务一个机会,来抢占这个锁.否则高优先级的任务只能boost锁的owner,这显然比较郁闷....,这个机会就是让高优先级
的任务在这个时候抢到锁,而不是boost 低优先级的任务...
 一个top waiter在标定为pending owner后,必须是他自己开始运行后自行获lock, 这里就是steler的机会.
 lock stealing的作用在这个情景下很有用处: 一个高优先级的任务不断的释放和获取一个锁,这个锁有一个低优先级的waiter,可以想像
没有stealing 的情况是如何的糟糕.

 我们来看看代码,我们的情景是:lock 有 owner, 一个低优先级的waiter, stealer是一个高优先级的抢占者. 在owner释放锁的过程
中,stealer试图获取lock.

static inline void
rt_spin_lock_fastunlock(struct rt_mutex *lock,
			void  (*slowfn)(struct rt_mutex *lock))
{
        ....
	if (likely(rt_mutex_cmpxchg(lock, current, NULL))) /*如果lock有waiter,lock->owner 1 bit被置位*/
		rt_mutex_deadlock_account_unlock(current);
	else
		slowfn(lock);  /*owner只能走slow path来释放锁*/
}
static void  noinline __sched
rt_spin_lock_slowunlock(struct rt_mutex *lock)
{
	....                      
	spin_lock_irqsave(&lock->wait_lock, flags); /*stealer不必要此之前开始尝试获取锁,因为
                    slow unlock只是标定pending owner*/
        .....
	wakeup_next_waiter(lock, 1);

	spin_unlock_irqrestore(&lock->wait_lock, flags);/*给stealer的机会就在这里unlock之后*/
	/* Undo pi boosting.when necessary */
	rt_mutex_adjust_prio(current);
}
/*Called with lock->wait_lock held.*/
static void wakeup_next_waiter(struct rt_mutex *lock, int savestate)
{
	....
	spin_lock(¤t->pi_lock); /*把top waiter从当前owner的pi队列,以及lock的waiter list中摘除*/

	waiter = rt_mutex_top_waiter(lock);
	plist_del(&waiter->list_entry, &lock->wait_list);

	/* 现在还不调整当前进程的优先级(undo boost,undo的时候不用pi walk),唤醒pending waiter后再调整 */
      	plist_del(&waiter->pi_list_entry, ¤t->pi_waiters);
	pendowner = waiter->task;
	waiter->task = NULL;/*boost pi walk(task_blocks_on_rt_mutex)会检查waiter->task=NULL这个动作*/

	/*
	 * Do the wakeup before the ownership change to give any spinning
	 * waiter grantees a headstart over the other threads that will
	 * trigger once owner changes.
	 */
	if (!savestate)
		wake_up_process(pendowner);
	else {
		/*
		 * The waiter-side protocol has the following pattern:
		 * 1: Set state != RUNNING
		 * 2: Conditionally sleep if waiter->task != NULL;
		 * And the owner-side has the following:
		 * A: Set waiter->task = NULL
		 * B: Conditionally wake if the state != RUNNING
		 *
		 * As long as we ensure 1->2 order, and A->B order, we
		 * will never miss a wakeup.
		 */
		smp_mb();
		/* 下面的判断实际上是:If !RUNNING && !RUNNING_MUTEX, RUNNING定义为0*/
		if (pendowner->state & ~TASK_RUNNING_MUTEX)
			wake_up_process_mutex(pendowner);
	}

	rt_mutex_set_owner(lock, pendowner, RT_MUTEX_OWNER_PENDING);/*设置top waiter为pending owner*/
	spin_unlock(¤t->pi_lock);

	/*
	 * Clear the pi_blocked_on variable and enqueue a possible
	 * waiter into the pi_waiters list of the pending owner. This
	 * prevents that in case the pending owner gets unboosted a
	 * waiter with higher priority than pending-owner->normal_prio
	 * is blocked on the unboosted (pending) owner.
	 */
        /*这个情况在如下情景下发生,task(当前优先级,原始优先级),如果B成为pending owner时,不清除pi_block_on,不把C加到B的
         *pi waiter,那么,A有可能放弃这个mutex,见rt_mutex_slowlock->remove_waiter(比如A有pending的signal,或者超时)
         *  1.这里,就是owner把B选定为当前pending owner,从这个函数返回后,lock->wait_lock解锁(rt_mutex_slowunlock)
         *  2.A 由于超时或者信号的原因放弃L1,从而rt_mutex_slowlock->remove_waiter将会unboost B
         *  3.显然,B应该获取C的优先级
         *  4.并且pi chain walk依赖与这里锁定pendowner->pi_lock用以获取同步waiter->task的状态
         */
       /* A(10,10)->L1->B(10,20)->             --->   A(10,10)->L1->
                                 |->L2 ->Owner --->                 |->B
                        C(15,15)->             --->   C(8,8)->L2->
       */
        if (rt_mutex_has_waiters(lock))
		next = rt_mutex_top_waiter(lock);
	else
		next = NULL;

	spin_lock(&pendowner->pi_lock); /*已经讨论过这个锁对pi chain walk的意义了*/
        ...... /*some Waring_on stuff...*/
	pendowner->pi_blocked_on = NULL;

	if (next)
		plist_add(&next->pi_list_entry, &pendowner->pi_waiters);

	spin_unlock(&pendowner->pi_lock);
}

毫无疑问,我们的情景下stealer会走slow path
static void  noinline __sched  rt_spin_lock_slowlock(struct rt_mutex *lock)
{
        ...
        waiter.task = NULL; /*waiter on current stack*/
	waiter.write_lock = 0;

	spin_lock_irqsave(&lock->wait_lock, flags);
	init_lists(lock);

	BUG_ON(rt_mutex_owner(lock) == current);

	/*
	 * Here we save whatever state the task was in originally,
	 * we'll restore it at the end of the function and we'll take
	 * any intermediate wakeup into account as well, independently
	 * of the lock sleep/wakeup mechanism. When we get a real
	 * wakeup the task->state is TASK_RUNNING and we change
	 * saved_state accordingly. If we did not get a real wakeup
	 * then we return with the saved state.
	 */
        /*  关于为什么要保留任务状态,设想如下流程, 如果mylock2被抢占, 再此唤醒时任务状态将变为RUNNING!!!!
            spin_lock(&mylock1);
	        current->state = TASK_UNINTERRUPTIBLE;
	        spin_lock(&mylock2);          // [*] /*在这里被抢占时保存任务状态,获取锁后恢复任务状态*/
	              blah();
	        spin_unlock(&mylock2);
	     spin_unlock(&mylock1);
         */
	saved_state = current->state;

	for (;;) {
		unsigned long saved_flags;
		int saved_lock_depth = current->lock_depth;

		/* Try to acquire the lock */
		if (do_try_to_take_rt_mutex(lock, STEAL_LATERAL)) {/*try lock(maybe steal it)*/
			/* If we never blocked break out now */
			if (!missed)
				goto unlock;
			break;
		}
		missed = 1;

		/*第一次是空,或者被选为pending owner(clear watier.task),但是又被slealing,需要重新block on*/
		if (!waiter.task) {
			task_blocks_on_rt_mutex(lock, &waiter, 0, flags); /*boost pi chain walk*/
			/* Wakeup during boost ? */
			if (unlikely(!waiter.task)) /*boost 进行中被选为pending owner并被stealing*/
				continue;   /*task_blocks_on_rt_mutex 进行chain walk时会解锁lock->wait_lock*/
		}

		/*
		 * schedule() 对BKL有个release, reacquire 的过程, 这里防止BLK被schedule release
		 * 做法就是吧lock_depth替换成-1,之后再恢复 (refer to release_kernel_lock)
		 */
              /*BKL: big kernel lock*/
		saved_flags = current->flags & PF_NOSCHED;
		current->lock_depth = -1;
		current->flags &= ~PF_NOSCHED; /*rtmutex 忽略PF_NOSCHED,之后恢复*/
		orig_owner = rt_mutex_owner(lock);
		spin_unlock_irqrestore(&lock->wait_lock, flags);

		debug_rt_mutex_print_deadlock(&waiter);

		if (adaptive_wait(&waiter, orig_owner)) {/*只有owner转入sleep状态,我们才会sleep,否就spin*/
			update_current(TASK_UNINTERRUPTIBLE, &saved_state);
			/*
			 * The xchg() in update_current() is an implicit
			 * barrier which we rely upon to ensure current->state
			 * is visible before we test waiter.task.
                      * 参考上面wakeup_next_waiter中对smp_mb的注解
                      */
			if (waiter.task)
				schedule_rt_mutex(lock);
		}

		spin_lock_irqsave(&lock->wait_lock, flags);
		current->flags |= saved_flags;
		current->lock_depth = saved_lock_depth;
	}

	state = xchg(¤t->state, saved_state); /*恢复任务状态*/
	if (unlikely(state == TASK_RUNNING))
               /*在被mutex owner唤醒时,任务状态将是RUNNING_MUTEX,这也是RUNNING_MUTEX的意义所在
                *见wakeup_next_waiter->wake_up_process_mutex->try_to_wake_up(mutex=1)
                */
		current->state = TASK_RUNNING;

	/*
	 * Extremely rare case, if we got woken up by a non-mutex wakeup,
	 * and we managed to steal the lock despite us not being the
	 * highest-prio waiter (due to SCHED_OTHER changing prio), then we
	 * can end up with a non-NULL waiter.task: (refer to prio_changed_fair)
        *0.  当前进程block on到lock 上(注意是block on到锁上,不是到进程)
        *1.  adaptive_wait 由于事件2,从而发生owner change,所以保持waiter.task不为空,且不睡眠 (held no lock)
        *2.  owner释放锁,另一个进程被选定为pending owner, then release lock
        *2.x 同时此进程被(boost?)改变优先级,变成pending onwer的top waiter,并且pending owner也被boosted(sth else?)
        *3.  新一轮循环中,do_try_to_take_rt_mutex->try_to_steal_lock 获得锁,并保持waiter.task非空  
        * question :seems pending owner's block_on is null, so remove_waiter really need chain walk?
        * (crruent的waiter应该已经从pending owner->pi_waiter中删除见,try_sealing_lock,这个例子好像不对)  
        */
	if (unlikely(waiter.task))
		remove_waiter(lock, &waiter, flags);
	/*
	 * try_to_take_rt_mutex() sets the waiter bit
	 * unconditionally. We might have to fix that up:
	 */
	fixup_rt_mutex_waiters(lock);

 unlock:
	spin_unlock_irqrestore(&lock->wait_lock, flags);

	debug_rt_mutex_free_waiter(&waiter);
}

 /* Must be called with lock->wait_lock held.*/
static int do_try_to_take_rt_mutex(struct rt_mutex *lock, int mode)
{
	/*
	 * We have to be careful here if the atomic speedups are
	 * enabled, such that, when
	 *  - no other waiter is on the lock
	 *  - the lock has been released since we did the cmpxchg
	 * the lock can be released or taken while we are doing the
	 * checks and marking the lock with RT_MUTEX_HAS_WAITERS.
	 *
	 * The atomic acquire/release aware variant of
	 * mark_rt_mutex_waiters uses a cmpxchg loop. After setting
	 * the WAITERS bit, the atomic release / acquire can not
	 * happen anymore and lock->wait_lock protects us from the
	 * non-atomic case.
	 *
	 * Note, that this might set lock->owner =
	 * RT_MUTEX_HAS_WAITERS in the case the lock is not contended
	 * any more. This is fixed up when we take the ownership.
	 * This is the transitional state explained at the top of this file.
	 */
	mark_rt_mutex_waiters(lock); /*对atomic acquire/release,即xx_fast_lock的互斥*/

       /*在owner处于pending状态,且优先级高于pending owner时,可以抢占同时吧top waiter从pending owner的pi_waiter
        *转移到当前进程的pi_waiter上(wakeup_next_waiter加进来的, 这里逆操作)
        */
	if (rt_mutex_owner(lock) && !try_to_steal_lock(lock, mode))
		return 0;

	/* We got the lock. */
	debug_rt_mutex_lock(lock);

	rt_mutex_set_owner(lock, current, 0);

	rt_mutex_deadlock_account_lock(lock, current);

	return 1;
}
lock/unlock 操作概要     
暂略。

```          
