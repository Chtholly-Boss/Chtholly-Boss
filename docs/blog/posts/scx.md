---
date:
    created: 2025-04-02
    updated: 2025-04-02
categories:
    - Linux
---
# SCX

In this post, we'll dive into the Extensible Scheduler (SCX) in Linux, a sophisticated framework that empowers users with fine-grained control over task scheduling and resource management.

<!-- more -->

Before proceeding, I strongly recommend reading [sched_ext: a BPF-extensible scheduler class (Part 1)](https://blogs.igalia.com/changwoo/sched-ext-a-bpf-extensible-scheduler-class-part-1/) and [its Part 2](https://blogs.igalia.com/changwoo/sched-ext-scheduler-architecture-and-interfaces-part-2/). These articles provide a thorough examination of SCX's architecture and principles—foundational knowledge that we'll build upon rather than duplicate.

To effectively develop a `sched_ext` program, you need to master three critical aspects:

- **The callback mechanism**: Understanding precisely when the kernel invokes specific callback functions
- **Functional logic**: Determining the intended behavior of each callback function
- **Implementation details**: Crafting efficient code to achieve the desired scheduling behavior

We'll examine the `scx_simple` scheduler as our initial case study, illustrating these key concepts through practical implementation. Following that, we'll advance to the more sophisticated `scx_userland` scheduler, which demonstrates an elegant coordination between kernel and user space components.

Both schedulers are available in [scx/sched/c](https://github.com/sched-ext/scx/tree/main/scheds/c), where you'll also find additional scheduler implementations worth exploring to deepen your understanding.

## scx_simple

In `scx_simple.bpf.c`, we define the following callbacks:

```c
SCX_OPS_DEFINE(simple_ops,
	       .select_cpu		= (void *)simple_select_cpu,
	       .enqueue			= (void *)simple_enqueue,
	       .dispatch		= (void *)simple_dispatch,
	       .running			= (void *)simple_running,
	       .stopping		= (void *)simple_stopping,
	       .enable			= (void *)simple_enable,
	       .init			= (void *)simple_init,
	       .exit			= (void *)simple_exit,
	       .name			= "simple");
```
At the beginning, we may be confused about "which callback should I implement?". The answer is: it depends on which part of the scheduler you want to extend. Let's get into the details.

When the BPF scheduler is initialized/, or we run `sudo scx_simple`, an `init()` callback will be called. So we must implement our `init()` function. In `scx_simple_init()`, we create a dispatch queue:

```c
#define SHARED_DSQ 0
s32 BPF_STRUCT_OPS_SLEEPABLE(simple_init)
{
return scx_bpf_create_dsq(SHARED_DSQ, -1);
}
```

You can refer to [BPF_STRUCT_OPS_SLEEPABLE](https://docs.ebpf.io/ebpf-library/scx/BPF_STRUCT_OPS_SLEEPABLE/) and [scx_bpf_create_dsq](https://docs.ebpf.io/linux/kfuncs/scx_bpf_create_dsq/) for more details. For now, we just need to know that `scx_bpf_create_dsq()` creates a dispatch queue (DSQ).

Correspondingly, when the BPF scheduler is exited, the `exit()` callback will be called. We can see:

```c
UEI_DEFINE(uei);
void BPF_STRUCT_OPS(simple_exit, struct scx_exit_info *ei)
{
UEI_RECORD(uei, ei);
}
```

According to [eunomia/scx-nest](https://eunomia.dev/tutorials/45-scx-nest/), This function records exit information through the `UEI_RECORD` macro, ensuring that any necessary cleanup actions are performed. We can just leave it as is.

Then we come to think about `task_struct`s. Because our scheduler will take over the scheduling logic, all previous tasks will "enable" our scheduler to schedule them. This is done by calling `simple_enable()` in the `enable()` callback. In `simple_enable()`, we just reflush the scheduling stats of the task by:

```c
void BPF_STRUCT_OPS(simple_enable, struct task_struct *p)
{
p->scx.dsq_vtime = vtime_now;
}
```

I will summarize the paragraphs from [sched_ext: a BPF-extensible scheduler class (Part 2)](https://blogs.igalia.com/changwoo/sched-ext-scheduler-architecture-and-interfaces-part-2/) to provide a cheat sheet for the callback functions:

| Event | Callback | Description |
|-------|------------------|-------------|
| New task creation | `init_task()` | Called when a new task is created |
| Task termination | `exit_task()` | Called when a task is terminated |
| Task becomes runnable | `runnable()` | Called when a task becomes ready to run |
| Task picked by scheduler | `running()` | Called when a task is picked and begins running |
| Task's time slice exhausted | `stopping()` | Called when a task is stopped and scheduled out |
| Task becomes non-runnable | `quiescent()` | Called when a task becomes inactive |
| Task wakeup - CPU selection | `select_cpu()` | Decides which CPU the task should run on |
| Task wakeup - enqueueing | `enqueue()` | Task is enqueued to a BPF-managed run queue |
| CPU has nothing to run | `dispatch()` | Gets a task from the BPF-managed run queue |
| CPU idle state transition | `update_idle()` | Handles state transition from/to idle state of CPU |
| Scheduler tick | `tick()` | Periodically performs maintenance tasks (e.g., power management, preemption) |

When a callback implementation is not provided by the BPF scheduler, the default implementation in [sched/ext.c](https://github.com/torvalds/linux/blob/v6.14/kernel/sched/ext.c) will be executed. You can quickly find the default implementation by searching for `SCX_HAS_OP` which serves as a macro to check if the callback is implemented by the BPF scheduler. 

## scx_userland

In this section, we will introduce the `scx_userland` scheduler, which delegate the scheduling logic to user space. We will learn how to communicate between kernel and user space, and how to implement a BPF scheduler that can be used in user space.

The schedule use 2 maps to communicate between kernel and user space:

- `enqueued`: kernel -> user space: a `scx_userland_enqueued_task` enqueued
- `dispatched`: user space -> kernel: which `pid` to run

The BPF program also maintain a `task_ctx_stor` to store per-task context, which shows how to customize task-specific data for scheduling.

We then dive into the `BPF_STRUCT_OPS`. Here we only show the most interesting parts.

```c
 void BPF_STRUCT_OPS(userland_update_idle, s32 cpu, bool idle)
 {
   if (!idle)
     return;
   /*
    * A CPU is now available, notify the user-space scheduler that tasks
    * can be dispatched, if there is at least one task waiting to be
    * scheduled, either queued (accounted in nr_queued) or scheduled
    * (accounted in nr_scheduled).
    * ...
    */
   if (nr_queued || nr_scheduled) {
     set_usersched_needed();
     scx_bpf_kick_cpu(cpu, 0);
   }
 }
```

`update_idle` is critical in `scx_userland`, we can check its function in [linux/kernel/sched/ext.c](https://github.com/torvalds/linux/blob/v6.14/kernel/sched/ext.c#L450):

```c
/**
	* @update_idle: Update the idle state of a CPU
	* @cpu: CPU to update the idle state for
	* @idle: whether entering or exiting the idle state
	*
	* This operation is called when @rq's CPU goes or leaves the idle
	* state. By default, implementing this operation disables the built-in
	* idle CPU tracking and the following helpers become unavailable:
	*
	* - scx_bpf_select_cpu_dfl()
	* - scx_bpf_test_and_clear_cpu_idle()
	* - scx_bpf_pick_idle_cpu()
	*
	* The user also must implement ops.select_cpu() as the default
	* implementation relies on scx_bpf_select_cpu_dfl().
	*
	* Specify the %SCX_OPS_KEEP_BUILTIN_IDLE flag to keep the built-in idle
	* tracking.
	*/
void (*update_idle)(s32 cpu, bool idle);
```

There are 2 important points here:
+ `bool idle` indicates whether the CPU is entering idle state or not. In `userland_update_idle()`, if `idle` is false, which means the CPU state: idle -> busy, we do nothing. 
+ implementing this operation disables the built-in idle CPU tracking and need to implement `select_cpu()`. 

`userland_update_idle()` gives the user space scheduler a chance to wake up and schedule tasks when the CPU is idle, which makes sense. 

We then look into `userland_dispatch`, which is called when the CPU's local DSQ is empty. 

```c
void BPF_STRUCT_OPS(userland_dispatch, s32 cpu, struct task_struct *prev)
 {
   if (test_and_clear_usersched_needed())
     dispatch_user_scheduler();
 
   bpf_repeat(MAX_ENQUEUED_TASKS) {
     s32 pid;
     struct task_struct *p;
 
     if (bpf_map_pop_elem(&dispatched, &pid))
       break;
 
     p = bpf_task_from_pid(pid);
     if (!p)
       continue;
 
     scx_bpf_dsq_insert(p, SCX_DSQ_GLOBAL, SCX_SLICE_DFL, 0);
     bpf_task_release(p);
   }
 }
```

It consists of 2 actions:
- There exists tasks to be sched --- wake up usersched
- Pop a task from `dispatched` map, insert it into local DSQ.

You can also see the usage of counterpart map `enqueued` in `enqueue_task_in_user_space`, which is called in `userland_enqueue`. 

Other ops are trivial, we skip them and start to look into `scx_userland.c`.

We first check `main()`, which is the entry point of the user space scheduler:

```c
 int main(int argc, char **argv)
 {
     __u64 ecode;
 
     pre_bootstrap(argc, argv);
 restart:
     bootstrap(argv[0]);
     sched_main_loop();
 
     exit_req = 1;
     bpf_link__destroy(ops_link);
     ecode = UEI_REPORT(skel, uei);
     scx_userland__destroy(skel);
 
     if (UEI_ECODE_RESTART(ecode))
         goto restart;
     return 0;
 }
```

`pre_bootstrap()` is used to parse the command line arguments and a page lock function:

```c
     /*
      * It's not always safe to allocate in a user space scheduler, as an
      * enqueued task could hold a lock that we require in order to be able
      * to allocate.
      */
     err = mlockall(MCL_CURRENT | MCL_FUTURE);
     SCX_BUG_ON(err, "Failed to prefault and lock address space");
```

`bootstrap()` is used to load the BPF program and attach it to the kernel. It also get the file descriptor of the `enqueued` and `dispatched` maps. 

Functions below `sched_main_loop()` are post processing functions, we don't care about them for now.

We look into `sched_main_loop()` to see how the user space scheduler works:

```c
static void sched_main_loop(void)
{
    while (!exit_req) {
        drain_enqueued_map();
        dispatch_batch();
        sched_yield();
    }
}
```

Code above shows the main logic of the user space scheduler. It will first drain the `enqueued` map where the kernel enqueues tasks, and then dispatch the tasks to the `dispatched` map, which will be picked up by the kernel. Then it will yield the CPU to allow other tasks to run.

We are most interested in `dispatch_batch()`:

```c
static void dispatch_batch(void)
{
    for (i = 0; i < batch_size; i++) {
        ...
        task = LIST_FIRST(&vruntime_head);

        min_vruntime = task->vruntime;
        pid = task_pid(task);
        LIST_REMOVE(task, entries);
        err = dispatch_task(pid);
        ...
        nr_curr_enqueued--;
    }
    skel->bss->nr_scheduled = nr_curr_enqueued;
}
```

I omit the declaration of variables and error handling code here. The main structure is an ordered list of tasks sorted by their virtual runtime. The `dispatch_batch()` use `LIST_FIRST()` to get the head of the list in each iteration, and then remove it from the list for dispatching.

You can inspect the source code of `scx_userland.c` for more details. Our travel stops here.

## Appendix

We recommend the following resources for further Reading:

### Reference
- [Extensible Scheduler Class](https://docs.kernel.org/scheduler/sched-ext.html)
- [ext.c/sched_ext_ops](https://github.com/torvalds/linux/blob/v6.13/kernel/sched/ext.c#L207)

### Repository
- [scx](https://github.com/sched-ext/scx)

### Blog
- [sched_ext: a BPF-extensible scheduler class (Part 1)](https://blogs.igalia.com/changwoo/sched-ext-a-bpf-extensible-scheduler-class-part-1/)
- [sched_ext: a BPF-extensible scheduler class (Part 2)](https://blogs.igalia.com/changwoo/sched-ext-scheduler-architecture-and-interfaces-part-2/)