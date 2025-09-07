# /proc/slabinfo
```bash
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
```

## 字段解释
### 第一部分：对象统计
+ **name**  
slab 缓存的名字（例如 `dentry`, `inode_cache`, `kmalloc-64` 等）。
+ **active_objs**  
当前正在使用（已分配）的对象数量。
+ **num_objs**  
当前 cache 中的对象总数（包括已分配和未分配的）。
+ **objsize**  
每个对象的大小（字节数）。
+ **objperslab**  
每个 slab 中能容纳的对象数量。  
（slab = 若干页的内存块，里面被切分成 `objperslab` 个对象）。
+ **pagesperslab**  
每个 slab 占用的物理页数。

---

### 第二部分：tunables（可调参数）
这些字段和 `array_cache`（CPU 本地缓存）有关，早期 slab 分配器里比较重要。

+ **limit**  
每个 CPU 的 `array_cache` 能缓存的对象最大数量。  
超过这个值，多余对象会被批量释放回 node 层。
+ **batchcount**  
CPU 缓存和 node/slab 之间批量迁移对象的数量。
    - 当 CPU 本地缓存耗尽，就会从 slab 批量拿 `batchcount` 个对象。
    - 当缓存太满时，就会批量释放 `batchcount` 个对象回 slab。
+ **sharedfactor**  
用于计算 node 级共享缓存（`shared`）大小的因子。
    - 多个 CPU 共享一个 node 的 `shared`，避免频繁访问 slab 链表。
    - 值越大，`shared` 里的对象越多。

---

### 第三部分：slabdata（slab 统计信息）
+ **active_slabs**  
正在使用的 slab 数量（里面至少有一个对象被分配）。
+ **num_slabs**  
总 slab 数量（包括 full、partial、free 三种）。
+ **sharedavail**  
NUMA 节点共享缓存（`shared`）中可用的对象数量。

---

# `kmem_cache`中`cpu_cache`和`node`成员的关系
```c
// include/linux/slab_def.h
struct kmem_cache {
    struct array_cache __percpu *cpu_cache;
    ...
        struct kmem_cache_node *node[MAX_NUMNODES];
};

// mm/slab.h
struct kmem_cache_node {
    struct list_head slabs_partial;	/* partial list first, better asm code */
    struct list_head slabs_full;
    struct list_head slabs_free;
    ...
        struct array_cache *shared;	/* shared per node */
    struct alien_cache **alien;	/* on other nodes */
    ...
};
```



`struct kmem_cache`中的`cpu_cache`和`node`，`struct kmem_cache_node`中的`slabs_partial`，`slabs_full`，`slabs_free`，`shared`，`alien`，这些成员之间的关系：

---

## 1. `struct kmem_cache`（描述某种对象的缓存池）
### `cpu_cache`
+ 类型：`struct array_cache __percpu *`，是 **每 CPU 缓存**。
+ 作用：加速对象的分配和释放，避免频繁访问全局的 slab 链表。
+ 原理：
    - 每个 CPU 有一个私有的 `array_cache`，里面保存若干“热对象”（free 对象的指针数组）。
    - 分配时，CPU 优先从自己的缓存里拿；释放时，也尽量先放到自己缓存里。
    - 如果 `cpu_cache` 空了，就从 `node` 的 slab 获取一批对象补充。
    - 如果 `cpu_cache` 太满，就把一批对象回收到 `node`。

换句话说：**cpu_cache 是 CPU 本地的快速对象池**。

---

### `node[MAX_NUMNODES]`
+ 类型：`struct kmem_cache_node *`
+ 作用：在 NUMA 系统中，不同内存节点（node）都有自己的对象管理器。
+ 每个 `node` 负责本节点分配的 slab。
+ 这样避免跨 NUMA 节点分配/释放对象，提高内存局部性。

---

## 2. `struct kmem_cache_node`（管理一个 NUMA 节点上的 slab）
### `slabs_partial`
+ 链表，保存 **部分分配（partial）** 的 slab：既有已分配对象，也有空闲对象。
+ 分配时，通常会优先从 partial slab 里拿空闲对象。

### `slabs_full`
+ 链表，保存 **已满（full）** 的 slab：里面没有空闲对象。
+ 基本不参与分配，除非有对象被释放回来，slab 可能从 full 转到 partial。

