# CPU架构，SMP和NUMP
 目前多处理器系统有三种体系结构。

（1）**非一致内存访问**（**Non-Uniform Memory Access，NUMA**）：指内存被划分成多个内存节点的多处理器系统，访问一个内存节点花费的时间取决于处理器和内存节点的距离。每个处理器有一个本地内存节点，处理器访问本地内存节点的速度比访问其他内存节点的速度快。 

（2）**对称多处理器**（**Symmetric Multi-Processor，SMP**）：即一致内存访问（Uniform Memory Access，**UMA**），所有处理器访问内存花费的时间是相同的。每个处理器的地位是平等的，仅在内核初始化的时候不平等：“0号处理器作为引导处理器负责初始化内核，其他处理器等待内核初始化完成。”在实际应用中可以采用混合体系结构，在NUMA节点内部使用SMP体系结构。

（3）MPP(Massive Parallel Processing)，本文不做介绍，后续补充。



下面两个图很好的说明了SMP和NUMA，图片来源于网上。

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1752626851509-0aad1941-6dcf-45be-b1e9-16075201b28b.png)![](https://cdn.nlark.com/yuque/0/2025/png/756577/1752626892606-620f93d1-c0e5-4735-a080-15ee46451541.png)

## 处理器拓扑
 	处理器内部的拓扑如下。

（1）**核（core）**：一个处理器包含多个核，每个核有独立的一级缓存，所有核共享二级缓存。

（2）**硬件线程**：也称为逻辑处理器或者虚拟处理器，一个处理器或者核包含多个硬件线程，硬件线程共享一级缓存和二级缓存。MIPS处理器的叫法是**同步多线程**（Simultaneous Multi-Threading，SMT），英特尔对它的叫法是**超线程**。

---

# SMP调度
 在SMP系统中，进程调度器必须支持以下特性。

（1）需要使每个处理器的**负载**尽可能**均衡**。

（2）可以设置进程的处理器**亲和性**（**affinity**），即允许进程在哪些处理器上执行。

（3）可以把进程从一个处理器**迁移**到另一个处理器。

# 处理器亲和性
在SMP这种多核/多处理器系统中，可以让进程在指定的处理器上运行，提高缓存命中率、减少上下文切换。

 	设置进程的处理器亲和性，就是把进程绑定到某些处理器，只允许进程在某些处理器上执行，默认情况是进程可以在所有处理器上执行。 

通过**处理器亲和性**能够提高缓存命中率、减少上下文切换等，特别适用于实时或性能关键型应用。  

进程描述符以下两个成员说明进程允许在哪些处理器上执行，处理器亲和性最终就是设置这两个成员：

```c
struct task_struct {
    ...
	int				nr_cpus_allowed; // 允许的处理器数量
	cpumask_t			cpus_allowed; // 允许的处理器掩码
    ...
}；
```

---

## CPU亲和性设置方法
---

### 方法一：使用 `taskset` 命令
#### 1. **查看进程当前的CPU亲和性**
```bash
taskset -cp <PID>
# 示例：查看PID为1234的进程
taskset -cp 1234
```

#### 2. **设置已运行进程的CPU亲和性**
```bash
taskset -cp <CPU列表> <PID>
# 示例：将PID 1234绑定到CPU核心0和1上
taskset -cp 0,1 1234
```

#### 3. **启动新进程时直接绑定CPU**
```bash
taskset -c <CPU列表> <命令>
# 示例：启动myapp并绑定到CPU核心0和2
taskset -c 0,2 ./myapp
```

#### **CPU列表格式**：
+ 单核心：`0`
+ 多个核心：`0,2,3`
+ 连续范围：`0-3`（表示核心0~3）

---

### 方法二：使用 `sched_setaffinity` 系统调用
在C/C++程序中直接调用系统函数：

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <sched.h>
#include <unistd.h>

int main() {
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(3, &mask);  // 设置为只在 CPU3 上运行

    if (sched_setaffinity(0, sizeof(mask), &mask) == -1) {
        perror("sched_setaffinity");
        return 1;
    }

    printf("Set CPU affinity to CPU 3 for PID %d\n", getpid());
    while (1);  // 持续运行方便观察
    return 0;
}
```

测试结果：

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753178990765-b2cec7ed-bf8d-4fc1-a9c7-636e7ced65c3.png)

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753179154695-8aa639dc-e03b-4e9b-a4ff-4633a9451ce4.png)

---

### 方法三：线程级别的 CPU 亲缘性设置（`pthread`）
每个线程也可以独立设置 CPU 亲缘性：

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void *thread_func(void *arg) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(1, &cpuset);  // 绑定到 CPU1

    pthread_t thread = pthread_self();
    if (pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset) != 0) {
        perror("pthread_setaffinity_np");
        pthread_exit(NULL);
    }

    printf("Thread bound to CPU 1\n");

    // 执行任务
    while (1) {
        // 可以添加工作负载，比如打印 CPU 编号
        int cpu = sched_getcpu();
        printf("Thread[%d] running on CPU %d\n", gettid(), cpu);
        sleep(10);
    }

    return NULL;
}

int main() {
    printf("Start Process %d\n", getpid());
    pthread_t thread;
    if (pthread_create(&thread, NULL, thread_func, NULL) != 0) {
        perror("pthread_create");
        return 1;
    }

    pthread_join(thread, NULL);
    return 0;
}

```

测试结果：

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753237236388-ecd7cc82-3392-46cc-84f4-fb5646aede35.png)

