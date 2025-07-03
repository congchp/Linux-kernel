<font style="color:rgb(64, 64, 64);">在Linux内核中，虽然已经有多种互斥机制（如</font>`<font style="color:rgb(64, 64, 64);">mutex</font>`<font style="color:rgb(64, 64, 64);">、</font>`<font style="color:rgb(64, 64, 64);">spinlock</font>`<font style="color:rgb(64, 64, 64);">、原子操作、</font>`<font style="color:rgb(64, 64, 64);">rwlock</font>`<font style="color:rgb(64, 64, 64);">等），但这些机制在某些场景下可能会成为性能瓶颈。</font>**<font style="color:rgb(64, 64, 64);">RCU（Read-Copy-Update）</font>**<font style="color:rgb(64, 64, 64);"> 是一种特殊的同步机制，主要用于解决</font>**<font style="color:rgb(64, 64, 64);">读多写少</font>**<font style="color:rgb(64, 64, 64);">的场景下的性能问题。RCU对于读多写少场景，性能比</font>`<font style="color:rgb(64, 64, 64);">rwsem</font>`<font style="color:rgb(64, 64, 64);">、</font>`<font style="color:rgb(64, 64, 64);">rwlock</font>`<font style="color:rgb(64, 64, 64);">等都要高。它的设计目标是最大限度地减少读操作的开销，同时保证数据的一致性。</font>

---

# <font style="color:rgb(64, 64, 64);">为什么需要RCU？</font>
## <font style="color:rgb(64, 64, 64);">传统互斥机制的问题</font>
+ **<font style="color:rgb(64, 64, 64);">锁的开销</font>**<font style="color:rgb(64, 64, 64);">：传统的锁机制（如</font>`<font style="color:rgb(64, 64, 64);">mutex</font>`<font style="color:rgb(64, 64, 64);">、</font>`<font style="color:rgb(64, 64, 64);">spinlock</font>`<font style="color:rgb(64, 64, 64);">）在读写冲突时会导致性能下降，尤其是在读多写少的场景中。</font>
    - <font style="color:rgb(64, 64, 64);">读操作需要加锁，即使没有写操作，也会引入额外的开销。</font>
    - <font style="color:rgb(64, 64, 64);">写操作需要阻塞所有读操作，导致读操作的延迟增加。</font>
    - <font style="color:rgb(64, 64, 64);">即使是读写锁，读者也是有开销的。</font>
+ **<font style="color:rgb(64, 64, 64);">扩展性问题</font>**<font style="color:rgb(64, 64, 64);">：在高并发场景下，锁的争用会显著降低系统的可扩展性。</font>

## <font style="color:rgb(64, 64, 64);">RCU的优势</font>
+ **<font style="color:rgb(64, 64, 64);">无锁读操作</font>**<font style="color:rgb(64, 64, 64);">：RCU允许读操作不加锁，因此读操作的开销非常低。</font>
+ **<font style="color:rgb(64, 64, 64);">写操作延迟更新</font>**<font style="color:rgb(64, 64, 64);">：写操作通过复制和延迟更新的方式，避免阻塞读操作。</font>
+ **<font style="color:rgb(64, 64, 64);">高并发性</font>**<font style="color:rgb(64, 64, 64);">：RCU特别适合读多写少的场景，能够显著提高系统的并发性能。</font>

---

# <font style="color:rgb(64, 64, 64);">RCU的原理</font>
**<font style="color:rgb(64, 64, 64);">RCU（Read-Copy-Update），</font>**<font style="color:rgb(64, 64, 64);">读-复制-更新，它是根据原理命名的。写者修改对象的过程是：首先复制生成一个副本，然后更新这个副本，最后使用新的对象替换旧的对象。在写者执行复制更新的时候读者可以读数据。</font>