### `slabs_free`
+ 链表，保存 **空 slab**（全空，没有对象被使用）。
+ 释放时，如果某个 slab 所有对象都被释放了，它会被移到 free 列表。
+ 需要回收内存时，可以把 free 列表里的 slab 释放回伙伴系统。

所以：

+ **分配顺序**：partial → free → 新建 slab。
+ **释放时**：对象释放回 slab → slab 状态可能在 full/partial/free 之间切换。

---

### `shared`
+ 类型：`struct array_cache *`
+ 含义：在 NUMA node 级别的 **共享缓存**（介于 cpu_cache 和 slab 链表之间的缓冲层）。
+ 作用：
    - 如果某个 CPU 的 `cpu_cache` 用完/太满，就可以和 `shared` 做批量交换。
    - 减少了频繁访问 slab 链表的开销。

可以理解为：**node 级别的中转站**。

---

### `alien`
+ 类型：`struct alien_cache **`
+ 作用：当某个 CPU 释放了 **不属于自己 NUMA 节点** 的对象时，不能直接放到远端 node 的 `shared`，否则会有锁竞争和跨 NUMA 问题。
+ 解决方案：
    - 先放到本 CPU 上的 `alien_cache`，批量转移到目标节点。
    - 这样避免频繁跨节点操作。

可以理解为：**跨 NUMA 的释放缓冲区**。

---

## 3. 这些成员之间的关系（层级）
如果按层次画出来，大概是这样：

```plain
kmem_cache（对象类型缓存池）
│
├─ cpu_cache (每CPU缓存，加速本地分配/释放)
│     ↓ 批量获取/释放
│
├─ node[MAX_NUMNODES]（NUMA节点缓存）
│    │
│    ├─ slabs_partial（部分分配的slab）
│    ├─ slabs_full   （已满的slab）
│    ├─ slabs_free   （全空的slab）
│    │
│    ├─ shared  （NUMA节点共享缓存，用于CPU间交换）
│    └─ alien   （跨NUMA释放的缓冲区）
```

### 分配流程（简化版）：
1. CPU 先从 `cpu_cache` 取对象；
2. `cpu_cache` 不够时，从 `node.shared` 或 `slabs_partial/free` 批量取；
3. 如果 node 也没有合适 slab，就向伙伴系统申请新页，建 slab。

### 释放流程（简化版）：
1. 对象优先回到 `cpu_cache`；
2. 如果 `cpu_cache` 太满，多余对象批量回到 `node.shared`；
3. 如果 slab 对象全释放，slab 放到 `slabs_free`；
4. 跨 NUMA 的释放，先放到 `alien`，再批量转移。

---

**总结一下**：

+ `cpu_cache` 是 CPU 层的快速对象池；
+ `node` 是 NUMA 节点的 slab 管理者；
+ `slabs_partial/full/free` 管理 slab 状态；
+ `shared` 是 node 内 CPU 之间的中转缓存；
+ `alien` 解决跨 NUMA 节点释放的性能问题。

---

# 测试代码
 	实现一个内核模块：

1）创建名称为`slab_test_cache`的`kmem_cache`大小为40字节，align为8字节。

2）设置构造函数，在构造函数中将对象初始化为`0xAA`。

代码如下：

```c
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/init.h>
#include <linux/delay.h>

static struct kmem_cache *mycache = NULL;
static void *obj = NULL;
static size_t obj_size = 40; // 对象大小

/* 构造函数：在对象分配时自动调用 */
static void my_ctor(void *ptr)
{
    pr_info("mycaches: ctor called for object %p\n", ptr);
    memset(ptr, 0xAA, obj_size);  /* 初始化为 0xAA */
}

static int __init mymodule_init(void)
{
    pr_info("mycaches: module init\n");
    size_t align = 8; // 对齐要求
    const char *name = "slab_test_cache";


    /* 创建 slab cache */
    mycache = kmem_cache_create(name, obj_size, align, SLAB_HWCACHE_ALIGN, &my_ctor);
    if (!mycache) {
        pr_err("mycaches: failed to create slab cache\n");
        return -ENOMEM;
    }

    pr_info("mycaches: slab cache[%s] created, obj_size = %zu, align = %zu\n", name, obj_size, align);

    /* 从 cache 中分配一个对象 */
    obj = kmem_cache_alloc(mycache, GFP_KERNEL);
    if (!obj) {
        pr_err("mycaches: failed to allocate object\n");
        kmem_cache_destroy(mycache);
        return -ENOMEM;
    }

    pr_info("mycaches: allocated object at %p, first 8 bytes = %*ph\n",
            obj, 8, (u8 *)obj);

    return 0;
}

static void __exit mymodule_exit(void)
{
    pr_info("mycaches: module exit\n");

    /* 释放对象 */
    if (obj) {
        kmem_cache_free(mycache, obj);
        pr_info("mycaches: object freed\n");
    }

    /* 销毁 slab cache */
    if (mycache) {
        kmem_cache_destroy(mycache);
        pr_info("mycaches: cache destroyed\n");
    }
}

module_init(mymodule_init);
module_exit(mymodule_exit);


MODULE_LICENSE("GPL");
MODULE_AUTHOR("congchp");
MODULE_DESCRIPTION("Example module for slab cache usage");
```

