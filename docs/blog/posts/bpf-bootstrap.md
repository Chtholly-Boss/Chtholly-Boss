---
date:
    created: 2025-03-30
    updated: 2025-03-30
categories:
    - Linux
tags:
    - eBPF
---

# BPF Bootstrap

In this post, we use `bootstrap` from [libbpf-bootstrap examples](https://github.com/libbpf/libbpf-bootstrap/tree/master/examples/c) as an example to introduce some data structures and functions in **eBPF**.

<!-- more -->

## Overview
We plan to achieve the following goals:

- compile and run `bootstrap.bpf.c` and `bootstrap.c` in an independent folder
- master the basic usage of BPF Map, including `HASH`, `RINGBUF`
- understand the usage of some BPF functions

## Compile and Run
### Prerequisites
I list my environment here, other environments may also work, but I cannot guarantee it.

| OS | Kernel | clang | bpftool | libbpf |
|---|-------|--------|--------|---------|
| ArchLinux | 6.13.7 | 21.0 | 7.6.0 | 1.6 |

### Migration
We first create a new folder and copy the relevant files:

```bash
mkdir foo && cd foo
```

Then, copy the `bootstrap.bpf.c`, `bootstrap.h`, `bootstrap.c` from [libbpf-bootstrap examples](https://github.com/libbpf/libbpf-bootstrap/tree/master/examples/c) to the directory.

We are missing `vmlinux.h` in `bootstrap.bpf.c`, which is a file that contains the BTF information of the kernel. We can generate it using the following command:

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

Now we can generate the `bootstrap.skel.h` file. 
```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_x86_64  -c bootstrap.bpf.c -o bootstrap.bpf.o
bpftool gen skeleton bootstrap.bpf.o > bootstrap.skel.h
```

Then we are able to compile `bootstrap.c` and link necessary libraries to form the executable file like:

```bash
clang -Wall -I. -c bootstrap.c -o bootstrap.o
clang -Wall bootstrap.o -L/usr/lib64 -lbpf -lelf -o bootstrap
```

If all goes well, we will get the executable file `bootstrap` in the current folder. Then, you can run `sudo ./bootstrap -d 10` to start the program and observe the output. 

## Code Analysis
Now we start to analyze the code. We begin with `bootstrap.bpf.c` as usual.

Overall, the program registers two `BPF` programs at `sched_process_exec` and `sched_process_exit` to monitor process creation and exit. The program has two core data structures:

- `struct exec_start SEC(".maps)` for recording the process creation time
- `struct rb SEC(".maps")` for recording process creation/exit events and interacting with user-space programs

We will analyze the usage of these two data structures in the program.

---

`struct exec_start` is defined as follows:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 8192);
	__type(key, pid_t);
	__type(value, u64);
} exec_start SEC(".maps");
```

In `handle_exec` function, the BPF program updates the map using `bpf_map_update_elem` function.

```c
pid = bpf_get_current_pid_tgid() >> 32;
ts = bpf_ktime_get_ns();
bpf_map_update_elem(&exec_start, &pid, &ts, BPF_ANY);
```

And in `handle_exit` function, the BPF program looks up the key-value pair using `bpf_map_lookup_elem` function. After processing, it deletes the key-value pair using `bpf_map_delete_elem` function.

```c
start_ts = bpf_map_lookup_elem(&exec_start, &pid);
if (start_ts)
    duration_ns = bpf_ktime_get_ns() - *start_ts;
else if (min_duration_ns)
    return 0;
bpf_map_delete_elem(&exec_start, &pid);
```

So we can summarize the basic usage of `BPF_MAP_TYPE_HASH` type `BPF Map` as follows:

- Create a `BPF Map` using `BPF_MAP_TYPE_HASH` type, specifying the key and value types and maximum number of entries.
- Use `bpf_map_update_elem` to update the key-value pair in the map.
- Use `bpf_map_lookup_elem` to look up the key-value pair in the map.
- Use `bpf_map_delete_elem` to delete the key-value pair from the map.

You can find details about `BPF_MAP_TYPE_HASH` in [bpf_map_type](https://docs.ebpf.io/linux/map-type/BPF_MAP_TYPE_HASH/).

--- 

`struct rb` is defined as follows:

```c
struct {
	__uint(type, BPF_MAP_TYPE_RINGBUF);
	__uint(max_entries, 256 * 1024);
} rb SEC(".maps");
```

I recommend reading [bpf ring buffer post](https://nakryiko.com/posts/bpf-ringbuf/) to understand the `BPF Ring Buffer`. Here we just briefly introduce the usage of it using `handle_exit` as an example.

```c
SEC("tp/sched/sched_process_exit")
int handle_exit(struct trace_event_raw_sched_process_template *ctx)
{
	struct task_struct *task;
	struct event *e;
	pid_t pid, tid;
	u64 id, ts, *start_ts, duration_ns = 0;

    ...

	/* reserve sample from BPF ringbuf */
	e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
	if (!e)
		return 0;

	/* fill out the sample with data */
	task = (struct task_struct *)bpf_get_current_task();

	e->exit_event = true;
	e->duration_ns = duration_ns;
	e->pid = pid;
	e->ppid = BPF_CORE_READ(task, real_parent, tgid);
	e->exit_code = (BPF_CORE_READ(task, exit_code) >> 8) & 0xff;
	bpf_get_current_comm(&e->comm, sizeof(e->comm));

	/* send data to user-space for post-processing */
	bpf_ringbuf_submit(e, 0);
	return 0;
}
```

From the code, we can see that to use `BPF Ring Buffer`, we should:

- reserve a buffer using `bpf_ringbuf_reserve` function
- fill the buffer with data
- submit the buffer using `bpf_ringbuf_submit` function

The data structure used in the buffer is defined in `bootstrap.h` as follows:

```c
#define TASK_COMM_LEN	 16
#define MAX_FILENAME_LEN 127

struct event {
	int pid;
	int ppid;
	unsigned exit_code;
	unsigned long long duration_ns;
	char comm[TASK_COMM_LEN];
	char filename[MAX_FILENAME_LEN];
	bool exit_event;
};
```

---

Now we turn to `bootstrap.c`. Compared to [minimal](./bpf-minimal.md), we need to create a `BPF Ring Buffer` and register an event handler after `bootstrap_bpf__attach`.

```c
rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), handle_event, NULL, NULL);
if (!rb) {
    err = -1;
    fprintf(stderr, "Failed to create ring buffer\n");
    goto cleanup;
}
```

The handler function is defined as follows:

```c
static int handle_event(void *ctx, void *data, size_t data_sz)
{
    ...
    // output event information
	if (e->exit_event) {
		printf("%-8s %-5s %-16s %-7d %-7d [%u]", ts, "EXIT", e->comm, e->pid, e->ppid,
		       e->exit_code);
		if (e->duration_ns)
			printf(" (%llums)", e->duration_ns / 1000000);
		printf("\n");
	} else {
		printf("%-8s %-5s %-16s %-7d %-7d %s\n", ts, "EXEC", e->comm, e->pid, e->ppid,
		       e->filename);
	}

	return 0;
}
```

## Appendix

We can find tracing events in the `/sys/kernel/debug/tracing/events` directory. For example, we can find the `sched` events in `/sys/kernel/debug/tracing/events/sched` like: 

- `sched_stat_runtime`: runtime of a task
- `sched_switch`: context switch between tasks
- `sched_migrate_task`: migration of a task between CPUs
- `sched_move_numa`: NUMA migration of a task
- `sched_wakeup`: wakeup of a task

Try to mount your customed handler to these events. Feel free to explore the usage of `BPF` in the kernel :)
