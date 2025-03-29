---
date:
    created: 2025-03-30
    updated: 2025-03-30
categories:
    - Linux
tags:
    - eBPF
---
# BPF Minimal

In this post, we use `minimal` from [libbpf-bootstrap examples](https://github.com/libbpf/libbpf-bootstrap/tree/master/examples/c) as an example to introduce the basic process of **eBPF** development.

<!-- more -->

## Overview
We plan to achieve the following goals:

- Compile and run `minimal.bpf.c` and `minimal.c` in an independent folder
- Provide a command line argument `-p` for the `minimal` program to specify the monitored program
- learn about `bpf_get_current_pid_tgid()` and `bpf_printk`

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
touch minimal.bpf.c minimal.c
```

Then, copy the content of `minimal.bpf.c`, `minimal.c` from [libbpf-bootstrap examples](https://github.com/libbpf/libbpf-bootstrap/tree/master/examples/c) to the created files.

By now, the statement `#include "minimal.skel.h"` in `minimal.c` is invalid because we have not generated the `minimal.skel.h` file yet. This file can be generated with the following command:

```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_x86_64  -c minimal.bpf.c -o minimal.bpf.o
bpftool gen skeleton minimal.bpf.o > minimal.skel.h
```

Then, we are able to compile `minimal.c` and link necessary libraries to form the executable file like:

```bash
clang -Wall -I. -c minimal.c -o minimal.o
clang -Wall minimal.o -L/usr/lib64 -lbpf -lelf -o minimal
```

If all goes well, we will get the executable file `minimal` in the current folder. Then, you can run `sudo ./minimal` to start the program. In another session, run `sudo cat /sys/kernel/debug/tracing/trace_pipe` to see the output of the program.

## Code Analysis

Now we start to analyze the code. We begin with `minimal.bpf.c`, which is relatively short:

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

Header files are included at the beginning, and the license is defined. The license is important because it determines how the BPF program can be used. Without a license, the kernel may reject to load the BPF program. 

```c
int my_pid = 0;
```

We define a global variable `my_pid` to store the PID of the monitored program. This variable will be stored in the `bss` section of the BPF program. We will see this later.

```c
SEC("tp/syscalls/sys_enter_write")
int handle_tp(void *ctx)
{
	int pid = bpf_get_current_pid_tgid() >> 32;

	if (pid != my_pid)
		return 0;

	bpf_printk("BPF triggered from PID %d.\n", pid);

	return 0;
}
```

Here we come to the main part of the BPF program. First, we define a `tracepoint` for the `sys_write` syscall using `SEC("tp/syscalls/sys_enter_write")`. `SEC` means `section`, you can find details in [libbpf-ebpf-macro-sec](https://docs.ebpf.io/ebpf-library/libbpf/ebpf/SEC/#libbpf-ebpf-macro-sec). The `handle_tp` function will be called when the `sys_write` syscall is triggered.

When `handle_tp` is called, we first get the PID of the current process using `bpf_get_current_pid_tgid()`. This function returns a 64-bit value, where the upper 32 bits represent the PID and the lower 32 bits represent the TID. We shift right by 32 bits to get the PID.

Common `bpf_get_current_pid_tgid()` usage is like:

```c
u64 pid_tgid = bpf_get_current_pid_tgid();
u32 pid = pid_tgid >> 32;
u32 tid = (u32)pid_tgid;
```

We use `pid` to filter the process we want to monitor, then use `bpf_printk` to print some debug information to `/sys/kernel/debug/tracing/trace_pipe`, which is a special file that allows us to see the output of the BPF program in real-time.

Then we check if the PID matches `my_pid`. If it does, we use `bpf_printk` to print a message to `/sys/kernel/debug/tracing/trace_pipe`. This is a special file that allows us to see the output of the BPF program in real-time.

Finally, we return 0 to indicate that the function executed successfully.

So far so good, we compile the progrom and use `bpftool` to generate the skeleton file `minimal.skel.h` for it.

```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_x86_64  -c minimal.bpf.c -o minimal.bpf.o
bpftool gen skeleton minimal.bpf.o > minimal.skel.h
```

We check the main struct in `minimal.skel.h`. 

```c
struct minimal_bpf {
	struct bpf_object_skeleton *skeleton;
	struct bpf_object *obj;
	struct {
		struct bpf_map *bss;
		struct bpf_map *rodata;
	} maps;
	struct {
		struct bpf_program *handle_tp;
	} progs;
	struct {
		struct bpf_link *handle_tp;
	} links;
	struct minimal_bpf__bss {
		int my_pid;
	} *bss;
    ...
};
```

You can see that the `minimal_bpf` struct contains a pointer to the BPF program, which will be used the open, load, and attach the BPF program in user space. As we define a global variable `my_pid` in the BPF program, we can see a `bss` map in the `maps` field, which contains `my_pid` in `minimal_bpf__bss` struct.

We now switch to `minimal.c`, which is the user space program. I extract the relevant code for loading and attaching the BPF program in a `bootstrap` function:

```c
void bootstrap(__pid_t pid)
{
	struct minimal_bpf *skel;
	/* Open BPF application */
	skel = minimal_bpf__open();
	if (!skel) {
		fprintf(stderr, "Failed to open BPF skeleton\n");
		exit(1);
	}
	/* ensure BPF program only handles write() syscalls from our process */
	skel->bss->my_pid = pid;
	int err = minimal_bpf__load(skel);
	if (err) {
		fprintf(stderr, "Failed to load and verify BPF skeleton\n");
		goto cleanup;
	}
	err = minimal_bpf__attach(skel);
	if (err) {
		fprintf(stderr, "Failed to attach BPF skeleton\n");
		goto cleanup;
	}
	return;
cleanup:
	minimal_bpf__destroy(skel);
	exit(err);
}
```

This procedure is relatively fixed, i.e., `open -> init -> load -> attach`.

## Customization

Now let's add a command line argument `-p` to the `minimal` program to specify the PID of the monitored program. We can use `getopt` to parse command line arguments.

```c
void argparse(int argc, char **argv)
{
	int opt;
	while ((opt = getopt(argc, argv, "p:")) != -1) {
		switch (opt) {
		case 'p':
			pid = atoi(optarg);
			break;
		default:
			fprintf(stderr, "Usage: %s [-p pid]\n", argv[0]);
			exit(1);
		}
	}
}
```

modify the `main` function like:
```c
int main(int argc, char **argv)
{
	/* Set up libbpf errors and debug info callback */
	libbpf_set_print(libbpf_print_fn);

	pid = getpid();

	argparse(argc, argv);

	bootstrap(pid);

	printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
	       "to see output of the BPF programs.\n");

	for (;;) {
		// Keep the process running
	}
	return 0;
}
```

Compile the modified `minimal.c` and `minimal.bpf.c` files again. Now we can write a separate program to test the monitoring functionality. 

For example, we write a simple program `test.c` to write to a file:

```c
#include <stdio.h>
#include <unistd.h>

int main(){
	while (1){
		fprintf(stdout, ".");
		sleep(1);
	}
}
```

Compile the program use `clang -o test test.c` and run it in a separate terminal. Use `ps aux | grep test` to get the PID of it. Then run the `minimal` program with the PID in another session:

```bash
sudo ./minimal -p <pid>
```

Then you can see the output of the `minimal` program in the terminal where you run `cat /sys/kernel/debug/tracing/trace_pipe`.