<font style="color:rgb(64, 64, 64);">它</font><font style="color:rgb(64, 64, 64);">的核心思想是通过</font>**<font style="color:rgb(64, 64, 64);">延迟回收</font>**<font style="color:rgb(64, 64, 64);">和</font>**<font style="color:rgb(64, 64, 64);">无锁读</font>**<font style="color:rgb(64, 64, 64);">来实现高效的并发访问。其基本原理如下：</font>

## <font style="color:rgb(64, 64, 64);">读操作</font>
+ <font style="color:rgb(64, 64, 64);">读操作不需要加锁，直接访问共享数据。</font>
+ <font style="color:rgb(64, 64, 64);">读操作可以并发执行，不会被写操作阻塞。</font>

## <font style="color:rgb(64, 64, 64);">写操作</font>
+ <font style="color:rgb(64, 64, 64);">写操作首先创建一个数据的副本，并在副本上进行修改。</font>
+ <font style="color:rgb(64, 64, 64);">修改完成后，通过原子操作将指针指向新的副本，替换旧数据。</font>
+ <font style="color:rgb(64, 64, 64);">旧的数据不会立即删除，而是等待所有正在使用旧数据的读操作完成后，再回收旧数据。</font>
+ <font style="color:rgb(64, 64, 64);">如果有多个写者，多个写者直接需要使用互斥机制。</font>

## <font style="color:rgb(64, 64, 64);">延迟回收</font>
+ <font style="color:rgb(64, 64, 64);">RCU通过</font>**<font style="color:rgb(64, 64, 64);">宽限期</font>**<font style="color:rgb(64, 64, 64);">（Grace Period）来确保所有读操作完成后再回收旧数据。</font>
+ <font style="color:rgb(64, 64, 64);">写者等待所有读者访问结束的时间称为</font>**<font style="color:rgb(64, 64, 64);">宽限期</font>**<font style="color:rgb(64, 64, 64);">（Grace Period）</font>
+ <font style="color:rgb(64, 64, 64);">宽限期的管理由RCU机制自动完成，开发者无需关心。</font>

---

RCU的**关键技术**是怎么判断所有读者已经完成访问，机制比较复杂（使用per-cpu等），此处不进行展开。

## RCU的约束
RCU对使用者的一些约束：

+ 对共享资源的访问在大部分时间应该是只读的，写访问应该相对很少。
+ 在RCU保护的代码范围内，内核**不能进入睡眠状态**。
+ 受保护资源必须通过**指针**访问。

# <font style="color:rgb(64, 64, 64);">RCU的API</font>
<font style="color:rgb(64, 64, 64);">Linux内核提供了丰富的RCU API，以下是一些常用的函数：</font>

## <font style="color:rgb(64, 64, 64);">读操作</font>
+ `<font style="color:rgb(64, 64, 64);">rcu_read_lock()</font>`<font style="color:rgb(64, 64, 64);">：标记读操作的开始。</font>
+ `<font style="color:rgb(64, 64, 64);">rcu_read_unlock()</font>`<font style="color:rgb(64, 64, 64);">：标记读操作的结束。</font>

```c
// 读者访问受保护数据
rcu_read_lock();
p = rcu_dereference(gp);
// 访问*p
rcu_read_unlock();
```

## <font style="color:rgb(64, 64, 64);">写操作</font>
+ `<font style="color:rgb(64, 64, 64);">rcu_assign_pointer()</font>`<font style="color:rgb(64, 64, 64);">：更新指针，指向新的数据。</font>
+ `<font style="color:rgb(64, 64, 64);">synchronize_rcu()</font>`<font style="color:rgb(64, 64, 64);">：</font>**<font style="color:rgb(64, 64, 64);">阻塞</font>**<font style="color:rgb(64, 64, 64);">等待宽限期结束，确保所有读操作完成，之后释放旧数据。</font>
+ `<font style="color:rgb(64, 64, 64);">call_rcu()</font>`<font style="color:rgb(64, 64, 64);">：非阻塞方式，注册回调，宽限期结束后回调释放旧数据。</font>

