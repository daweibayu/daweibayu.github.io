---
layout: post
title:  "线程的本质（内核层实现）"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---
 <!--more-->

 ### 源码位置

 注意 Android 源码中并不包含 kernel 部分的源码，需要单独下载 [构建内核](https://source.android.com/docs/setup/build/building-kernels) 或者线上 [Common Android Kernel Tree](https://android.googlesource.com/kernel/common/)
 注意版本选择，本文中参考的是 android-gs-bluejay-5.10-android13 版本的源码

### 主题
 书接上回，由 android bionic 层代码  [__bionic_clone.S](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/arch-arm64/bionic/__bionic_clone.S) 知道通过 svc 进入内核态，调用编号为 __NR_clone：
 ```shell
     # Make the system call.
    mov     x8, __NR_clone
    svc     #0
 ```

在 [unistd32.h](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/arch/arm64/include/asm/unistd32.h) 中可以检索到 ：
```shell
#define __NR_clone 120
__SYSCALL(__NR_clone, sys_clone)
```

即映射到内核中的 sys_clone 函数，代码位于[fork.c](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/kernel/fork.c)，clone 的具体实现如下：
```
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 unsigned long, tls,
		 int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#elif defined(CONFIG_CLONE_BACKWARDS3)
SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
		int, stack_size,
		int __user *, parent_tidptr,
		int __user *, child_tidptr,
		unsigned long, tls)
#else
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#endif
{
	struct kernel_clone_args args = {
		.flags		= (lower_32_bits(clone_flags) & ~CSIGNAL),
		.pidfd		= parent_tidptr,
		.child_tid	= child_tidptr,
		.parent_tid	= parent_tidptr,
		.exit_signal	= (lower_32_bits(clone_flags) & CSIGNAL),
		.stack		= newsp,
		.tls		= tls,
	};

	return kernel_clone(&args);
}
#endif
  ```

最终调用 kernel_clone，代码同样位于 fork.c：
```c++
pid_t kernel_clone(struct kernel_clone_args *args)
{
	u64 clone_flags = args->flags;
	struct completion vfork;
	struct pid *pid;
	struct task_struct *p;
	int trace = 0;
	pid_t nr;

        ...

	p = copy_process(NULL, trace, NUMA_NO_NODE, args);
	add_latent_entropy();

	if (IS_ERR(p))
		return PTR_ERR(p);

	cpufreq_task_times_alloc(p);

	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);

	if (clone_flags & CLONE_PARENT_SETTID)
		put_user(nr, args->parent_tid);

	if (clone_flags & CLONE_VFORK) {
		p->vfork_done = &vfork;
		init_completion(&vfork);
		get_task_struct(p);
	}

	if (IS_ENABLED(CONFIG_LRU_GEN) && !(clone_flags & CLONE_VM)) {
		/* lock the task to synchronize with memcg migration */
		task_lock(p);
		lru_gen_add_mm(p->mm);
		task_unlock(p);
	}

	wake_up_new_task(p);

	/* forking complete and child started to run, tell ptracer */
	if (unlikely(trace))
		ptrace_event_pid(trace, pid);

	if (clone_flags & CLONE_VFORK) {
		if (!wait_for_vfork_done(p, &vfork))
			ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
	}

	put_pid(pid);
	return nr;
}
```


```c++
static __latent_entropy struct task_struct *copy_process(
					struct pid *pid,
					int trace,
					int node,
					struct kernel_clone_args *args)
{
	int pidfd = -1, retval;
	struct task_struct *p;
	struct file *pidfile = NULL;

        ...
	retval = copy_thread(p, args);
	...
}
```

copy_thread 的实现
```c++
int copy_thread(struct task_struct *p, const struct kernel_clone_args *args)
{
	unsigned long clone_flags = args->flags;
	unsigned long stack_start = args->stack;
	unsigned long tls = args->tls;
	struct pt_regs *childregs = task_pt_regs(p);

	memset(&p->thread.cpu_context, 0, sizeof(struct cpu_context));

	/*
	 * In case p was allocated the same task_struct pointer as some
	 * other recently-exited task, make sure p is disassociated from
	 * any cpu that may have run that now-exited task recently.
	 * Otherwise we could erroneously skip reloading the FPSIMD
	 * registers for p.
	 */
	fpsimd_flush_task_state(p);

	ptrauth_thread_init_kernel(p);

	if (likely(!args->fn)) {
		*childregs = *current_pt_regs();
		childregs->regs[0] = 0;

		/*
		 * Read the current TLS pointer from tpidr_el0 as it may be
		 * out-of-sync with the saved value.
		 */
		*task_user_tls(p) = read_sysreg(tpidr_el0);
		if (system_supports_tpidr2())
			p->thread.tpidr2_el0 = read_sysreg_s(SYS_TPIDR2_EL0);

		if (stack_start) {
			if (is_compat_thread(task_thread_info(p)))
				childregs->compat_sp = stack_start;
			else
				childregs->sp = stack_start;
		}

		/*
		 * If a TLS pointer was passed to clone, use it for the new
		 * thread.  We also reset TPIDR2 if it's in use.
		 */
		if (clone_flags & CLONE_SETTLS) {
			p->thread.uw.tp_value = tls;
			p->thread.tpidr2_el0 = 0;
		}
	} else {
		/*
		 * A kthread has no context to ERET to, so ensure any buggy
		 * ERET is treated as an illegal exception return.
		 *
		 * When a user task is created from a kthread, childregs will
		 * be initialized by start_thread() or start_compat_thread().
		 */
		memset(childregs, 0, sizeof(struct pt_regs));
		childregs->pstate = PSR_MODE_EL1h | PSR_IL_BIT;

		p->thread.cpu_context.x19 = (unsigned long)args->fn;
		p->thread.cpu_context.x20 = (unsigned long)args->fn_arg;
	}
	p->thread.cpu_context.pc = (unsigned long)ret_from_fork;
	p->thread.cpu_context.sp = (unsigned long)childregs;
	/*
	 * For the benefit of the unwinder, set up childregs->stackframe
	 * as the final frame for the new task.
	 */
	p->thread.cpu_context.fp = (unsigned long)childregs->stackframe;

	ptrace_hw_copy_thread(p);

	return 0;
}
```


未完待续

 
[kthread.c](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/kernel/kthread.c)

[arm64 process.c](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/arch/arm64/kernel/process.c)

[sched.h](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/include/linux/sched.h)

### 总结
art 线程 Thread.cc 通过 pthread 函数库同内核线程建立 1:1 的联系并交互。
那线程的本质是啥？线程的本质就是一个数据结构，在 linux 下就是 task_struct。这么说就好像说人的本质就是大脑，手眼胳膊腿全成了无关紧要之物，事实自然不是如此，就像佛教非要在六识之上搞个阿赖耶识，为了作为轮回的承载主体。
是 cpu 执行线程上下文中的代码，而不是线程执行代码，只不过在用户层很多人感觉是线程在执行逻辑，就像人们感知到的是太阳围着地球转。所有的代码都是死物，只有 cpu 或 gpu 才是拨动命运转盘的手。


