---
layout: post
title: Linux Preemption - 2
description: Linux 调度器的系列文章。本文主要介绍抢占的基本概念和 Linux 内核的相关实现。 
categories: [Chinese, Software, Hardware]
tags: [scheduler, kernel, linux, hardware]
---

>转载时请包含原文或者作者网站链接：<http://oliveryang.net>

* content
{:toc}

## 1. Scheduler Overview

本文主要围绕 Linux 内核调度器 Preemption 的相关实现进行讨论。其中涉及的一般操作系统和 x86 处理器和硬件概念，可能也适用于其它操作系统。

Linux 调度器的实现实际上主要做了两部分事情，

1. 任务上下文切换

   在 [Preemption Overview](http://oliveryang.net/2016/03/linux-scheduler-1/) 里，我们对任务上下文切换做了简单介绍。
   可以看到，任务上下文切换有两个层次的实现：**公共层**和**处理器架构相关层**。任务运行状态的切换的实现最终与处理器架构密切关联。因此 Linux 做了很好的抽象。
   在不同的处理器架构上，处理器架构相关的代码和公共层的代码相互配合共同实现了任务上下文切换的功能。这也使得任务上下文切换代码可以很容易的移植到不同的处理器架构上。

2. 任务调度策略

   同样的，为了满足不同类型应用场景的调度需求，Linux 调度器也做了模块化处理。调度策略的代码也可被定义两层 **Scheduler Core (调度核心)** 和 **Scheduling Class (调度类)**。
   调度核心的代码实现了调度器任务调度的基本操作，所有具体的调度策略都被封装在具体调度类的实现中。这样，Linux 内核的调度策略就支持了模块化的扩展能力。
   Linux v3.19 支持以下调度类和调度策略，

   * Real Time (实时)调度类 - 支持 SCHED_FIFO 和 SCHED_RR 调度策略。
   * CFS (完全公平)调度类 - 支持 SCHED_OTHER(SCHED_NORMAL)，SCHED_BATCH 和 SCHED_IDLE 调度策略。(注：SCHED_IDLE 是一种调度策略，与 CPU IDLE 进程无关)。
   * Deadline (最后期限)调度类 - 支持 SCHED_DEADLINE 调度策略。

   Linux 调度策略设置的系统调用 [SCHED_SETATTR(2)](http://man7.org/linux/man-pages/man2/sched_setattr.2.html) 的手册有对内核支持的各种调度策略的详细说明。
   内核的调度类和 `sched_setattr` 支持的调度策略命名上不一致但是存在对应关系，而且调度策略的命名更一般化。这样做的一个好处是，同一种调度策略未来可能有不同的内核调度算法来实现。
   新的调度算法必然引入新的调度类。内核引入新调度类的时候，使用这个系统调用的应用不需要去为之修改。调度策略本身也是 POSIX 结构规范的一部分。
   上述调度策略中，SCHED_DEADLINE 是 Linux 独有的，POSIX 规范中并无此调度策略。[SCHED(7)](http://man7.org/linux/man-pages/man7/sched.7.html) 对 Linux 调度 API 和历史发展提供了概览，值得参考。

### 1.1 Scheduler Core

调度器核心代码位于 kernel/sched/core.c 文件。主要包含了以下实现，

* 调度器的初始化，调度域初始化。
* 核心调度函数 `__schedule` 及上下文切换的通用层代码。
* 时钟周期处理的通用层代码，包含 Tick Preemption 的代码。
* 唤醒函数，Per-CPU Run Queue 操作的代码，包含 Wakeup Preemption 通用层的代码。
* 基于高精度定时器中断实现的高精度调度，处理器间调度中断。
* 处理器 IDLE 线程，调度负载均衡，迁移任务的代码。
* 与调度器有关的系统调用的实现代码。

调度器核心代码的主要作用就是调度器的模块化实现，降低了跨处理器平台移植和实现新调度算法模块的重复代码和模块间的耦合度，提高了内核可移植性和可扩展性。

### 1.2 Scheduling Class

在 Linux 内核引入一种新调度算法，基本上就是实现一个新的 Scheduling Class (调度类)。调度类需要实现的所有借口定义在 `struct sched_class` 里。
下面对其中最重要的一些调度类接口做简单的介绍，

* enqueue_task

  将待运行的任务插入到 Per-CPU Run Queue。典型的场景就是内核里的唤醒函数，将被唤醒的任务插入 Run Queue 然后设置任务运行态为 `TASK_RUNNING`。

  对 CFS 调度器来说，则是将任务插入红黑树，给 `nr_running` 增加计数。

* dequeue_task

  将非运行态任务移除出 Per-CPU Run Queue。典型的场景就是任务调度引起阻塞的内核函数，把任务运行态设置成 `TASK_INTERRUPTIBLE` 或 `TASK_UNINTERRUPTIBLE`，然后调用 `schedule` 函数，最终触发本操作。

  对 CFS 调度器来说，则是将不在处于运行态的任务从红黑树中移除，给 `nr_running` 减少计数。

* yield_task

  处于运行态的任务申请主动让出 CPU。典型的场景就是处于运行态的应用调用 [`sched_yield(2)`](http://man7.org/linux/man-pages/man2/sched_yield.2.html) 系统调用，直接让出 CPU。
  此时系统调用 `sched_yield` 系统调用先调用 `yield_task` 申请让出 CPU，然后调用 `schedule` 去做上下文切换。

  对 CFS 调度器来说，如果 `nr_running` 是 1，则直接返回，最终 `schedule` 函数也不产生上下文切换。否则，任务被标记为 skip 状态。调度器在红黑树上选择待运行任务时肯定会跳过该任务。
  之后，因为 `schedule`  函数被调用，`pick_next_task` 最终会被调用。其代码会从红黑树中最左侧选择一个任务，然后把要放弃运行的任务放回红黑树，然后调用上下文切换函数做任务上下文切换。

* check_preempt_curr

  用于在待运行任务插入 Run Queue 后，检查是否应该 Preempt 正在 CPU 运行的当前任务。Wakeup Preemption 的实现逻辑主要在这里。

  对 CFS 调度器而言，主要是在是否能满足调度时延和是否能保证足够任务运行时间之间来取舍。CFS 调度器也提供了预定义的 Threshold 允许做 Wakeup Preemption 的调优。
  本文有专门章节对 Wakeup Preemption 做详细分析。

* pick_next_task

  选择下一个最适合调度的任务，将其从 Run Queue 移除。并且如果前一个任务还保持在运行态，即没有从 Run Queue 移除，则将当前的任务重新放回到 Run Queue。内核 `schedule` 函数利用它来完成调度时任务的选择。

  对 CFS 调度器而言，大多数情况下，下一个调度任务是从红黑树的最左侧节点选择并移除。
  如果前一个任务是其它调度类，则调用该调度类的 `put_prev_task` 方法将前一个任务做正确的安置处理。
  但如果前一个任务如果也属于 CFS 调度类的话，为了效率，跳过调度类标准方法 `put_prev_task`，但核心逻辑仍旧是 `put_prev_task_fair` 的主要部分。
  关于 `put_prev_task` 的具体功能，请参考随后的说明。

* put_prev_task

  将前一个正在 CPU 上运行的任务做拿下 CPU 的处理。如果任务还在运行态则将任务放回 Run Queue，否则，根据调度类要求做简单处理。此函数通常是 `pick_next_task` 的密切关联操作，是 `schedule` 实现的关键部分。

  如果前一个任务属于 CFS 调度类，则使用 CFS 调度类的具体实现 `put_prev_task_fair`。此时，如果任务还是 `TASK_RUNNING` 状态，则被重新插入到红黑树的最右侧。
  如果这个任务不是 `TASK_RUNNING` 状态，则已经从红黑树移除过了，只需要修改 CFS 当前任务指针 `cfs_rq->curr` 即可。

* select_task_rq

  为给定的任务选择一个 Run Queue，返回 Run Queue 所属的 CPU 号。典型的使用场景是唤醒，fork/exec 进程时，给进程选择一个 Run Queue，这也给调度器一个 CPU 负载均衡的机会。

  对 CFS 调度器而言，主要是根据传入的参数要求找到符合亲和性要求的最空闲的 CPU 所属的 Run Queue。

* set_curr_task

  当任务改变自己的调度类或者任务组时，该函数被调用。用户进程可以使用 [`sched_setscheduler`](http://man7.org/linux/man-pages/man2/sched_setscheduler.2.html)
  系统调用，通过设置自己新的调度策略来修改自己的调度类。

  对 CFS 调度器而言，当任务把自己调度类从其它类型修改成 CFS 调度类，此时需要把该任务设置成正当前 CPU 正在运行的任务。例如把任务从红黑树上移除，设置 CFS 当前任务指针 `cfs_rq->curr` 和调度统计数据等。

* task_tick

  这个函数通常在系统周期性 (Per-tick) 的时钟中断上下文调用，调度类可以把 Per-tick 处理的事务交给该方法执行。例如，调度器的统计数据更新，Tick Preemption 的实现逻辑主要在这里。
  Tick Preemption 主要判断是否当前运行任务需要 Preemption 来被强制剥夺运行。

  对 CFS 调度器而言，Tick Preemption 主要是在是否能满足调度时延和是否能保证足够任务运行时间之间来取舍。CFS 调度器也提供了预定义的 Threshold 允许做 Tick Preemption 的调优。
  需要进一步了解 Tick Preemption，请参考 2.1 章节。

Linux 内核的 CFS 调度算法就是通过实现该调度类结构来实现其主要逻辑的，CFS 的代码主要集中在 kernel/sched/fair.c 源文件。
下面的 `sched_class` 结构初始化代码包含了本节介绍的所有方法在 CFS 调度器实现中的入口函数名称，

	const struct sched_class fair_sched_class = {

			[...snipped...]

		.enqueue_task		= enqueue_task_fair,
		.dequeue_task		= dequeue_task_fair,
		.yield_task		= yield_task_fair,

			[...snipped...]

		.check_preempt_curr	= check_preempt_wakeup,

			[...snipped...]

		.pick_next_task		= pick_next_task_fair,
		.put_prev_task      = put_prev_task_fair,

			[...snipped...]

		.select_task_rq     = select_task_rq_fair,

			[...snipped...]

		.set_curr_task          = set_curr_task_fair,

			[...snipped...]

		.task_tick		= task_tick_fair,

			[...snipped...]
	};

### 1.3 preempt_count

Linux 内核为支持 Kernel Preemption 而引入了 `preempt_count` 计数器。如果 `preempt_count` 为 0，就允许 Kernel Preemption，否则就不允许。
内核函数 `preempt_disable` 和 `preempt_enable` 用来内核代码的临界区动态关闭和打开 Kernel Preemption。其主要原理就是要通过对这个计数器的加和减来实现关闭和打开。

一般而言，打开 Kernel Preemption 特性的内核，在尽可能的情况下，允许在内核态通过 Tick Preemption 和 Wakeup Preemption 去触发和执行 Kernel Preemption。
但在以下情形，Kernel Preemption 会有关闭和打开操作，

* 内核显式调用  `preempt_disable` 关闭抢占期间，
* 进入中断上下文时，`preempt_count` 计数器被加操作置为非零。退出中断时打开 Kernel Preemption。
* 获取各种内核锁以后，`preempt_disable` 被间接调用。退出内核锁会有 `preempt_enable` 操作。

关于`preempt_disable` 和 `preempt_enable` 的用法，请参考 [Proper Locking Under a Preemptible Kernel](https://github.com/torvalds/linux/blob/v3.19/Documentation/preempt-locking.txt)。

早期 Linux 内核，`preempt_count` 是每个任务所属的 `struct thread_info` 里的一个成员，是 Per-thread 的。

而在 Linux 新内核，为了优化 Kernel Preemption 带来的频繁检查 `preempt_count` 的开销，[Linus 和调度器的维护者决定对其做更多的优化](https://lwn.net/Articles/563185/)。
因此，[Per-CPU `preempt_count` 的优化](https://github.com/torvalds/linux/commit/c2daa3bed53a81171cf8c1a36db798e82b91afe8)被集成到 3.13 版内核。
所以，新内核的的 `preempt_count` 定义如下，

	DECLARE_PER_CPU(int, __preempt_count);

源代码里有对这个计数器不同位意义的详细说明，

	/*
	 * We put the hardirq and softirq counter into the preemption
	 * counter. The bitmask has the following meaning:
	 *
	 * - bits 0-7 are the preemption count (max preemption depth: 256)
	 * - bits 8-15 are the softirq count (max # of softirqs: 256)
	 *
	 * The hardirq count could in theory be the same as the number of
	 * interrupts in the system, but we run all interrupt handlers with
	 * interrupts disabled, so we cannot have nesting interrupts. Though
	 * there are a few palaeontologic drivers which reenable interrupts in
	 * the handler, so we need more than one bit here.
	 *
	 * PREEMPT_MASK:	0x000000ff
	 * SOFTIRQ_MASK:	0x0000ff00
	 * HARDIRQ_MASK:	0x000f0000
	 *     NMI_MASK:	0x00100000
	 * PREEMPT_ACTIVE:	0x00200000
	 */

这里要特别注意的是，如上面的注释中所说，`preempt_count` 的设计时允许嵌套的。例如，

- 内核的锁原语都有 `preempt_disable` 和 `preempt_enable`，而锁是可以嵌套的。
- 内核在拿锁的同时，被中断打断，锁和中断进入的代码路径里也会有 `preempt_count` 操作。

在有嵌套调用的情况下，调用 `preempt_enable` 时 `preempt_count` 也不会立刻减成零。
[sched: likely profiling](https://github.com/torvalds/linux/commit/beed33a816204cb402c69266475b6a60a2433ceb) 这个布丁的最后一个 `unlikely` 到 `likely` 的优化很好的说明了一点：

	在内核里，`preempt_enable` 在 `preempt_disable` 和中断关闭情形下的调用比例更高。

另外要特别注意 `PREEMPT_ACTIVE` 位的用法，

- 在内核打开 Kernel Preemption 的时候，`PREEMPT_ACTIVE` 用来指示 `__schedule` 函数正确处理 Kernel Preemption 语义，防止被打断的即将睡眠的任务被从 Run Queue 误删。
- 在内核关闭 Kernel Preemption 时，虽然只有 User Preemption，`cond_resched` 还是利用这个标志来判断内核调度器是否初始化完成。

以上两点在后续的 User Preemption 和 Kernel Preemption 的相关章节会展开介绍。

## 2. 触发抢占

### 2.1 Tick Preemption

[Preemption Overview](http://oliveryang.net/2016/03/linux-scheduler-1/) 里对时钟中断和 Tick Preemption 都有简单的介绍。本节主要关注 Tick Preemption 在 Linux v3.19 里的实现。

Tick Preemption 的主要逻辑都实现在调度类的 `task_tick` 方法里，调度核心代码里并不做处理。

如前所述，Tick Preemption 主要在时钟中断上下文处理。从时钟中断处理函数到 CFS 调度类的 `task_tick` 方法，中间要经历四个层次的处理。

#### 2.1.1 处理器相关的时钟中断处理

以 x86 为例，时钟中断处理是 Per-CPU 的 Local APIC 定时器中断，中断处理函数 `apic_timer_interrupt` 被初始化到中断门 IDT 的 `LOCAL_TIMER_VECTOR` 上。
当 CPU LAPIC 时钟中断发生时，`apic_timer_interrupt` 中断处理函数会一路调用到处理器无关的通用时钟中断处理函数 `tick_handle_periodic`，

	apic_timer_interrupt->smp_apic_timer_interrupt->local_apic_timer_interrupt->tick_handle_periodic

进一步的细节请参考 arch/x86/kernel/entry_64.S 里的 [`apic_timer_interrupt` 汇编代码](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/entry_64.S#L984)。
特别注意 `apicinterrupt` 宏展开后 `apic_timer_interrupt` 是如何调用 `smp_apic_timer_interrupt` 的汇编技巧。

#### 2.1.2 处理器无关的时钟中断处理

函数 `tick_handle_periodic` 在时钟中断的处理器平台无关层，经过一路调用，最终会进入到调度核心代码层的 `scheduler_tick` 函数，

	tick_handle_periodic->tick_periodic->update_process_times->scheduler_tick

内核时钟中断处理函数要做很多其它复杂的工作，例如，jiffies 和进程时间的维护，Per-CPU 的定时器的调用，RCU 的处理。
进一步的细节请参考 kernel/time/tick-common.c 里的 [`tick_handle_periodic` 的实现](https://github.com/torvalds/linux/blob/v3.19/kernel/time/tick-common.c#L95)。

#### 2.1.3 调度器核心层

函数 `scheduler_tick` 属于调度核心层代码，通过调用当前任务调度类的 `task_tick` 方法，进入到具体调度类的入口函数。对 CFS 调度类而言，就是 `task_tick_fair`，

	void scheduler_tick(void)
	{
		[...snipped...]

		curr->sched_class->task_tick(rq, curr, 0); /* CFS 调度类时，指向 task_tick_fair */

		[...snipped...]
	}

除了 Tick Preemption 处理，`scheduler_tick` 函数还做了调度时钟维护和处理器的负载均衡等工作。
进一步的细节请参考 kernel/sched/core.c 里的 [`scheduler_tick` 的实现](https://github.com/torvalds/linux/blob/v3.19/kernel/sched/core.c#L2514)。

#### 2.1.4 调度类层

如前所述，Linux 支持多种调度类，而且可以通过系统调用设置进程的调度类。不同调度类对 Tick Preemption 的支持可以是不同的，只需要实现 `task_tick` 的方法即可。本小节只关注 CFS 调度类的实现。

CFS 调度器的 `task_tick_fair` 会最终调用到 `check_preempt_tick` 来检查是否需要 Tick Preemption，进而调用 `resched_curr` 申请 Preemption，

	task_tick_fair->entity_tick->check_preempt_tick->resched_curr

进入到 `check_preempt_tick` 之前，`entity_tick` 需要检查本 CPU 所属处于运行状态的任务数是否大于 1，只有一个运行的任务则根本没有必要触发 Preemption。
在 `check_preempt_tick` 内部，主要做以下几件事情，

1. 调用 `sched_slice` 根据 Run Queue 任务数和调度延迟，任务权重计算任务理想运行时间: `ideal_runtime`
2. 计算当前 CPU 上运行任务的运行时间 `delta_exec`
   - 如果 `delta_exec` 已经超过了 `ideal_runtime`，则调用 `resched_curr` 触发 Tick Preemption。
   - 如果 `delta_exec` 小于 CFS 调度器预设的最小调度粒度，则意味着当前任务运行时间太短，直接返回而不触发 Tick Preemption。
3. 计算当前任务和红黑树最左节点任务的虚拟运行时间 `vruntime` 的差值 `delta`
   - 如果 `delta` 小于 0，则意味着没有比当前任务急需调度的任务。
   - 如果 `delta` 大于 `ideal_runtime`，则意味着红黑树里有更需要调度的任务。例如，唤醒后没能做 Wakeup Preemption 的任务可以通过 Tick Preemption 被今早调度。

进一步的细节请参考 kernel/sched/fair.c 里的 [`check_preempt_tick` 的实现](https://github.com/torvalds/linux/blob/v3.19/kernel/sched/fair.c#L3124)。

### 2.2 Wakeup Preemption

如 [Preemption Overview](http://oliveryang.net/2016/03/linux-scheduler-1/) 所述，Wakeup Preemption 与 Linux 内核唤醒机制密切相关。
从唤醒发生到 Wakeup Preemption，涉及到个重要的层次。

#### 2.2.1 同步原语层

Linux 内核里，很多同步原语都会触发进程唤醒，典型的场景如下，

- 锁退出时，唤醒其它等待锁的任务。

  例如，semaphore，mutex，futex 等机制退出时会调用 `wake_up_process` 或 `wake_up_state` 等待该锁的任务列表里的第一个等待任务。

- 等待队列 (wait queue) 或者 completion 机制里，唤醒其它等待在指定等待队列 (wait queue) 或者 completion 上的一个或者多个其它任务。

  Linux 定义了 [wake_up 即其各种变体](https://github.com/torvalds/linux/blob/v3.19/include/linux/wait.h#L165) 主动唤醒等待队列上的任务。

而以上各种机制触发的唤醒任务操作最终都会进入一个共同的入口点 `try_to_wake_up`。

#### 2.2.2 调度器核心层

作为调度核心层代码，[`try_to_wake_up` 定义在 kernel/sched/core.c 原文件里](https://github.com/torvalds/linux/blob/v3.19/kernel/sched/core.c#L1673)。
首先 `try_to_wake_up` 会使用 `select_task_rq` 方法为要被唤醒的任务选择一个 Run Queue，返回目标 Run Queue 所属的 CPU 号。
然后，`ttwu_queue` 的代码会判断这个选择的 CPU 与执行 `try_to_wake_up` 任务的当前运行的 CPU 是否 **共享缓存** (即 LLC，最后一级 cache，x86 就是 L3 Cache)。
在 [Preemption Overview](http://oliveryang.net/2016/03/linux-scheduler-1/) 的相关章节里，针对共享缓存的两个不同情况都做了详细的介绍。因此，这里只给出相关的代码调用路径，作为参考，

- 共享缓存

  这类唤醒是**同步**的，在调用唤醒任务当前的上下文完成。从 `try_to_wake_up` 到调用到触发 Wakeup Preemption 检查的代码路径如下，

	  try_to_wake_up->ttwu_queue->ttwu_do_activate->ttwu_activate->activate_task->enqueue_task->...
	                                         |
	                                         +->ttwu_do_wakeup->check_preempt_curr->...

  唤醒任务被调用具体调度类的 `enqueue_task` 方法插入到目标 CPU Run Queue 之后，再调用 `check_preempt_curr` 来检查是否触发 Wakeup Preemption。

- 不共享缓存

  这类唤醒是**异步**的，调用唤醒任务当前的上文只是将待唤醒的任务加入到目标 CPU Run Queue 的专用唤醒队列里 (`wake_list`)，然后给目标 CPU 触发调度处理器间中断 (IPI) 后，立即返回，

	  try_to_wake_up->ttwu_queue->ttwu_queue_remote->smp_send_reschedule->...

  其中 `smp_send_reschedule` 函数是处理器相关的调度 IPI 触发函数，在不同处理器架构实现时不同的，下个小节会简单介绍。

  调度 IPI 触发后，目标 CPU 会接收到该中断，然后通过处理器相关的中断处理函数调入到调度核心层的中断处理函数 `scheduler_ipi`。
  在这个 `scheduler_ipi` 处理上下文中，任务通过 `sched_ttwu_pending` 调用 `ttwu_activate` 被插入目标 CPU Run Queue，然后最终的 Wakeup Preemption 检查和触发代码会被调用，

	  scheduler_ipi->sched_ttwu_pending->ttwu_do_activate->ttwu_do_wakeup->check_preempt_curr->...

  总之，不共享缓存的情况下，Linux 内核通过实现异步的唤醒操作，将任务实际唤醒操作的下半部分移到被唤醒任务所在 Run Queue 的 CPU 上的 IPI 中断处理上下文中执行。
  这样做的好处主要是减少同步唤醒操作的 Run Queue 锁竞争和缓存方面的开销。
  详情请参考 [sched: Move the second half of ttwu() to the remote cpu](https://github.com/torvalds/linux/commit/317f394160e9beb97d19a84c39b7e5eb3d7815a8)。

此外，在 `try_to_wake_up` 函数一进入时，还有一个特殊情况的检查：当被唤醒任务还在 Run Queue 上没有被删除时 (如睡眠途中)，则代码走如下快速处理路径，

	try_to_wake_up->ttwu_remote->ttwu_do_wakeup->check_preempt_curr->...

注意此时不需要有 Run queue 的选择和插入操作，因此不需要调用 `ttwu_do_activate`，而是直接调用 `ttwu_do_wakeup`。


函数 `check_preempt_curr` 是核心调度器的代码，主要的处理逻辑有三点，

1. 检查目标 Run Queue 所属 CPU 上正在运行的任务和唤醒的任务是否同属一个调度类

   - 如果是相同调度类，则调用具体调度类的 `check_preempt_curr` 方法来处理真正的 Wakeup Preemption。
   - 如果不是相同的调度类，如果被唤醒任务的调度类优先级比当前 CPU 运行任务高，则直接调用 `resched_curr` 触发 Wakeup Preemption 申请。否则，则直接退出，没有抢占资格

2. 函数退出前检查是否进入 `check_preempt_curr` 之前是否发生过队列插入操作，并且是否  Wakeup Preemption 申请成功。

   如果两个条件都满足，就把 `skip_clock_update` 置 1，这样接下来的 `__schedule` 调用里，`update_rq_clock` 会被调用，但在这个函数里会跳过整个函数的处理，这算是个小小的优化。
   因为在插入队列操作时，同样的 `update_rq_clock` 已经被调用过了。

下面是 `check_preempt_curr` 的代码，关键行有代码注释，

	void check_preempt_curr(struct rq *rq, struct task_struct *p, int flags)
	{
		const struct sched_class *class;

		if (p->sched_class == rq->curr->sched_class) {
			rq->curr->sched_class->check_preempt_curr(rq, p, flags); /* 具体调度类方法，CFS 的是 check_preempt_wakeup */
		} else {
			for_each_class(class) { /* 调度类的优先级就是链表的顺序: DL > RT > CFS > IDLE */
				if (class == rq->curr->sched_class)	/* 当前 CPU 运行任务先匹配，意味着新唤醒的调度类优先级低 */
					break;
				if (class == p->sched_class) { /* 新唤醒的任务先匹配到，说明当前 CPU 上的优先级低 */
					resched_curr(rq); /* 触发 Wakeup Preemption */
					break;
				}
			}
		}

		/*
		 * A queue event has occurred, and we're going to schedule.  In
		 * this case, we can save a useless back to back clock update.
		 */
		if (task_on_rq_queued(rq->curr) && test_tsk_need_resched(rq->curr))	/* 满足条件即可以跳过一次 update_rq_clock */
			rq->skip_clock_update = 1;
	}

#### 2.2.3 处理器相关的调度中断处理

上一小节里，介绍了被唤醒任务的目标运行 CPU 和执行 `try_to_wake_up` 的任务所在 CPU 不共享缓存时，需要利用调度 IPI 执行异步唤醒流程。
整个过程中，主要分为以下两个阶段，

- 前半部：触发异步唤醒的调度 IPI

  从 `try_to_wake_up` 一直调用到处理器相关实现 `smp_send_reschedule`，

	  try_to_wake_up->ttwu_queue->ttwu_queue_remote->smp_send_reschedule->...

  以 Intel x86 平台为例，[`smp_send_reschedule` 最终调用 `smp_ops` 结构的 `smp_send_reschedule` 方法](https://github.com/torvalds/linux/blob/v3.19/arch/x86/include/asm/smp.h#L137)。
  而 [`smp_send_reschedule` 方法则早已被初始化成 `native_smp_send_reschedule`](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/smp.c#L350)。

  在 `native_smp_send_reschedule` 函数里，调用 apic 的 `send_IPI_mask` 方法给指定 CPU 的中断向量 `RESCHEDULE_VECTOR` 触发中断，

	  apic->send_IPI_mask(cpumask_of(cpu), RESCHEDULE_VECTOR);

  在大于 8  个 CPU 的 x86 平台上，[apic 结构的 send_IPI_mask 成员被初始化成 physflat_send_IPI_mask](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/apic/apic_flat_64.c#L296)，

	  physflat_send_IPI_mask->default_send_IPI_mask_sequence_phys->__default_send_IPI_dest_field->__default_send_IPI_dest_field

  而在 [`__default_send_IPI_dest_field` 函数里，代码通过对 `APIC_ICR` 寄存器编程来触发 IPI](https://github.com/torvalds/linux/blob/v3.19/arch/x86/include/asm/ipi.h#L88)。

- 后半部：处理调度 IPI 中断

  IPI 触发后，目标 CPU 的 Local APIC 收到中断，陷入中断门，而
  [IDT 表的 `RESCHEDULE_VECTOR` 向量的中断处理入口被初始化成了 `reschedule_interrupt`](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/irqinit.c#L109)，
  因此，从 `reschedule_interrupt` 一直会调用到 `scheduler_ipi`，

	  reschedule_interrupt->smp_reschedule_interrupt->__smp_reschedule_interrupt->scheduler_ipi->...

  其中，[reschedule_interrupt 调用 smp_reschedule_interrupt 的汇编技巧](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/entry_64.S#L1008) 与之前介绍的时钟中断类似。

  上小节中，已经介绍了 `scheduler_ipi` 的代码是通过调用 `sched_ttwu_pending` 来实现异步唤醒的下半部操作，并触发 Wakeup Preemption 的。

至此，通过本节和上节的描述，我们可以清楚的知道处理器相关的调度 IPI 处理代码是如何与调度器核心代码紧密合作，实现异步唤醒并触发 Wakeup Preemption 的。
此外，调度 IPI 更重要的一个功能就是触发真正的 User Preemption 和 Kernel Preemption，这部分在 [Preemption Overview](http://oliveryang.net/2016/03/linux-scheduler-1/) 已有相关说明，此处不再赘述。

#### 2.2.4 调度类层

本节以 CFS 调度类为例，介绍 `check_preempt_curr` 在 CFS 调度类里的实现。

如前所述，在调度核心层的 `check_preempt_curr` 函数如果发现被唤醒的任务和正在被唤醒任务目标 CPU 上运行的任务共同属于一个调度类，则立即调用具体调度类的 `check_preempt_curr` 方法。
具体触发 Wakeup Preemption 的代码路径如下，

	check_preempt_curr->check_preempt_wakeup->resched_curr

如上所示，CFS 调度类里，该方法的具体实现为 `check_preempt_wakeup`。这个函数主要做以下工作，

1. 如果被唤醒的任务已经被目标 CPU 调度运行，立即返回。
2. 如果唤醒的任务处于被 throttled 节流状态 (CFS 带宽控制)，就不做抢占。因为 throttled 的任务已经睡眠。
3. 如果 NEXT_BUDDY 特性被打开，则调用 `set_next_buddy` 标记任务。该任务会在[下次调度调用 `pick_next_entity` 被优先选择](https://github.com/torvalds/linux/blob/v3.19/kernel/sched/fair.c#L3247)。
4. 如果 `TIF_NEED_RESCHED` 已经被置位，则已经申请 Preemption 成功，退出。
5. 如果当前正在运行的 CFS 调度类任务的调度策略是 SCHED_IDLE，而当前被唤醒任务不是这个调度策略，则肯定当前任务有更高优先级，可以触发 Preemption。
6. 如果被唤醒的 CFS 调度类任务的调度策略是 SCHED_BATCH 或 SCHED_IDLE，或者 Wakeup Preemption 特性没有打开，则退出。
7. 调用 `wakeup_preempt_entity` 函数，判断当前运行任务和被唤醒任务的 vruntime 的差值是否足够大。
   - 如果被唤醒任务 `vruntime` 足够落后，差值大于 `sysctl_sched_wakeup_granularity` 则 Wakeup Preemption 条件成立。
   - 如果被唤醒任务和当前运行任务 `vruntime` 的差值太小，则不满足 Wakeup Preemption 条件。
   - 满足抢占条件的任务会被调用 `set_next_buddy` 标记该任务，该任务会在[下次调度调用 `pick_next_entity` 被优先选择](https://github.com/torvalds/linux/blob/v3.19/kernel/sched/fair.c#L3247)。
   - NEXT_BUDDY 特性默认是关闭的，只能手动设置打开。打开后，当新唤醒任务做 Wakeup Preemption 失败时，被设置为调度器偏爱。但是，Wakeup Preemption 成功的任务会覆盖这个标记。
8. 满足 Wakeup Preemption 条件的情况下，调用 `resched_curr` 触发抢占。
   - 如果 LAST_BUDDY 特性被打开，则调用 `set_last_buddy` 标记该任务，该任务会在[下次调度调用 `pick_next_entity` 被优先选择](https://github.com/torvalds/linux/blob/v3.19/kernel/sched/fair.c#L3241)。
   - 缺省条件下，LAST_BUDDY 特性是打开的，表示调度器偏爱调度上次 Wakeup Preemption 成功的任务。
9. 值得注意的是，在 `pick_next_entity` 的优先选择逻辑里，还要利用 `wakeup_preempt_entity` 保证被标记为偏爱调度的任务和 CFS 红黑树最左侧的任务之间 vruntime 的差值是**足够小**的，否则不公平。

### 2.3 Reschedule Request

不论是 Tick Preemption 还是 Wakeup Preemption，一旦满足抢占条件，都会调用 `resched_curr` 来请求抢占。
这个函数的主要功能如下，

1. 一进入函数，检查是否当前要抢占的 CPU 当前运行的任务已经有人标记了 `TIF_NEED_RESCHED` 标志，如果有，就无需重复请求抢占。
2. 检查目标抢占的 CPU 是否是当前执行 `resched_curr` 的 CPU，如果是，则请求抢占当前运行任务，然后直接返回。
   - 早期 Linux 内核，只需要调用 `set_tsk_need_resched` 给当前任务设置 `struct thread_info` 的 `TIF_NEED_RESCHED` 标志。
   - 新 Linux 内核的实现，除了这一步，还需要调用 `set_preempt_need_resched` 给 Per-CPU `preempt_count` 设置 `PREEMPT_NEED_RESCHED`。
   - 这一个优化主要是为了[减少频繁访问 `struct thread_info` 成员 `preempt_count` 和 flags 的开销](https://github.com/torvalds/linux/commit/c2daa3bed53a81171cf8c1a36db798e82b91afe8)。
3. 如果目标抢占的 CPU 和当前执行 `resched_curr` 的 CPU 不是同一个 CPU，则有以下两种情况，
   - 如果目标 CPU 上正在运行的的任务不是正在轮询 `TIF_NEED_RESCHED` 的 IDLE 线程，则触发一个 cross-CPU call (INtel 叫 IPI) 给目标 CPU
   - 如果想反，目标 CPU 上 真该运行的任务是 IDLE 线程，则不需要 IPI，只需要实现特定的内核 Trace Point。
4. 成功请求 Preemption 后，随后的 scheduler IPI，Timer Interrupt，外设 Interrupt 都可以触发真正的 User Preemption 或者 Kernel Preemption。
   - 早期 Linux 内核，`scheduler_ipi` 被实现为空函数，User or Kernel Preemption 的触发代码应该在 scheduler IPI 退出中断到用户/内核空间来发展。
   - 新 Linux 内核，`scheduler_ipi` 的代码加入了唤醒任务的功能，被用于不共享缓存的的请况下，任务唤醒的下半部，这样可以减少 Run Queue 锁竞争。

下面是相关的代码，

	/*
	 * resched_curr - mark rq's current task 'to be rescheduled now'.
	 *
	 * On UP this means the setting of the need_resched flag, on SMP it
	 * might also involve a cross-CPU call to trigger the scheduler on
	 * the target CPU.
	 */
	void resched_curr(struct rq *rq)
	{
		struct task_struct *curr = rq->curr;
		int cpu;

		lockdep_assert_held(&rq->lock);

		if (test_tsk_need_resched(curr)) /* 是否已经有人申请抢占 */
			return;

		cpu = cpu_of(rq);

		if (cpu == smp_processor_id()) { /* 目标 CPU 与当前运行 CPU 相同 */
			set_tsk_need_resched(curr); /* 标记 `TIF_NEED_RESCHED` */
			set_preempt_need_resched(); /* 标记 `PREEMPT_NEED_RESCHED` */
			return;
		}

		if (set_nr_and_not_polling(curr)) /* 是否是 IDLE 线程正在做轮询 */
			smp_send_reschedule(cpu); /* 在给定 CPU 上触发 IPI，引起 scheduler_ipi 被执行, 间接触发 Preemption. */
		else
			trace_sched_wake_idle_without_ipi(cpu);
	}

## 3. 执行 Preemption

### 3.1 User Preemption

如前所述，User Preemption 主要发生在以下两类场景，

- 系统调用，中断，异常时返回用户空间时。

  此处的代码都是和处理器架构相关的，本文都以 x86 64 位 CPU 为例。

  1. 在[系统调用返回用户空间的代码里](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/entry_64.S#L453)检查 `TIF_NEED_RESCHED` 标志，决定是否调用 `schedule`。
  2. 不论是外设中断还是 CPU 的 APIC 中断，都会在
  [entry_64.S 里的中断公共代码里的返回用户空间路径上](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/entry_64.S#L896)检查 `TIF_NEED_RESCHED` 标志，决定是否调用 `schedule`。
  3. [异常返回用户空间的代码](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/entry_64.S#L1433)实际上与中断返回的代码共享相同的代码，`retint_careful`。

- 任务为 `TASK_RUNNING` 状态时，直接或间接地调用 `schedule`

  Linux 内核的 Kernel Preemption 没有打开的话，除了系统调用，中断，异常返回用户空间时发生 Preemption，使用 `cond_resched` 是推荐的方式来防止内核滥用 CPU。
  由于这些代码可以在只有 User Preemption 打开的时候工作，因此本文将此类代码归类为 User Preemption。

  3.13 之前的内核版本，`cond_resched` 在内核代码主动调用它时，先检查 `TIF_NEED_RESCHED` 标志和 `preempt_count` 的 `PREEMPT_ACTIVE` 标志，然后再决定是否调用 `schedule`。
  这里检查 `PREEMPT_ACTIVE` 标志，只是为了[阻止内核使用 `cond_resched` 的代码在调度器初始化完成前执行调度](https://github.com/torvalds/linux/commit/d86ee4809d0329d4aa0d0f2c76c2295a16862799)。

  而 3.13 引入的 [per_CPU 的 preempt_count patch](https://github.com/torvalds/linux/commit/c2daa3bed53a81171cf8c1a36db798e82b91afe8)，
  则将 `TIF_NEED_RESCHED` 标志设置到 preempt_count 里保存，以便一条指令就可以完成原来的两个条件判断。因此，`TIF_NEED_RESCHED` 标志检查的代码变成了只检查 `preempt_count`。
  需要注意的是，虽然 `preempt_count` 已经包含 `TIF_NEED_RESCHED` 标志，但原有的 task_struct::state 的`TIF_NEED_RESCHED` 标志仍旧在 User Preemption 代码里发挥作用。

  这里不再分析 `yield` 的实现。但需要注意的是，内核中的循环代码应该尽量使用 `cond_resched` 来让出 CPU，而不是使用 `yield`。
  [详见 `yield` 的注释](https://github.com/torvalds/linux/blob/v3.19/kernel/sched/core.c#L4287)。
  POSIX 规范里规定了 [`sched_yield(2)`](http://man7.org/linux/man-pages/man2/sched_yield.2.html) 调用，一些实时调度类的应用可以使用 `sched_yield` 让出 CPU。
  内核 API `yield` 使用了 `sched_yield` 的实现。与 `cond_resched` 最大的不同是，`yield` 会使用具体调度类的 `yield_task` 方法。不同调度类对 `yield_task` 可以有很大不同。
  例如，`SCHED_DEADLINE` 调度策略里，`yield_task` 方法会让任务睡眠，这时的 `sched_yield` 已经不再属于 Preemption 的范畴。

#### 3.1.1 schedule 对 User Preemption 的处理

User Preemption 的代码同样是显示地调用 schedule 函数，但与主动上下文切换中很大的不同是，调用 schedule 函数时，当前上下文任务的状态还是 **TASK_RUNNING**。
只要调用 schedule 时当前任务是 TASK_RUNNING，这时 schedule 的代码就把这次上下文切换算作强制上下文切换，并且这次上下文切换不会涉及到把被 Preempt 任务从 Run Queue 移除操作。

下面是 schedule 代码在 Linux 3.19 的实现，

	static void __sched __schedule(void)
	{
		struct task_struct *prev, *next;
		unsigned long *switch_count;
		struct rq *rq;
		int cpu;

		[...snipped...]

		raw_spin_lock_irq(&rq->lock);

		switch_count = &prev->nivcsw; /* 默认 switch_count 是强制上下文切换的 */
		if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) { /* User Preemption 是 TASK_RUNNING 且无 PREEMPT_ACTIVE 置位，所以下面代码不会执行 */
			if (unlikely(signal_pending_state(prev->state, prev))) {
				prev->state = TASK_RUNNING;	/* 可中断睡眠有 Pending 信号，只做上下文切换，无需从运行队列移除 */
			} else {
				deactivate_task(rq, prev, DEQUEUE_SLEEP); /* 不是 TASK_RUNNING 且无 PREEMPT_ACTIVE 置位，需要从运行队列移除 */
				prev->on_rq = 0;


			[...snipped...]

			switch_count = &prev->nvcsw; /* 不是 TASK_RUNNING 且无 PREEMPT_ACTIVE 置位，
			                             swtich_count 则指向主动上下文切换计数器 */
		}

		[...snipped...]

		next = pick_next_task(rq, prev);

		[...snipped...]

		if (likely(prev != next)) { /* Run Queue 上真有待调度的任务才做上下文切换 */
			rq->nr_switches++;
			rq->curr = next;
			++*switch_count; /* 此时确实发生了调度，要给 nivcsw 或者 nvcsw 计数器累加 */

			rq = context_switch(rq, prev, next); /* unlocks the rq 真正上下文切换发生 */
			cpu = cpu_of(rq);
		} else
			raw_spin_unlock_irq(&rq->lock);

从代码可以看出，User Preemption 触发的上下文切换，都被算作了**强制上下文切换**。

### 3.2 Kernel Preemption

内核抢占需要打开特定的 Kconfig (CONFIG_PREEMPT=y)。本文只介绍引起 Kernel Preemption 的关键代码。如前所述，Kernel Preemption 主要发生在以下两类场景，

- 中断和异常时返回内核空间时。

  如前面章节介绍，系统调用返回不会发生 Kernel Preemption，但中断和异常则会。
  [中断和异常返回内核空间的代码](https://github.com/torvalds/linux/blob/v3.19/arch/x86/kernel/entry_64.S#L929)是共享同一段实现，
  调用 `preempt_schedule_irq` 来检查 `TIF_NEED_RESCHED` 标志，决定是否调用 `schedule`。

- 禁止抢占上下文结束时。

  内核代码调用 `preempt_enable`，`preempt_check_resched` 和 `preempt_schedule` 退出禁止抢占的临界区。下面主要针对这部分实现做详细介绍。

如 [Preemption Overview](http://oliveryang.net/2016/03/linux-scheduler-1/) 所述，User Preemption 总是限定在任务处于 `TASK_RUNNING` 的几个有限的固定时机发生。
而 Kernel Preemption 发生时，任务的运行态是不可预料的，任务运行态可能处于任何运行状态，如 `TASK_UNINTERRUPTIBLE` 状态。

一个典型的例子就是，任务睡眠时要先将任务设置成睡眠态，然后再调用 `schedule` 来做真正的睡眠。

	set_current_state(TASK_UNINTERRUPTIBLE);
	/* 中断在 schedule 之前发生，触发 Kernel Preemption */
	schedule();

设置睡眠态和 `schedule` 调用之间并不是原子的操作，大多时候也没有禁止抢占和关中断。这时 Kernel Preemption 如果正好发生在两者之间，那么就会造成我们所说的情况。
上面的例子里，中断恰好在任务被设置成 `TASK_UNINTERRUPTIBLE` 之后发生。中断退出后，`preempt_schedule_irq` 就会触发 Kernel Preemption。

下面的例子里，Kernel Preemption 可以发生在最后一个 `spin_unlock` 退出时，这时当前任务状态是 `TASK_UNINTERRUPTIBLE`，

	prepare_to_wait(wq, &wait.wait, TASK_UNINTERRUPTIBLE);
	spin_unlock(&inode->i_lock);
	spin_unlock(&inode_hash_lock); /* preempt_enable 在 spin_unlock 内部被调用 */
	schedule();

不论是中断退出代码调用 `preempt_schedule_irq`， 还是 `preempt_enable` 调用 `preempt_schedule`，都会最在满足条件时触发 Kernel Preemption。
下面以 `preempt_enable` 调用 `preempt_schedule` 为例，剖析内核代码实现。

#### 3.2.1 preempt_disable 和 preempt_enable

在内核中需要禁止抢占的临界区代码，直接使用 preempt_disable 和 preempt_enable 即可达到目的。
关于为何以及如何禁止抢占，请参考 [Proper Locking Under a Preemptible Kernel](https://github.com/torvalds/linux/blob/v3.19/Documentation/preempt-locking.txt) 这篇文档。

如 [Preemption Overview](http://oliveryang.net/2016/03/linux-scheduler-1/) 所述，preempt_disable 和 preempt_enable 函数也被嵌入到很多内核函数的实现里，例如各种锁的进入和退出函数。

以 [preempt_enable](https://github.com/torvalds/linux/blob/v3.19/include/linux/preempt.h#L53) 的代码为例，如果 preempt_count 为 0，则调用 `__preempt_schedule`，
而该函数会最终调用 `preempt_schedule` 来尝试内核抢占。

	#define preempt_enable() \
	do { \
		barrier(); \
		if (unlikely(preempt_count_dec_and_test())) \
			__preempt_schedule(); \  /* 最终会调用 preempt_schedule */
	} while (0)

#### 3.2.2 preempt_schedule

在 `preempt_schedule` 函数内部，在调用 `schedule` 之前，做如下检查，

1. 检查 `preempt_count` 是否非零和 IRQ 是否处于 disabled 状态，如果是则不允许抢占。

   做这个检查是为防止抢占的嵌套调用。例如，[`preempt_enable` 可以在关中断时被调用](https://github.com/torvalds/linux/commit/beed33a816204cb402c69266475b6a60a2433ceb)。
   总之，内核并不保证调用 `preempt_enable` 之前，总是可以被抢占的。这是因为，`preempt_enable` 嵌入在很多内核函数里，可以被嵌套间接调用。
   此外，抢占正在进行时也能让这种嵌套的抢占调用不会再次触发抢占。

2. 设置 `preempt_count` 的 `PREEMPT_ACTIVE`，避免抢占发生途中，再有内核抢占。

3. 被抢占的进程再次返回调度点时，检查 `TIF_NEED_RESCHED` 标志，如果有新的内核 Preemption 申请，则再次触发 Kernel Preemption。

   这一步骤是循环条件，直到当前 CPU 的 Run Queue 里再也没有申请 Preemption 的任务。

Linux v3.19 `preempt_schedule` 的代码如下，

	/*
	 * this is the entry point to schedule() from in-kernel preemption
	 * off of preempt_enable. Kernel preemptions off return from interrupt
	 * occur there and call schedule directly.
	 */
	asmlinkage __visible void __sched notrace preempt_schedule(void)
	{
		/*
		 * If there is a non-zero preempt_count or interrupts are disabled,
		 * we do not want to preempt the current task. Just return..
		 */
		if (likely(!preemptible())) /* preempt_enable 可能在被关抢占和关中断后被嵌套调用 */
			return;

		do {
			__preempt_count_add(PREEMPT_ACTIVE); /* 调用 schedule 前，PREEMPT_ACTIVE 被设置 */
			__schedule();
			__preempt_count_sub(PREEMPT_ACTIVE); /* 结束一次抢占，PREEMPT_ACTIVE 被清除 */

			/*
			 * Check again in case we missed a preemption opportunity
			 * between schedule and now.
			 */
			barrier();
		} while (need_resched());	/* 恢复执行时，检查 TIF_NEED_RESCHED 标志是否设置 */
	}

需要注意，`schedule` 调用前，`PREEMPT_ACTIVE` 标志已经被设置好了。

#### 3.2.3 schedule 对 Kernel Preemption 的处理

如前所述，进入函数调用前，`PREEMPT_ACTIVE` 标志已经被设置。根据当前的任务的运行状态，我们分别做出如下分析，

1. 当前任务是 `TASK_RUNNING`。

   任务不会被从其所属 CPU 的 Run Queue 上移除。这时只发生上下文切换，当前任务被下一个任务取代后在 CPU 上运行。

2. 当前任务是其它非运行态。

   继续本节开始的例子，当前任务设置好 `TASK_UNINTERRUPTIBLE` 状态，即将调用 `schedule` 之前被 `spin_unlock` 里的 `preempt_enable` 调用 `preempt_schedule`。

   由于是 Kernel Preemption 上下文，`PREEMPT_ACTIVE` 被设置，任务不会被从 CPU 所属 Run Queue 移除而睡眠，这时只发生上下文切换，当前任务被下一个任务取代在 CPU 上运行。
   当 Run Queue 中已经处于 `TASK_UNINTERRUPTIBLE` 状态的任务被调度到 CPU 上时，`PREEMPT_ACTIVE` 标志早被清除，因此，该任务会被 `deactivate_task` 从 Run Queue 上删除，进入到睡眠状态。

   这样的处理保证了 Kernel Preemption 的正确性，以及后续被 Preempt 任务再度被调度时的正确性，

   * Preemption 的本质是一种打断引起的上下文切换，不应该处理任务的睡眠操作。

     当前被 Preempt 的任务从 Run Queue 移除去睡眠的工作，本来就应该由任务自己代码调用的 `schedule` 来完成。
     假如没有 `PREEMPT_ACTIVE` 标志的检查，那么当前被 Preempt 任务就在 `preempt_schedule` 调用 `schedule` 时提前被从 Run Queue 移除而睡眠。
     这样一来，该任务原来代码的语义发生了变化，从任务角度看，Preemption 只是一种任务打断，被 Preempt 任务的睡眠不应该由 `preempt_schedule` 的代码来做。

   * Run Queue 队列移除操作给 Kernel Preemption 的代码路径被增加了不必要的时延。

     不但如此，这个被 Preempt 任务再次被唤醒后，该任务还未执行的 `schedule` 调用还会被执行一次。

下面是 `schedule` 的代码，针对 Kernel Preemption 做了详细注释，

	static void __sched __schedule(void)
	{
		struct task_struct *prev, *next;
		unsigned long *switch_count;
		struct rq *rq;
		int cpu;

		[...snipped...]

		raw_spin_lock_irq(&rq->lock);

		switch_count = &prev->nivcsw; /* Kernel Preemption 使用强制上下文切换计数器 */
		if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) { /* 非 TASK_RUNNING 和非 Kernel Preemption 任务才从运行队列移除 */
			if (unlikely(signal_pending_state(prev->state, prev))) {
				prev->state = TASK_RUNNING;	/* 可中断睡眠有 Pending 信号，只做上下文切换，无需从运行队列移除 */
			} else {
				deactivate_task(rq, prev, DEQUEUE_SLEEP); /* 非 TASK_RUNNING，非 Kernel Preemption，需要从运行队列移除 */
				prev->on_rq = 0;


			[...snipped...]

				switch_count = &prev->nvcsw; /* 非 TASK_RUNNING 和非 Kernel Preemption 任务使用这个计数器 */
			}
		}

## 4. 关联阅读

* [Linux Preemption - 1](http://oliveryang.net/2016/03/linux-scheduler-1/)
* [Proper Locking Under a Preemptible Kernel](https://github.com/torvalds/linux/blob/v3.19/Documentation/preempt-locking.txt)
* [Optimizing preemption](https://lwn.net/Articles/563185/)
* [Modular Scheduler Core and Completely Fair Scheduler](http://lwn.net/Articles/230501/)
* [CFS scheduler design](https://github.com/torvalds/linux/blob/v3.19/Documentation/scheduler/sched-design-CFS.txt)
* [Deadline scheduling for Linux](http://lwn.net/Articles/356576/)
* [x86 系统调用入门](http://blog.csdn.net/yayong/article/details/416477)
* [Linux Kernel Stack](https://github.com/torvalds/linux/blob/v3.19/Documentation/x86/x86_64/kernel-stacks)
* [Intel 64 and IA-32 Architectures Software Developer's Manual Volume 3](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) 6.14 和 13.4 章节