测试结果如下：

```bash
[25583.310309] mycaches: module init
[25583.310328] mycaches: slab cache[slab_test_cache] created, obj_size = 40, align = 8
[25583.310331] mycaches: ctor called for object 000000005332292f
[25583.310332] mycaches: ctor called for object 0000000009013862
[25583.310333] mycaches: ctor called for object 00000000a6e088c6
[25583.310334] mycaches: ctor called for object 000000003d904c26
[25583.310334] mycaches: ctor called for object 000000007c9fac9b
[25583.310335] mycaches: ctor called for object 000000000098edc2
[25583.310336] mycaches: ctor called for object 00000000ff3fa139
[25583.310336] mycaches: ctor called for object 000000001ce80991
[25583.310337] mycaches: ctor called for object 0000000090d5068a
[25583.310337] mycaches: ctor called for object 0000000057b47b33
[25583.310338] mycaches: ctor called for object 00000000e606fd32
[25583.310338] mycaches: ctor called for object 00000000d113c42c
[25583.310339] mycaches: ctor called for object 000000001f7f816c
[25583.310339] mycaches: ctor called for object 000000008e5b5f55
[25583.310340] mycaches: ctor called for object 0000000074e7df7a
[25583.310340] mycaches: ctor called for object 00000000d4dfa738
[25583.310341] mycaches: ctor called for object 000000001880f040
[25583.310341] mycaches: ctor called for object 0000000029e8e63a
[25583.310342] mycaches: ctor called for object 00000000f88c68dc
[25583.310342] mycaches: ctor called for object 000000001b9f86d1
[25583.310343] mycaches: ctor called for object 00000000dc179a8d
[25583.310343] mycaches: ctor called for object 00000000f93ccce4
[25583.310344] mycaches: ctor called for object 00000000d83ad1da
[25583.310344] mycaches: ctor called for object 000000002b819647
[25583.310345] mycaches: ctor called for object 00000000e4763ffc
[25583.310346] mycaches: ctor called for object 000000001078b356
[25583.310346] mycaches: ctor called for object 00000000dfc333a7
[25583.310347] mycaches: ctor called for object 0000000005ef5f3e
[25583.310347] mycaches: ctor called for object 0000000051a1d56f
[25583.310348] mycaches: ctor called for object 0000000045d96db7
[25583.310348] mycaches: ctor called for object 00000000ba6c2cb0
[25583.310349] mycaches: ctor called for object 00000000b70bef14
[25583.310349] mycaches: ctor called for object 00000000b82e2428
[25583.310350] mycaches: ctor called for object 000000008f41f66b
[25583.310350] mycaches: ctor called for object 00000000b11a4b53
[25583.310351] mycaches: ctor called for object 000000002de03941
[25583.310351] mycaches: ctor called for object 00000000d521c8ce
[25583.310352] mycaches: ctor called for object 000000006842339c
[25583.310353] mycaches: ctor called for object 0000000003bf6fde
[25583.310353] mycaches: ctor called for object 00000000c00f1ad0
[25583.310354] mycaches: ctor called for object 0000000079441e29
[25583.310354] mycaches: ctor called for object 00000000396645ae
[25583.310355] mycaches: ctor called for object 000000007aee9b35
[25583.310355] mycaches: ctor called for object 000000009049f398
[25583.310356] mycaches: ctor called for object 0000000008a5af82
[25583.310356] mycaches: ctor called for object 00000000a2788b07
[25583.310357] mycaches: ctor called for object 00000000318a2f52
[25583.310357] mycaches: ctor called for object 00000000af6b3bcd
[25583.310358] mycaches: ctor called for object 00000000f724e4d2
[25583.310359] mycaches: ctor called for object 0000000097fc2cdf
[25583.310359] mycaches: ctor called for object 000000004440532d
[25583.310360] mycaches: ctor called for object 0000000037932300
[25583.310360] mycaches: ctor called for object 00000000c9c40901
[25583.310361] mycaches: ctor called for object 000000005c3fe42f
[25583.310361] mycaches: ctor called for object 0000000097b2b54f
[25583.310362] mycaches: ctor called for object 000000009d575be3
[25583.310362] mycaches: ctor called for object 000000007763afbb
[25583.310363] mycaches: ctor called for object 0000000002105838
[25583.310363] mycaches: ctor called for object 0000000091a0da01
[25583.310364] mycaches: ctor called for object 000000003c43bcfb
[25583.310364] mycaches: ctor called for object 00000000b3c694a4
[25583.310365] mycaches: ctor called for object 0000000017ede172
[25583.310365] mycaches: ctor called for object 0000000048c6629c
[25583.310366] mycaches: ctor called for object 00000000bb0e4141
[25583.310367] mycaches: allocated object at 000000005332292f, first 8 bytes = aa aa aa aa aa aa aa aa
```