```c
// 写者更新数据
old = gp;
new = kmalloc(...);
rcu_assign_pointer(gp, new);
synchronize_rcu(); // 或者使用call_rcu的方式
kfree(old);
```

## 测试代码
### 使用`synchronize_rcu`
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/rcupdate.h>
#include <linux/slab.h>
#include <linux/delay.h>
#include <linux/kthread.h>
#include <linux/err.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("congchp");
MODULE_DESCRIPTION("RCU Test Module with synchronize_rcu()");

// 受RCU保护的数据结构
struct rcu_test_data
{
    int value;
};

static struct rcu_test_data __rcu *global_data;
static struct task_struct *reader_thread;
static struct task_struct *writer_thread;
static int stop_threads;

// 读者线程函数
static int rcu_reader_thread(void *arg)
{
    struct rcu_test_data *data;
    int counter = 0;

    while (!kthread_should_stop() && !stop_threads)
    {
        rcu_read_lock();
        data = rcu_dereference(global_data);
        if (data)
            printk(KERN_INFO "Reader[%d]: read value = %d\n", counter, data->value);
        else
            printk(KERN_INFO "Reader[%d]: data is NULL\n", counter);
        rcu_read_unlock();

        msleep(500); // 每500ms读取一次
        counter++;
    }
    return 0;
}

// 写者线程函数
static int rcu_writer_thread(void *arg)
{
    struct rcu_test_data *new_data, *old_data;
    int value = 0;

    while (!kthread_should_stop() && !stop_threads)
    {
        // 创建新数据
        new_data = kmalloc(sizeof(*new_data), GFP_KERNEL);
        if (!new_data)
        {
            printk(KERN_ERR "Failed to allocate new data\n");
            return -ENOMEM;
        }
        new_data->value = value++;

        // 替换全局指针
        old_data = rcu_dereference_protected(global_data,
                                             lockdep_is_held(&global_data));
        rcu_assign_pointer(global_data, new_data);

        // 使用synchronize_rcu等待所有读者完成
        printk(KERN_INFO "Writer: updating value to %d (waiting for readers...)\n", new_data->value);
        synchronize_rcu();
        printk(KERN_INFO "Writer: grace period passed, safe to free old data\n");

        // 释放旧数据
        if (old_data)
        {
            printk(KERN_INFO "Writer: freeing old data (value=%d)\n", old_data->value);
            kfree(old_data);
        }

        msleep(2000); // 每2秒更新一次
    }
    return 0;
}

// 模块初始化
static int __init rcu_test_init(void)
{
    printk(KERN_INFO "RCU Test Module loaded\n");

    // 初始化全局数据
    global_data = kmalloc(sizeof(*global_data), GFP_KERNEL);
    if (!global_data)
        return -ENOMEM;

    global_data->value = 0;
    rcu_assign_pointer(global_data, global_data);

    // 创建读者线程
    reader_thread = kthread_run(rcu_reader_thread, NULL, "rcu_reader");
    if (IS_ERR(reader_thread))
    {
        printk(KERN_ERR "Failed to create reader thread\n");
        kfree(global_data);
        return PTR_ERR(reader_thread);
    }

    // 创建写者线程
    writer_thread = kthread_run(rcu_writer_thread, NULL, "rcu_writer");
    if (IS_ERR(writer_thread))
    {
        printk(KERN_ERR "Failed to create writer thread\n");
        kthread_stop(reader_thread);
        kfree(global_data);
        return PTR_ERR(writer_thread);
    }

    return 0;
}

// 模块退出
static void __exit rcu_test_exit(void)
{
    struct rcu_test_data *data;

    printk(KERN_INFO "Stopping RCU test module\n");
    stop_threads = 1;

    // 停止线程
    if (reader_thread)
        kthread_stop(reader_thread);
    if (writer_thread)
        kthread_stop(writer_thread);

    // 清理全局数据
    data = rcu_dereference_protected(global_data, 1);
    if (data)
    {
        rcu_assign_pointer(global_data, NULL);
        synchronize_rcu();
        kfree(data);
    }

    printk(KERN_INFO "RCU Test Module unloaded\n");
}

