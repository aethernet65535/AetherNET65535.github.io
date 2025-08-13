+++
title = "DESIGN OF CFS - DEEPSEEK TRANSLATION"
date = 2025-08-13

[taxonomies]
tags = ["linux"]

[extra]
comment = true
+++     

由DEEPSEEK V3翻译：     

# 翻译版                      

## 概述
========

CFS（Completely Fair Scheduler，完全公平调度器）由Ingo Molnar开发，自Linux 2.6.23版本并入内核，最初用于替代原调度器中SCHED_OTHER类型的交互式代码。当前CFS正逐步为EEVDF调度器让路，相关文档见Documentation/scheduler/sched-eevdf.rst。

CFS的设计理念可用一句话概括：它试图在真实硬件上模拟"理想的多任务处理器"的行为。所谓"理想多任务处理器"是指一种（现实中不存在的）能并行精确分配计算资源的CPU——当运行n个任务时，每个任务始终能获得1/n的物理算力。例如：两个任务运行时，每个任务能获得50%的实际并行算力。

由于真实硬件一次只能运行单个任务，CFS引入了"虚拟运行时（virtual runtime）"概念。任务的虚拟运行时表示其在理想多任务CPU上获得的下一个时间片起始时刻。实际实现中，虚拟运行时是任务实际运行时间经任务总数标准化后的值。

## 关键实现细节
================

CFS通过p->se.vruntime（纳秒单位）来追踪每个任务的虚拟运行时，借此精确计量任务应获得的CPU时间。

理想硬件上所有任务的p->se.vruntime值应始终相同，意味着所有任务能同步执行且永远不会偏离理想的CPU时间分配。CFS的调度策略基于该值实现：总是选择p->se.vruntime最小的任务（即累计执行时间最短的任务），尽可能接近理想多任务处理器的分配效果。

其他设计要素如nice级别、多处理器支持等均围绕这一核心思想展开。

## 红黑树机制
==============

CFS采用革命性设计：使用时间排序的红黑树构建任务执行时间线，摒弃传统运行队列结构，从而消除旧调度器（包括RSDL/SD）的"数组切换"问题。

关键数据结构包括：
- rq->cfs.min_vruntime：单调递增值，记录运行队列中最小的vruntime
- rq->cfs.load：运行队列中所有任务的权重总和

调度器总是选择红黑树最左端任务执行。随着系统运行，已执行任务会被逐渐移到红黑树右侧，确保每个任务都能在确定时间内获得CPU资源。

具体流程：任务运行一段时间后，其实际CPU占用时间会被累加到p->se.vruntime。当该值超过当前最左端任务的vruntime（考虑调度粒度避免频繁切换），则触发任务切换。

## 核心特性
============

- 纳秒级精度计时，不依赖jiffies或HZ
- 唯一可调参数：/sys/kernel/debug/sched/base_slice_ns（用于调整桌面/服务器场景的延迟特性）
- 天然免疫传统调度器的攻击向量（如fiftyp.c等测试用例）
- 更严格的nice级别和SCHED_BATCH隔离
- 简化的SMP负载均衡实现

## 调度策略
============

实现三种策略：
- SCHED_NORMAL（原SCHED_OTHER）：常规任务
- SCHED_BATCH：适合批处理作业，减少抢占
- SCHED_IDLE：优先级低于nice 19，但非真正的空闲调度

实时策略（SCHED_FIFO/_RR）由sched/rt.c实现，符合POSIX标准。

## 调度类架构
==============

采用模块化的调度类层次结构，核心模块包括：
- sched/fair.c：实现CFS
- sched/rt.c：简化实现的实时调度器（使用100个运行队列）

通过sched_class结构体实现关键操作钩子：
- enqueue_task/dequeue_task：任务进出可运行状态
- yield_task：任务主动让出CPU
- pick_next_task：选择下一个运行任务
- task_tick：驱动抢占的时间滴答处理

## 组调度扩展
==============

通过以下配置支持任务分组调度：
- CONFIG_CGROUP_SCHED：基础组调度支持
- CONFIG_RT_GROUP_SCHED：实时任务组
- CONFIG_FAIR_GROUP_SCHED：CFS任务组

使用示例：
```bash
# 创建多媒体/浏览器任务组并分配CPU权重
echo 2048 > multimedia/cpu.shares
echo 1024 > browser/cpu.shares
# 将firefox加入浏览器组
echo <pid> > browser/tasks
```

# 原版（对比用）
##  OVERVIEW
============

