# 文件系统概念
常用的文件系统，包含内核文件系统、用户态文件系统及分布式文件系统等。

## 文件系统（File System）
文件系统是操作系统用于**组织、存储和检索数据**的一种结构和机制。它定义了文件如何存储在磁盘上，并提供文件访问的接口。常见的文件系统包括：

+ **EXT4**（Linux 默认文件系统）
+ **NTFS**（Windows 常用）
+ **FAT32/exFAT**（用于 U 盘、存储设备）
+ **XFS/ZFS**（高性能服务器文件系统）

文件系统通常运行在 **内核态**，直接管理磁盘块和文件的组织结构。

---

## 用户态文件系统（User-space File System）
用户态文件系统指的是**不驻留在内核，而在用户态运行**的文件系统。它通常通过 **FUSE（Filesystem in Userspace）** 技术来实现，允许开发者在 **用户态** 编写文件系统，而无需修改内核。

### 用户态文件系统特点：
+ **无需内核开发**：直接在用户态编写文件系统逻辑 
+ **灵活性高**：可用于模拟数据库、远程存储等文件系统
+ **性能较低**：因需与 FUSE 交互，涉及多次用户态-内核态切换

### 常见的用户态文件系统
+ **SSHFS**：基于 SSH 远程访问文件系统
+ **EncFS**：加密文件系统
+ **S3FS**：将 Amazon S3 云存储映射成本地文件系统

---

## 分布式文件系统（Distributed File System）
分布式文件系统用于**跨多个服务器存储数据**，保证高可用性、冗余和负载均衡。它支持多个客户端同时访问数据，并自动处理数据分布和故障恢复。

### 分布式文件系统特点
+ **多节点存储**：数据存储在多个服务器 
+ **并发访问**：多个客户端共享文件 
+ **容灾机制**：自动备份，防止单点故障

### 常见分布式文件系统
+ **GFS（Google File System）**：谷歌大规模数据存储
+ **FastDFS**：适用于小文件存储
+ **CephFS**：可扩展、高可用文件系统
+ **HDFS（Hadoop Distributed File System）**：大数据存储框架



文件系统，是操作系统用于明确存储设备或者分区上的文件的方法和数据结构。

分布式文件系统,，是指文件系统管理的物理存储资源不一定直接本地节点上，而是通过计算机网络与节点相连。比如 TFS，GFS等。

用户态文件系统（User-space File System）指的是 **运行在用户态** 而 **不直接驻留在内核** 的文件系统，让开发者在用户态编写文件系统逻辑，而无需修改内核代码。  

# 什么是fuse
 FUSE（Filesystem in Userspace）是一种让非特权用户在用户空间实现文件系统的机制。它的核心思想是：**文件系统的逻辑实现不在内核态运行，而是在用户态运行，通过与内核中的 FUSE 模块通信实现文件操作的响应。**

# fuse架构
FUSE 包含两个主要组件：

1. **内核模块（fuse.ko）**  
捕捉来自 VFS（虚拟文件系统）的文件操作请求。
2. **用户空间库（libfuse）和用户FUSE程序（fuse-based FS）**  
用户空间FUSE程序负责真正实现文件系统逻辑（如 ext2、ftpfs、sshfs、s3fs 等）。libfuse是用户程序与内核 FUSE 模块通信的桥梁。

```plain
+-----------------------+
|      应用程序          |  (e.g., cp, vim, cat)
+-----------------------+
           ↓
+-----------------------+
|        VFS            |  (Linux 虚拟文件系统层)
+-----------------------+
           ↓
+-----------------------+
|     FUSE 内核模块      |  (负责与用户态通信)
+-----------------------+
           ↓
+-----------------------+
|      /dev/fuse        |  (字符设备，用于内核⇄用户态通信)
+-----------------------+
           ↓
+-----------------------+
|  用户态 FUSE 程序      |  (开发者实现的文件系统逻辑)
+-----------------------+
```