module_init(rcu_test_init);
module_exit(rcu_test_exit);
```

### 使用`call_rcu`
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/rcupdate.h>
#include <linux/slab.h>
#include <linux/delay.h>
#include <linux/kthread.h>
#include <linux/sched.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("congchp");
MODULE_DESCRIPTION("Basic RCU example using call_rcu");

struct my_data
{
    int value;
    struct rcu_head rcu;
};

static struct my_data __rcu *global_ptr = NULL;
static struct task_struct *reader_task;
static struct task_struct *writer_task;

static void rcu_free_callback(struct rcu_head *rcu)
{
    struct my_data *old = container_of(rcu, struct my_data, rcu);
    pr_info("RCU callback: freeing value = %d\n", old->value);
    kfree(old);
}

static int reader_fn(void *data)
{
    while (!kthread_should_stop())
    {
        struct my_data *p;

        rcu_read_lock();
        p = rcu_dereference(global_ptr);
        if (p)
            pr_info("Reader: value = %d\n", p->value);
        rcu_read_unlock();

        msleep(500);
    }
    return 0;
}

static int writer_fn(void *data)
{
    int val = 100;

    while (!kthread_should_stop())
    {
        struct my_data *new = kmalloc(sizeof(*new), GFP_KERNEL);
        struct my_data *old;

        if (!new)
            continue;

        new->value = val++;

        old = rcu_dereference(global_ptr);
        rcu_assign_pointer(global_ptr, new);

        if (old)
            call_rcu(&old->rcu, rcu_free_callback);
        pr_info("Writer: updated value to %d\n", new->value);

        msleep(2000);
    }
    return 0;
}

static int __init rcu_demo_init(void)
{
    pr_info("RCU demo module loaded\n");

    writer_task = kthread_run(writer_fn, NULL, "rcu_writer");
    reader_task = kthread_run(reader_fn, NULL, "rcu_reader");

    return 0;
}

static void __exit rcu_demo_exit(void)
{
    kthread_stop(writer_task);
    kthread_stop(reader_task);

    synchronize_rcu();

    if (global_ptr)
    {
        struct my_data *p = rcu_dereference(global_ptr);
        kfree(p);
    }

    pr_info("RCU demo module unloaded\n");
}

module_init(rcu_demo_init);
module_exit(rcu_demo_exit);

```

---

linux内核中RCU主要用户链表访问的保护，对于链表操作有一套专门的RCU方式的API。

# <font style="color:rgb(64, 64, 64);">用户空间可以使用RCU吗？</font>
<font style="color:rgb(64, 64, 64);">RCU最初是为内核空间设计的，但其思想也可以应用于用户空间。用户空间有一些库和工具可以实现类似内核RCU的功能：</font>

## <font style="color:rgb(64, 64, 64);">用户空间RCU库（Userspace RCU, liburcu）</font>
+ **<font style="color:rgb(64, 64, 64);">l</font>****<font style="color:rgb(64, 64, 64);">iburcu</font>**<font style="color:rgb(64, 64, 64);"> 是一个开源的用户空间RCU实现，提供了与内核RCU类似的API。</font>
+ <font style="color:rgb(64, 64, 64);">它适用于高性能的用户空间应用程序，例如：</font>
    - <font style="color:rgb(64, 64, 64);">多线程服务器。</font>
    - <font style="color:rgb(64, 64, 64);">高性能数据库。</font>
    - <font style="color:rgb(64, 64, 64);">并发数据结构。</font>