---

## 进程绑定时的注意事项
| 注意点 | 说明 |
| --- | --- |
| 多核系统 | 核心编号从 0 开始，编号顺序与实际物理核心不一定对应。可用 `lscpu` 查看拓扑 |
| root 权限 | 某些场景设置他人进程的亲缘性需要 root 权限 |
| NUMA 系统 | NUMA 上建议结合 `numactl` 使用，更精细控制内存亲和性 |
| 热插拔 CPU | 如果 CPU 动态插拔（如虚拟机），需要监控 CPU 可用性变化 |


---

## 配合工具
+ `top/htop`：查看进程/线程实时运行在哪个CPU
+ `ps -eo pid,psr,comm`：查看进程当前运行在哪个 CPU 上
+ `numactl --physcpubind=0-3 ./program`：NUMA 控制器

---

# 处理器隔离
有时我们想把一部分处理器作为专用处理器，比如在网络设备上为了提高转发速度，让一 部分处理器专门负责转发报文，实现方法是在引导内核时向内核传递参数`isolcpus=`，**隔离**这些处理器，被隔离的处理器不会参与SMP负载均衡。如果没有把进程绑定到被隔离的处理器，那么不会有进程在被隔离的处理器上执行。 

CPU列表有以下3中格式：

1）<cpu number>，......，<cpu number>

2）按升序排列的范围：<cpu number>-<cpu number>

3）混合格式：<cpu number>，......，<cpu number>-<cpu number>

## 处理器隔离设置方法
```bash
# 编辑 GRUB 配置文件
sudo vim /etc/default/grub

# 在 GRUB_CMDLINE_LINUX 中添加 isolcpus 参数（例如隔离 CPU 2-3）
GRUB_CMDLINE_LINUX="isolcpus=2,3"

# 更新 GRUB 配置
sudo update-grub  # Debian/Ubuntu

# 重启生效
sudo reboot
```

## 测试验证
**测试前提：**

+ 4个CPU(0-3)，隔离CPU2和CPU3。

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753317807281-d3ba60f7-839b-4913-b775-0edb163958a8.png)

**测试内**容：

+ 启动两个线程，`while(1)`运行。
+ 查看CPU0-3的负载情况。

**测试代码**如下：

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sched.h>

// 线程函数 1
void* thread_func1(void* arg) {
    // 获取当前线程 ID 和 CPU 核心
    pid_t tid = gettid();
    int cpu = sched_getcpu();
    printf("线程1 (TID:%d) 在 CPU %d 上运行\n", tid, cpu);

    while(1);

    return NULL;
}