![](https://cdn.nlark.com/yuque/0/2025/png/756577/1749433537979-5d60b7b7-6d36-42c8-b992-b19b5a7cc47d.png)

# <font style="color:rgb(64, 64, 64);">FUSE 的核心机制</font>
## <font style="color:rgb(64, 64, 64);">(1) 内核 FUSE 模块</font>
+ <font style="color:rgb(64, 64, 64);">内核中的 FUSE 模块负责：</font>
    - <font style="color:rgb(64, 64, 64);">接收来自 VFS（虚拟文件系统）的文件操作请求（如</font><font style="color:rgb(64, 64, 64);"> </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">open</font>**`<font style="color:rgb(64, 64, 64);">、</font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">read</font>**`<font style="color:rgb(64, 64, 64);">、</font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">write</font>**`<font style="color:rgb(64, 64, 64);">）。</font>
    - <font style="color:rgb(64, 64, 64);">将这些请求封装成</font><font style="color:rgb(64, 64, 64);"> </font>**<font style="color:rgb(64, 64, 64);">FUSE 协议消息</font>**<font style="color:rgb(64, 64, 64);">，并通过</font><font style="color:rgb(64, 64, 64);"> </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">/dev/fuse</font>**`<font style="color:rgb(64, 64, 64);"> </font><font style="color:rgb(64, 64, 64);">发送给用户态程序。</font>
    - <font style="color:rgb(64, 64, 64);">接收用户态程序的响应，并返回给 VFS。</font>

## <font style="color:rgb(64, 64, 64);">(2) </font>`<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">/dev/fuse</font>`<font style="color:rgb(64, 64, 64);">（字符设备）</font>
+ `**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">/dev/fuse</font>**`<font style="color:rgb(64, 64, 64);"> </font><font style="color:rgb(64, 64, 64);">是一个</font><font style="color:rgb(64, 64, 64);"> </font>**<font style="color:rgb(64, 64, 64);">字符设备</font>**<font style="color:rgb(64, 64, 64);">（不是块设备），用于内核和用户态之间的通信。</font>
+ <font style="color:rgb(64, 64, 64);">所有文件操作（如</font><font style="color:rgb(64, 64, 64);"> </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">read</font>**`<font style="color:rgb(64, 64, 64);">、</font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">write</font>**`<font style="color:rgb(64, 64, 64);">、</font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">mkdir</font>**`<font style="color:rgb(64, 64, 64);">）都通过它传递。</font>

## <font style="color:rgb(64, 64, 64);">(3) 用户态 FUSE 程序</font>
+ <font style="color:rgb(64, 64, 64);">开发者需要实现一个 </font>**<font style="color:rgb(64, 64, 64);">FUSE 文件系统守护进程</font>**<font style="color:rgb(64, 64, 64);">：</font>
    - <font style="color:rgb(64, 64, 64);">从</font><font style="color:rgb(64, 64, 64);"> </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">/dev/fuse</font>**`<font style="color:rgb(64, 64, 64);"> </font><font style="color:rgb(64, 64, 64);">读取内核发送的请求（如 "读取文件 X 的 4096 字节"）。</font>
    - <font style="color:rgb(64, 64, 64);">执行自定义逻辑（如从网络获取数据、解密文件等）。</font>
    - <font style="color:rgb(64, 64, 64);">将结果写回 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">/dev/fuse</font>**`<font style="color:rgb(64, 64, 64);">，内核再返回给应用程序。</font>
    - `<font style="color:rgb(64, 64, 64);">libfuse</font>`<font style="color:rgb(64, 64, 64);">已经实现帮助用户程序实现了与内核通信的部分，用户程序只需要实现</font>`<font style="color:rgb(64, 64, 64);">struct fuse_operations</font>`<font style="color:rgb(64, 64, 64);">中相应的函数。</font>

# 工作流程
1. **用户挂载一个 FUSE 文件系统**  
执行如 `./fuse_sample /mnt/myfs`，FUSE 内核模块注册文件系统，建立 /dev/fuse 与用户程序之间的通信通道。
2. **用户 触发文件系统操作**  
当用户访问 `/mnt/myfs` 里的文件（如 open、read、write），这些操作首先由 VFS 捕获。
3. **FUSE 内核模块接管请求**  
VFS 将请求转发给 FUSE 内核模块，模块把请求写入 `/dev/fuse` 设备。
4. **用户空间程序读取请求**  
挂载 FUSE 文件系统时启动的用户进程（如`./fuse_sample`）会阻塞式地从 `/dev/fuse` 读取这些请求。
5. **用户程序处理请求**  
它根据请求内容执行相应的逻辑（比如通过网络访问远端文件、或者加密/解密数据等），处理完成后把结果写回 `/dev/fuse`。
6. **内核模块返回结果给 VFS**  
内核模块将用户程序返回的处理结果提交给 VFS，最终返回给调用者（如用户进程）。

# <font style="color:rgb(64, 64, 64);">FUSE 的通信协议</font>
<font style="color:rgb(64, 64, 64);">FUSE 使用一种简单的 </font>**<font style="color:rgb(64, 64, 64);">请求-响应协议</font>**<font style="color:rgb(64, 64, 64);">，消息格式大致如下：</font>

```c
struct fuse_in_header {
    uint32_t opcode;   // 操作类型（如 FUSE_READ, FUSE_WRITE）
    uint64_t unique;    // 唯一标识本次请求
    uint64_t nodeid;   // 文件/目录的 inode
    // ... 其他字段
};

struct fuse_out_header {
    int32_t error;     // 错误码（0 表示成功）
    uint64_t unique;   // 对应请求的 unique 值
    // ... 其他字段
};
```

+ **<font style="color:rgb(64, 64, 64);">请求</font>**<font style="color:rgb(64, 64, 64);">：内核 → 用户态（如</font><font style="color:rgb(64, 64, 64);"> </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">FUSE_READ</font>**`<font style="color:rgb(64, 64, 64);">）。</font>
+ **<font style="color:rgb(64, 64, 64);">响应</font>**<font style="color:rgb(64, 64, 64);">：用户态 → 内核（返回数据或错误码）。</font>

# fuse测试代码
利用`libfuse`，实现一个简单的用户态文件系统，支持读写文件，代码如下：

```c
// Compile with: gcc -Wall fuse_sample.c -o fuse_copilot `pkg-config fuse3 --cflags --libs`

#define FUSE_USE_VERSION 30

#include <fuse3/fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

static const char *test_file = "/test.txt";
static char file_content[1024] = "Hello, FUSE!\n";  // 默认文件内容

// 获取文件属性
static int my_getattr(const char *path, struct stat *stbuf, struct fuse_file_info *fi) {
    (void) fi;
    memset(stbuf, 0, sizeof(struct stat));

    printf("Getattr called for path: %s\n", path);

    if (strcmp(path, "/") == 0) {  // 根目录
        stbuf->st_mode = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
    } else if (strcmp(path, test_file) == 0) {  // test.txt 文件
        stbuf->st_mode = S_IFREG | 0644;
        stbuf->st_nlink = 1;
        stbuf->st_size = strlen(file_content);
    } else {
        return -ENOENT;
    }
    return 0;
}

// 读取目录
static int my_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                      off_t offset, struct fuse_file_info *fi, enum fuse_readdir_flags flags) {
    (void) offset;
    (void) fi;
    (void) flags;

    printf("Readdir called for path: %s\n", path);
    if (strcmp(path, "/") != 0)
        return -ENOENT;

    filler(buf, ".", NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);
    filler(buf, test_file + 1, NULL, 0, 0);  // 添加 test.txt 文件

    return 0;
}

// 读取文件内容
static int my_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    (void) fi;

    printf("Read called for path: %s, offset: %lld, size: %zu\n", path, (long long)offset, size);
    if (strcmp(path, test_file) != 0)
        return -ENOENT;

    size_t len = strlen(file_content);
    if (offset >= len)
        return 0;

    if (size > len - offset)
        size = len - offset;

    memcpy(buf, file_content + offset, size);
    return size;
}

// 写入文件
static int my_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    (void) fi;

    printf("Write called for path: %s, offset: %lld, size: %zu\n", path, (long long)offset, size);
    if (strcmp(path, test_file) != 0)
        return -ENOENT;

    if (offset + size > sizeof(file_content) - 1)
        size = sizeof(file_content) - 1 - offset;

    memcpy(file_content + offset, buf, size);
    file_content[offset + size] = '\0';
    return size;
}

// FUSE 操作集
static struct fuse_operations my_oper = {
    .getattr = my_getattr,
    .readdir = my_readdir,
    .read = my_read,
    .write = my_write,
};

int main(int argc, char *argv[]) {
    return fuse_main(argc, argv, &my_oper, NULL);
}

```

---

测试结果：

![](https://cdn.nlark.com/yuque/0/2025/png/756577/1749450079676-10416aeb-27b3-4aba-ba77-c253673a92fd.png)

---

上面测认代码中，有几点需要说明：

+ 数据最终并没有存入磁盘，而是保存在内存中。是否存入磁盘，由应用程序的处理逻辑决定。
+ 基于`libfuse`实现的用户态文件系统的应用程序，默认是以`deamon`的形式后台运行。如果要在前台执行，运行的时候加上`-f`。
+ 可以通过`umount`卸载该用户态文件系统。
+ 对于`fuse`挂载的一些特殊的选项，使用的时候需要特别注意，比如`<font style="color:rgb(64, 64, 64);">-o direct_io</font>``<font style="color:rgb(64, 64, 64);">-o kernel_cache</font>`<font style="color:rgb(64, 64, 64);">等</font>

# fuse优缺点
## <font style="color:rgb(64, 64, 64);">优点</font>
+ **<font style="color:rgb(64, 64, 64);">开发简单</font>**<font style="color:rgb(64, 64, 64);">：无需编写内核模块，用普通 C/Python/Go 等语言即可实现文件系统。</font>
+ **<font style="color:rgb(64, 64, 64);">安全</font>**<font style="color:rgb(64, 64, 64);">：用户态崩溃不会导致内核 panic。</font>
+ **<font style="color:rgb(64, 64, 64);">灵活</font>**<font style="color:rgb(64, 64, 64);">：可以实现各种特殊用途的文件系统（如加密、压缩、网络存储）。</font>

## <font style="color:rgb(64, 64, 64);">缺点</font>
+ **<font style="color:rgb(64, 64, 64);">性能较低</font>**<font style="color:rgb(64, 64, 64);">：每次文件操作都需要内核⇄用户态切换，比内核文件系统慢。</font>
+ **功能受限**：某些复杂或低层次的文件系统操作不能很好支持（比如 mmap）。  

# fuse应用场景
+ 文件加水印。
+ log网络同步。
+ 文件加密。
+ 打印日志时加时间戳。