## <font style="color:rgb(64, 64, 64);">liburcu的API</font>
+ `<font style="color:rgb(64, 64, 64);">rcu_read_lock()</font>`<font style="color:rgb(64, 64, 64);"> 和 </font>`<font style="color:rgb(64, 64, 64);">rcu_read_unlock()</font>`<font style="color:rgb(64, 64, 64);">：标记读操作的开始和结束。</font>
+ `<font style="color:rgb(64, 64, 64);">rcu_assign_pointer()</font>`<font style="color:rgb(64, 64, 64);"> </font><font style="color:rgb(64, 64, 64);">和</font><font style="color:rgb(64, 64, 64);"> </font>`<font style="color:rgb(64, 64, 64);">rcu_dereference()</font>`<font style="color:rgb(64, 64, 64);">：更新和访问指针。</font>
+ `<font style="color:rgb(64, 64, 64);">synchronize_rcu()</font>`<font style="color:rgb(64, 64, 64);">：等待宽限期结束。</font>

---

## RCU与rwlock，rwsem比较
RCU，rwlock及rwsem，三者都是内核中用于读写并发控制的机制。 但它们各自的设计目标和使用场景有显著区别。  

| **<font style="color:rgb(64, 64, 64);">特性</font>** | **<font style="color:rgb(64, 64, 64);">RCU</font>** | **<font style="color:rgb(64, 64, 64);">读写锁(rwlock)</font>** | **<font style="color:rgb(64, 64, 64);">读写信号量(rwsem)</font>** |
| --- | --- | --- | --- |
| **<font style="color:rgb(64, 64, 64);">读者开销</font>** | <font style="color:rgb(64, 64, 64);">极低(无锁)</font> | <font style="color:rgb(64, 64, 64);">中等(原子操作)</font> | <font style="color:rgb(64, 64, 64);">较高(可能睡眠)</font> |
| **<font style="color:rgb(64, 64, 64);">写者开销</font>** | <font style="color:rgb(64, 64, 64);">很高(宽限期等待)</font> | <font style="color:rgb(64, 64, 64);">高(排他锁)</font> | <font style="color:rgb(64, 64, 64);">高(排他锁+可能睡眠)</font> |
| **<font style="color:rgb(64, 64, 64);">阻塞行为</font>** | <font style="color:rgb(64, 64, 64);">非阻塞(读者/写者)</font> | <font style="color:rgb(64, 64, 64);">非阻塞(自旋等待)</font> | <font style="color:rgb(64, 64, 64);">阻塞(可睡眠)</font> |
| **<font style="color:rgb(64, 64, 64);">内存占用</font>** | <font style="color:rgb(64, 64, 64);">较高(延迟释放)</font> | <font style="color:rgb(64, 64, 64);">低</font> | <font style="color:rgb(64, 64, 64);">低</font> |
| **<font style="color:rgb(64, 64, 64);">优先级反转风险</font>** | <font style="color:rgb(64, 64, 64);">无</font> | <font style="color:rgb(64, 64, 64);">有</font> | <font style="color:rgb(64, 64, 64);">有</font> |
| **<font style="color:rgb(64, 64, 64);">扩展性</font>** | <font style="color:rgb(64, 64, 64);">极佳(随CPU线性扩展)</font> | <font style="color:rgb(64, 64, 64);">差(锁竞争)</font> | <font style="color:rgb(64, 64, 64);">中等</font> |
| **<font style="color:rgb(64, 64, 64);">适用最大读者数</font>** | <font style="color:rgb(64, 64, 64);">理论上无限(无count)</font> | <font style="color:rgb(64, 64, 64);">受计数器(count)限制</font> | <font style="color:rgb(64, 64, 64);">受计数器(count)限制</font> |


# <font style="color:rgb(38, 38, 38);">参考资料</font>
1. <font style="color:rgb(64, 64, 64);">Professional Linux Kernel Architecture，Wolfgang Mauerer</font>
2. <font style="color:rgb(64, 64, 64);">Linux内核深度解析，余华兵</font>
3. <font style="color:rgb(64, 64, 64);">Linux设备驱动开发详解，宋宝华</font>
4. <font style="color:rgb(64, 64, 64);">linux kernel 4.12</font>