// 线程函数 2
void* thread_func2(void* arg) {
    pid_t tid = gettid();
    int cpu = sched_getcpu();
    printf("线程2 (TID:%d) 在 CPU %d 上运行\n", tid, cpu);
    sleep(1);

    while(1);

    return NULL;
}

int main() {
    pthread_t tid1, tid2;
    
    // 创建线程1
    if(pthread_create(&tid1, NULL, thread_func1, NULL) != 0) {
        perror("创建线程1失败");
        return 1;
    }
    
    // 创建线程2
    if(pthread_create(&tid2, NULL, thread_func2, NULL) != 0) {
        perror("创建线程2失败");
        return 1;
    }
    
    printf("主线程 PID: %d\n", getpid());
    printf("线程1 ID: %lu, 线程2 ID: %lu\n", (unsigned long)tid1, (unsigned long)tid2);
    printf("按 Ctrl+C 退出程序\n");
    
    // 主线程等待子线程结束（实际上永远不会结束）
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    
    return 0;
}
```

**测试结果**：

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753257017479-7e0d258a-18d9-4369-ba6f-c5283296d781.png)

+ 因为CPU2和CPU3被隔离，所以两个线程`while(1)`在CPU0和CPU1上执行。
+ CPU0和CPU1占用率几乎是100%，CPU2和CPU3空闲。



如果将线程绑定到CPU2和CPU3，结果就是CPU2和CPU3占用率100%，CPU0和CPU1空闲。

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753318072754-faadaef6-0bc7-46da-bf46-9ac02d16344f.png)

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753318141030-d56cd3ac-6b0b-42e7-b90d-a94fb0639d31.png)

※top后按1，显示各CPU的负荷情况。

# 调度类对SMP的支持
结构体 `struct sched_class` 是 Linux 内核中**调度器类**的核心数据结构，它定义了调度器必须实现的回调函数接口。每个具体的调度器（如 CFS、实时调度器）都有自己的 `sched_class` 实例。

调度类中对于SMP的支持，主要是CPU**亲和性**和**负载均衡**相关。

---

	以下两种情况下，进程在内存和缓存中的数据是最少的，是**有价值的实现负载均衡的机会**。

（1）调用fork或clone以创建新进程

（2）调用execve装载程序

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1753684632162-2bd51a6a-6f43-4c29-bf1c-5868d5e93c5c.png)

---

**SMP（对称多处理）**相关的部分代码如下：

```c
struct sched_class {
    ...
#ifdef CONFIG_SMP
	int  (*select_task_rq)(struct task_struct *p, int task_cpu, int sd_flag, int flags);
	void (*migrate_task_rq)(struct task_struct *p);

	void (*task_woken) (struct rq *this_rq, struct task_struct *task);

	void (*set_cpus_allowed)(struct task_struct *p,
				 const struct cpumask *newmask);