只调用了一次`kmem_cache_alloc`，但为什么会执行64次`my_ctor`？

应该是在第一次调用就将整个slab（一页，4K bytes）中的所有object给初始化了。



可以通过`/sys/kernel/slab/`或者`/proc/slabinfo`查询到该`kmem_cache`的具体信息

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1756804238948-a489b2f2-2650-442c-b6c4-479414781772.png)



![](https://cdn.nlark.com/yuque/0/2025/png/756577/1756806170101-84c7718f-236c-4690-9f8e-6ff2dd033ea2.png)

# 伙伴分配器、slab分配器与用户空间malloc的不同
## 1. 用户空间的 `malloc()`
+ 用户空间调用 `malloc()` 时，glibc 只是向内核申请 **虚拟地址空间**，很多时候并不会立刻分配物理页。
+ 真正的物理内存在 **缺页异常 (page fault)** 时才按需分配（称为 _demand paging_）。

---

## 2. 内核空间的 `alloc_pages()`（伙伴分配器）
+ `alloc_pages(order, gfp_mask)` 直接向伙伴系统要物理页框。
+ 调用返回时，就已经拿到 **真实的物理页**，并且映射到内核的线性映射区。
+ 不会拖延到缺页异常时才分配。

---

## 3. 内核空间的 `kmalloc()`（slab/slub 分配器）
+ `kmalloc(size, gfp_mask)` 本质上最终也是基于伙伴系统分配物理页，然后在 slab 中 carve 成小对象。
+ 调用返回时，拿到的是 **内核虚拟地址**，它已经对应着 **实际分配好的物理内存**。
+ 也就是说，kmalloc 之后马上就能安全访问，不存在 “先占坑、等 page fault 再分配” 的机制。

---

## 4. 为什么内核不能用 _demand paging_？
+ 内核自己必须保证分配时拿到的是可立即使用的物理内存，否则驱动、DMA、页表等都会出问题。
+ 如果像用户空间一样延迟到缺页时才分配，那在处理缺页异常时又需要新的内存，就可能 **递归陷入** 或 **死锁**。

---

## 结论：
+ **用户空间 **`**malloc()**`** → 延迟分配物理内存（缺页异常时才分配）。**
+ **内核空间 **`**alloc_pages()**`** / **`**kmalloc()**`** → 调用时立即分配物理内存，返回的虚拟地址已经有对应物理页。**

---

# <font style="color:rgb(38, 38, 38);">参考资料</font>
1. <font style="color:rgb(64, 64, 64);">Professional Linux Kernel Architecture，Wolfgang Mauerer</font>
2. <font style="color:rgb(64, 64, 64);">Linux内核深度解析，余华兵</font>
3. <font style="color:rgb(64, 64, 64);">Linux设备驱动开发详解，宋宝华</font>
4. <font style="color:rgb(64, 64, 64);">linux kernel 4.12</font>