CFS stands for "Completely Fair Scheduler," and is the "desktop" process
scheduler implemented by Ingo Molnar and merged in Linux 2.6.23. When
originally merged, it was the replacement for the previous vanilla
scheduler's SCHED_OTHER interactivity code. Nowadays, CFS is making room
for EEVDF, for which documentation can be found in
Documentation/scheduler/sched-eevdf.rst.

80% of CFS's design can be summed up in a single sentence: CFS basically models
an "ideal, precise multi-tasking CPU" on real hardware.

"Ideal multi-tasking CPU" is a (non-existent  :-)) CPU that has 100% physical
power and which can run each task at precise equal speed, in parallel, each at
1/nr_running speed.  For example: if there are 2 tasks running, then it runs
each at 50% physical power --- i.e., actually in parallel.

On real hardware, we can run only a single task at once, so we have to
introduce the concept of "virtual runtime."  The virtual runtime of a task
specifies when its next timeslice would start execution on the ideal
multi-tasking CPU described above.  In practice, the virtual runtime of a task
is its actual runtime normalized to the total number of running tasks.



##  FEW IMPLEMENTATION DETAILS
==============================

In CFS the virtual runtime is expressed and tracked via the per-task
p->se.vruntime (nanosec-unit) value.  This way, it's possible to accurately
timestamp and measure the "expected CPU time" a task should have gotten.

   Small detail: on "ideal" hardware, at any time all tasks would have the same
   p->se.vruntime value --- i.e., tasks would execute simultaneously and no task
   would ever get "out of balance" from the "ideal" share of CPU time.

CFS's task picking logic is based on this p->se.vruntime value and it is thus
very simple: it always tries to run the task with the smallest p->se.vruntime
value (i.e., the task which executed least so far).  CFS always tries to split
up CPU time between runnable tasks as close to "ideal multitasking hardware" as
possible.

Most of the rest of CFS's design just falls out of this really simple concept,
with a few add-on embellishments like nice levels, multiprocessing and various
algorithm variants to recognize sleepers.



##  THE RBTREE
==============

CFS's design is quite radical: it does not use the old data structures for the
runqueues, but it uses a time-ordered rbtree to build a "timeline" of future
task execution, and thus has no "array switch" artifacts (by which both the
previous vanilla scheduler and RSDL/SD are affected).

CFS also maintains the rq->cfs.min_vruntime value, which is a monotonic
increasing value tracking the smallest vruntime among all tasks in the
runqueue.  The total amount of work done by the system is tracked using
min_vruntime; that value is used to place newly activated entities on the left
side of the tree as much as possible.

The total number of running tasks in the runqueue is accounted through the
rq->cfs.load value, which is the sum of the weights of the tasks queued on the
runqueue.

CFS maintains a time-ordered rbtree, where all runnable tasks are sorted by the
p->se.vruntime key. CFS picks the "leftmost" task from this tree and sticks to it.
As the system progresses forwards, the executed tasks are put into the tree
more and more to the right --- slowly but surely giving a chance for every task
to become the "leftmost task" and thus get on the CPU within a deterministic
amount of time.

Summing up, CFS works like this: it runs a task a bit, and when the task
schedules (or a scheduler tick happens) the task's CPU usage is "accounted
for": the (small) time it just spent using the physical CPU is added to
p->se.vruntime.  Once p->se.vruntime gets high enough so that another task
becomes the "leftmost task" of the time-ordered rbtree it maintains (plus a
small amount of "granularity" distance relative to the leftmost task so that we
do not over-schedule tasks and trash the cache), then the new leftmost task is
picked and the current task is preempted.



##  SOME FEATURES OF CFS
========================

CFS uses nanosecond granularity accounting and does not rely on any jiffies or
other HZ detail.  Thus the CFS scheduler has no notion of "timeslices" in the
way the previous scheduler had, and has no heuristics whatsoever.  There is
only one central tunable:

   /sys/kernel/debug/sched/base_slice_ns

which can be used to tune the scheduler from "desktop" (i.e., low latencies) to
"server" (i.e., good batching) workloads.  It defaults to a setting suitable
for desktop workloads.  SCHED_BATCH is handled by the CFS scheduler module too.

In case CONFIG_HZ results in base_slice_ns < TICK_NSEC, the value of
base_slice_ns will have little to no impact on the workloads.

Due to its design, the CFS scheduler is not prone to any of the "attacks" that
exist today against the heuristics of the stock scheduler: fiftyp.c, thud.c,
chew.c, ring-test.c, massive_intr.c all work fine and do not impact
interactivity and produce the expected behavior.

The CFS scheduler has a much stronger handling of nice levels and SCHED_BATCH
than the previous vanilla scheduler: both types of workloads are isolated much
more aggressively.