	void (*rq_online)(struct rq *rq);
	void (*rq_offline)(struct rq *rq);
#endif
    ...
};
```

## 1. `select_task_rq` - 选择运行队列
```c
int (*select_task_rq)(struct task_struct *p, int task_cpu, int sd_flag, int flags);
```

+ **作用**：为任务 `p` **选择**合适的**运行队列，**实际上就是**选择处理器**
+ **参数**：
    - `p`：需要调度的任务（正在唤醒或新创建的任务）
    - `task_cpu`： 建议或默认的 CPU
    - `sd_flag`： 调度域（sched_domain）标志，如 `SD_BALANCE_WAKE`, `SD_BALANCE_EXEC`
    - `flags`：调度行为标志，比如：
        * `WF_TTWU`：任务即将被唤醒
        * `WF_FORK`：新任务
        * `WF_EXEC`：exec 时
+ **返回值**：选中的 CPU 编号
+ **典型实现**：`CFS`（完全公平调度）中会参考负载、亲缘性、能否迁移、负载均衡等条件返回目标 CPU。  

---

## 2. `migrate_task_rq` - 迁移任务
```c
void (*migrate_task_rq)(struct task_struct *p);
```

+ **作用**： 处理任务 `p` 从一个运行队列迁移到另一个运行队列（即跨 CPU 迁移）时的调度类逻辑清理和准备。
+ **典型作用**：
    - 在改变 CPU affinity 或执行负载均衡时，会被调用。
    - 通常用于清理该任务在旧 CPU 上的数据结构（如红黑树中的调度实体），准备插入新 CPU。

---

## 3. `task_woken` - 任务唤醒
```c
void (*task_woken)(struct rq *this_rq, struct task_struct *task);
```

+ **作用**： 在 一个任务被**唤醒（wake-up）**后**，**尚未**入队（enqueue）**前 被调用
    - **唤醒**是指任务从**阻塞状态**如`TASK_INTERRUPTABLE`到`TASK_RUNNING`状态
+ **参数**：
    - `this_rq`：当前 CPU 的运行队列
    - `task`：被唤醒的任务
+ **用途**： 调度器可以借助 `task_woken()` 做一些 **调度策略相关的唤醒处理优化**
    - **判断**被唤醒任务是否**抢占**当前CPU正在运行任务
    - **判断**是否应该跨CPU**迁移**被唤醒的任务。

---

## 4. `set_cpus_allowed` - 设置 CPU 亲和性
```c
void (*set_cpus_allowed)(struct task_struct *p, const struct cpumask *newmask);
```

+ **作用**：设置任务的CPU亲和性
+ **参数**：
    - `p`：目标任务
    - `newmask`：新的 CPU 位掩码
+ **关键操作**：
    1. 检查当前 CPU 是否在新掩码中
    2. 若不在，触发任务迁移

---

## 5. `rq_online/rq_offline` - CPU 热插拔回调
```c
void (*rq_online)(struct rq *rq);  // CPU 上线时调用
void (*rq_offline)(struct rq *rq); // CPU 下线时调用
```

+ **作用**：CPU 热插拔事件通知
+ **典型操作**：
    - 迁移下线 CPU 上的任务
    - 初始化/清理调度器数据结构
    - 更新负载均衡域

---



# SMP负载均衡
以实时调度器说明SMP处理器负载均衡。

## 实时调度类的处理器负载均衡
调度器选择下一个实时进程时，如果当前处理器的实时运行队列中的进程的最高调度优先级比当前正在执行的进程的调度优先级低，将会试图从实时进程**超载**的处理器把**可推送实时进程**拉过来。

实时进程**超载**的定义如下。

（1）实时运行队列至少有两个实时进程。

（2）至少有一个**可推送**实时进程。

 **可推送**实时进程是指绑定到多个处理器的实时进程，可以在处理器之间迁移。  

![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/756577/1753740224966-841ff5ad-4950-4804-95ee-ef34c8a41cee.jpeg)

+ `pick_next_task_rt`，选择下一个实时进程
+ `**need**_pull_rt_task`**，**判断是否需要从其他处理器拉进程
+ `pull_rt_task`，试图从实时进程超载的处理器将可推送实时进程拉过来

```c
static void pull_rt_task(struct rq *this_rq)
{
	int this_cpu = this_rq->cpu, cpu;
	bool resched = false;
	struct task_struct *p;
	struct rq *src_rq;

	if (likely(!rt_overloaded(this_rq)))
		return;

	/*
	 * Match the barrier from rt_set_overloaded; this guarantees that if we
	 * see overloaded we must also see the rto_mask bit.
	 */
	smp_rmb();

#ifdef HAVE_RT_PUSH_IPI
	if (sched_feat(RT_PUSH_IPI)) {
		tell_cpu_to_push(this_rq);
		return;
	}
#endif

	for_each_cpu(cpu, this_rq->rd->rto_mask) {
		if (this_cpu == cpu)
			continue;

		src_rq = cpu_rq(cpu);

		/*
		 * Don't bother taking the src_rq->lock if the next highest
		 * task is known to be lower-priority than our current task.
		 * This may look racy, but if this value is about to go
		 * logically higher, the src_rq will push this task away.
		 * And if its going logically lower, we do not care
		 */
		if (src_rq->rt.highest_prio.next >=
		    this_rq->rt.highest_prio.curr)
			continue;

		/*
		 * We can potentially drop this_rq's lock in
		 * double_lock_balance, and another CPU could
		 * alter this_rq
		 */
		double_lock_balance(this_rq, src_rq);

		/*
		 * We can pull only a task, which is pushable
		 * on its rq, and no others.
		 */
		p = pick_highest_pushable_task(src_rq, this_cpu);

		/*
		 * Do we have an RT task that preempts
		 * the to-be-scheduled task?
		 */
		if (p && (p->prio < this_rq->rt.highest_prio.curr)) {
			WARN_ON(p == src_rq->curr);
			WARN_ON(!task_on_rq_queued(p));

			/*
			 * There's a chance that p is higher in priority
			 * than what's currently running on its cpu.
			 * This is just that p is wakeing up and hasn't
			 * had a chance to schedule. We only pull
			 * p if it is lower in priority than the
			 * current task on the run queue
			 */
			if (p->prio < src_rq->curr->prio)
				goto skip;

			resched = true;

			deactivate_task(src_rq, p, 0);
			set_task_cpu(p, this_cpu);
			activate_task(this_rq, p, 0);
			/*
			 * We continue with the search, just in
			 * case there's an even higher prio task
			 * in another runqueue. (low likelihood
			 * but possible)
			 */
		}
skip:
		double_unlock_balance(this_rq, src_rq);
	}

	if (resched)
		resched_curr(this_rq);
}
```

## 调度域和调度组
	参考文章中的**处理器拓扑**章节**，**当一个进程在不同的处理器拓扑层次上迁移的时候，付出的代价是不同的。 

（1）如果从同一个核的一个硬件线程迁移到另一个硬件线程，进程在一级缓存和二级缓存中的数据可以继续使用。

（2）如果从同一个处理器的一个核迁移到另一个核，进程在源核的一级缓存中的数据失效，在二级缓存中的数据可以继续使用。

（3）如果从同一个NUMA节点的一个处理器迁移到另一个处理器，进程在源处理器的一级缓存和二级缓存中的数据失效。

（4）如果从一个NUMA节点迁移到另一个NUMA节点，进程在源处理器的一级缓存和二级缓存中的数据失效，并且访问内存可能变慢。

---

linux调度器，为了实现 **多处理器之间的负载均衡**，引入了处理器的**调度域**和**调度组**的概念，通过它们实现层次化的负载均衡。使用下图说明调度组和调度域的层级关系：

![](https://cdn.nlark.com/yuque/__mermaid_v3/d31e42592900d27759d08d0eeeb609aa.svg)

### **关键特点：**
1. **层级负载均衡****：**
    - **优先在底层域（SMT/核心级）平衡负载**
    - **无法解决时才向上级域请求**
    - **减少不必要的跨节点迁移**
2. **调度组关系****：**

```c
// include/linux/sched/topology.h
struct sched_domain {
    struct sched_group *groups; /* the balancing groups of the domain */
    // ...
};

// kernel/sched/sched.h
struct sched_group {
	struct sched_group *next;	/* Must be a circular list */
	atomic_t ref;

	unsigned int group_weight;
	struct sched_group_capacity *sgc;
	int asym_prefer_cpu;		/* cpu of highest priority in group */

	/*
	 * The CPUs this group covers.
	 *
	 * NOTE: this field is variable length. (Allocated dynamically
	 * by attaching extra space to the end of the structure,
	 * depending on how many CPUs the kernel has booted up with)
	 */
	unsigned long cpumask[0]; // 柔性数组
};
```

3. **负载均衡流程****：**

![](https://cdn.nlark.com/yuque/__mermaid_v3/6800e6e04a4df17281042f20a20aaf33.svg)

# <font style="color:rgb(38, 38, 38);">参考资料</font>
1. <font style="color:rgb(64, 64, 64);">Professional Linux Kernel Architecture，Wolfgang Mauerer</font>
2. <font style="color:rgb(64, 64, 64);">Linux内核深度解析，余华兵</font>
3. <font style="color:rgb(64, 64, 64);">Linux设备驱动开发详解，宋宝华</font>
4. <font style="color:rgb(64, 64, 64);">linux kernel 4.12</font>