SMP load-balancing has been reworked/sanitized: the runqueue-walking
assumptions are gone from the load-balancing code now, and iterators of the
scheduling modules are used.  The balancing code got quite a bit simpler as a
result.



## Scheduling policies
======================

CFS implements three scheduling policies:

  - SCHED_NORMAL (traditionally called SCHED_OTHER): The scheduling
    policy that is used for regular tasks.

  - SCHED_BATCH: Does not preempt nearly as often as regular tasks
    would, thereby allowing tasks to run longer and make better use of
    caches but at the cost of interactivity. This is well suited for
    batch jobs.

  - SCHED_IDLE: This is even weaker than nice 19, but its not a true
    idle timer scheduler in order to avoid to get into priority
    inversion problems which would deadlock the machine.

SCHED_FIFO/_RR are implemented in sched/rt.c and are as specified by
POSIX.

The command chrt from util-linux-ng 2.13.1.1 can set all of these except
SCHED_IDLE.



## SCHEDULING CLASSES
======================

The new CFS scheduler has been designed in such a way to introduce "Scheduling
Classes," an extensible hierarchy of scheduler modules.  These modules
encapsulate scheduling policy details and are handled by the scheduler core
without the core code assuming too much about them.

sched/fair.c implements the CFS scheduler described above.

sched/rt.c implements SCHED_FIFO and SCHED_RR semantics, in a simpler way than
the previous vanilla scheduler did.  It uses 100 runqueues (for all 100 RT
priority levels, instead of 140 in the previous scheduler) and it needs no
expired array.

Scheduling classes are implemented through the sched_class structure, which
contains hooks to functions that must be called whenever an interesting event
occurs.

This is the (partial) list of the hooks:

 - enqueue_task(...)

   Called when a task enters a runnable state.
   It puts the scheduling entity (task) into the red-black tree and
   increments the nr_running variable.

 - dequeue_task(...)

   When a task is no longer runnable, this function is called to keep the
   corresponding scheduling entity out of the red-black tree.  It decrements
   the nr_running variable.

 - yield_task(...)

   This function is basically just a dequeue followed by an enqueue, unless the
   compat_yield sysctl is turned on; in that case, it places the scheduling
   entity at the right-most end of the red-black tree.

 - wakeup_preempt(...)

   This function checks if a task that entered the runnable state should
   preempt the currently running task.

 - pick_next_task(...)

   This function chooses the most appropriate task eligible to run next.

 - set_next_task(...)

   This function is called when a task changes its scheduling class, changes
   its task group or is scheduled.

 - task_tick(...)

   This function is mostly called from time tick functions; it might lead to
   process switch.  This drives the running preemption.




##  GROUP SCHEDULER EXTENSIONS TO CFS
=====================================

Normally, the scheduler operates on individual tasks and strives to provide
fair CPU time to each task.  Sometimes, it may be desirable to group tasks and
provide fair CPU time to each such task group.  For example, it may be
desirable to first provide fair CPU time to each user on the system and then to
each task belonging to a user.

CONFIG_CGROUP_SCHED strives to achieve exactly that.  It lets tasks to be
grouped and divides CPU time fairly among such groups.

CONFIG_RT_GROUP_SCHED permits to group real-time (i.e., SCHED_FIFO and
SCHED_RR) tasks.

CONFIG_FAIR_GROUP_SCHED permits to group CFS (i.e., SCHED_NORMAL and
SCHED_BATCH) tasks.

   These options need CONFIG_CGROUPS to be defined, and let the administrator
   create arbitrary groups of tasks, using the "cgroup" pseudo filesystem.  See
   Documentation/admin-guide/cgroup-v1/cgroups.rst for more information about this filesystem.

When CONFIG_FAIR_GROUP_SCHED is defined, a "cpu.shares" file is created for each
group created using the pseudo filesystem.  See example steps below to create
task groups and modify their CPU share using the "cgroups" pseudo filesystem::

	# mount -t tmpfs cgroup_root /sys/fs/cgroup
	# mkdir /sys/fs/cgroup/cpu
	# mount -t cgroup -ocpu none /sys/fs/cgroup/cpu
	# cd /sys/fs/cgroup/cpu

	# mkdir multimedia	# create "multimedia" group of tasks
	# mkdir browser		# create "browser" group of tasks

	# #Configure the multimedia group to receive twice the CPU bandwidth
	# #that of browser group

	# echo 2048 > multimedia/cpu.shares
	# echo 1024 > browser/cpu.shares

	# firefox &	# Launch firefox and move it to "browser" group
	# echo <firefox_pid> > browser/tasks

	# #Launch gmplayer (or your favourite movie player)
	# echo <movie_player_pid> > multimedia/tasks    