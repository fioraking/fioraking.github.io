---
layout: post
title: Understand ELF Loader
tags: ["Loader","ELF"]
mathjax: true
---


- [目标](#目标)
- [strace](#strace)
- [过程](#过程)
  - [**1️⃣ 用户态发起 execve**](#1️⃣-用户态发起-execve)
  - [**2️⃣ 内核处理 execve**](#2️⃣-内核处理-execve)
  - [**3️⃣ ELF 加载（load\_elf\_binary）**](#3️⃣-elf-加载load_elf_binary)
  - [**4️⃣ 动态加载器（ld-linux.so）**](#4️⃣-动态加载器ld-linuxso)
  - [**5️⃣ 用户程序执行**](#5️⃣-用户程序执行)
  - [**6️⃣ 重定位**](#6️⃣-重定位)
  - [**7️⃣ 流程图**](#7️⃣-流程图)
    - [**总结**](#总结)
- [Segments](#segments)
- [C运行时库](#c运行时库)
  - [1️⃣ CRT 的主要功能](#1️⃣-crt-的主要功能)
  - [2️⃣ 常见的 CRT / libc 实现](#2️⃣-常见的-crt--libc-实现)
  - [3️⃣ 启动部分](#3️⃣-启动部分)
  - [crt\*.o](#crto)
- [参考](#参考)

# 目标
分析Linux C程序ELF文件从磁盘加载到执行的过程。

# strace
```
strace ./a.out // 可以看到粗粒度调用

execve("./a.out", ["./a.out"], 0x7ffe2895f3e0 /* 59 vars */) = 0
brk(NULL)                               = 0x990000
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffe76f08460) = -1 EINVAL (Invalid argument)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f4497b01000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/glibc-hwcaps/x86-64-v3/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/glibc-hwcaps/x86-64-v3", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/glibc-hwcaps/x86-64-v2/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/glibc-hwcaps/x86-64-v2", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/tls/haswell/x86_64/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/tls/haswell/x86_64", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/tls/haswell/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/tls/haswell", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/tls/x86_64/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/tls/x86_64", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/tls/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/tls", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/haswell/x86_64/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/haswell/x86_64", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/haswell/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/haswell", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/x86_64/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64/x86_64", 0x7ffe76f07660) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/cuda-12.4/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/cuda-12.4/lib64", {st_mode=S_IFDIR|0755, st_size=8192, ...}) = 0
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=64987, ...}) = 0
mmap(NULL, 64987, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f4497af1000
close(3)                                = 0
openat(AT_FDCWD, "/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0`\256\3\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=2093744, ...}) = 0
lseek(3, 808, SEEK_SET)                 = 808
read(3, "\4\0\0\0\20\0\0\0\5\0\0\0GNU\0\2\0\0\300\4\0\0\0\3\0\0\0\0\0\0\0", 32) = 32
mmap(NULL, 3954880, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f4497510000
mprotect(0x7f44976cc000, 2097152, PROT_NONE) = 0
mmap(0x7f44978cc000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1bc000) = 0x7f44978cc000
mmap(0x7f44978d2000, 14528, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f44978d2000
close(3)                                = 0
arch_prctl(ARCH_SET_FS, 0x7f4497b02580) = 0
mprotect(0x7f44978cc000, 16384, PROT_READ) = 0
mprotect(0x600000, 4096, PROT_READ)     = 0
mprotect(0x7f4497b03000, 4096, PROT_READ) = 0
munmap(0x7f4497af1000, 64987)           = 0
fstat(1, {st_mode=S_IFREG|0644, st_size=3895, ...}) = 0
brk(NULL)                               = 0x990000
brk(0x9b1000)                           = 0x9b1000
brk(NULL)                               = 0x9b1000
write(1, "1760089382.829363\n", 181760089382.829363
)     = 18
exit_group(0)                           = ?
+++ exited with 0 +++

```

# 过程

## **1️⃣ 用户态发起 execve**

用户程序调用：

```c
execve("./a.out", argv, envp);
```

* 内核参数验证
* 生成 `linux_binprm` 结构体
* 打开可执行文件

---

## **2️⃣ 内核处理 execve**

关键函数：

```c
do_execveat_common()
alloc_bprm()
prepare_binprm()
search_binary_handler()
```

作用：

1. 分配 `linux_binprm`：

   * 保存文件对象、文件头、argv/envp
   * 初始化 mm_struct（新进程地址空间）
2. 读取 ELF 头部，判断二进制类型
3. 根据类型调用对应 loader：

   * ELF → `load_elf_binary()`
   * 脚本 → `load_script()`

---

## **3️⃣ ELF 加载（load_elf_binary）**
可以在内核源码fs/binfmt_elf.c中找到load_elf_binary

步骤：

1. **解析 ELF header 与 program headers**
2. **映射各段到内存**

   * `.text` → 可执行
   * `.data` → 可读写
   * `.bss` → 零初始化
3. **处理动态段 `.dynamic`**

   * 确定所需 `.so` 依赖
   * 记录 lazy binding 信息（PLT/GOT）
4. **初始化用户栈**

   * 将 argc/argv/envp 放入 `bprm->page[]`
   * 设置栈指针 (`bprm->p`)
5. **检查 `.interp` 字段**

   * 如果存在解释器（动态 linker）路径 `/lib64/ld-linux.so`
   * 内核跳转执行 loader

---

## **4️⃣ 动态加载器（ld-linux.so）**

Loader 执行流程：

1. `_dl_start` → `_dl_init`
2. **映射依赖库**

   * 调用 `_dl_map_object_from_fd`
   * mmap 所有共享库
3. **全局符号立即重定位**

   * 处理 `.rela.dyn` / `.rel.dyn`
   * 确保全局变量、静态数据正确绑定
4. **初始化 TLS**

   * `_dl_allocate_tls` 等
5. **初始化 `.init_array`**

   * 调用 C++ 静态构造函数
6. **设置 lazy binding**

   * `.rela.plt` / PLT entry 在第一次调用时解析
   * `_dl_runtime_resolve()` 完成函数地址绑定
7. **准备跳转程序入口**

   * `_start` 函数执行，栈和寄存器已经设置好

---

## **5️⃣ 用户程序执行**

* `_start` 调用 `__libc_start_main(main, argc, argv, init, fini, rtld_fini, stack_end)`
* 执行 `main(argc, argv, envp)`
* 所有依赖库已映射，部分函数通过 lazy bind 延迟解析

---

## **6️⃣ 重定位**

| 类型          | 地址                        | 处理时间                  |
| ----------- | ------------------------- | --------------------- |
| `.rela.dyn` | 全局变量/非函数符号                | loader 初始化 `_dl_init` |
| `.rela.plt` | 函数调用                      | 第一次调用时 lazy bind      |
| 优化          | 可用 `-Wl,-z,now` 让所有函数立即绑定 | 减少 lazy bind 影响       |

---

## **7️⃣ 流程图**

```
execve()                # 用户态系统调用
   │
   ▼
do_execveat_common()    # 内核处理
   │
   ▼
linux_binprm            # 存储 ELF + argv/envp + mm_struct
   │
   ▼
search_binary_handler() # 选择 ELF loader
   │
   ▼
load_elf_binary()       # 内核加载 ELF
   ├─> mmap segments
   ├─> copy argv/envp
   └─> .interp → ld-linux.so
        │
        ▼
     _dl_start/_dl_init
        ├─> mmap 所有共享库
        ├─> relocate .rela.dyn (立即)
        ├─> setup GOT/PLT for lazy bind
        ├─> init .init_array
        └─> jump _start
             │
             ▼
     __libc_start_main(main)
             │
             ▼
          main(argc, argv, envp)
```

---

###  **总结**

1. **linux_binprm** 承载整个 execve 上下文
2. **内核负责 ELF 基本加载、栈准备、切换到 loader**
3. **动态 linker 负责共享库映射、全局符号重定位、lazy bind 设置**
4. **函数调用默认 lazy bind，数据符号立即绑定**
5. **最终 `_start` → libc → main 执行**

---

# Segments

```
llvm-readelf --segments a.out


Elf file type is EXEC (Executable file)
Entry point 0x4004f0
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000400040 0x0000000000400040 0x0001f8 0x0001f8 R   0x8
  INTERP         0x000238 0x0000000000400238 0x0000000000400238 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x0007d8 0x0007d8 R E 0x200000
  LOAD           0x000e00 0x0000000000600e00 0x0000000000600e00 0x00022c 0x000230 RW  0x200000
  DYNAMIC        0x000e10 0x0000000000600e10 0x0000000000600e10 0x0001d0 0x0001d0 RW  0x8
  NOTE           0x000254 0x0000000000400254 0x0000000000400254 0x000044 0x000044 R   0x4
  GNU_EH_FRAME   0x0006b4 0x00000000004006b4 0x00000000004006b4 0x00003c 0x00003c R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x000e00 0x0000000000600e00 0x0000000000600e00 0x000200 0x000200 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .init_array .fini_array .dynamic .got 
   None   .comment .gnu.build.attributes .symtab .strtab .shstrtab

```


# C运行时库
C 运行时库（C runtime library，简称 CRT 或 libc）的 **实现者主要是操作系统/发行版提供商或者开源社区**，它负责在程序执行前、操作系统接口之上提供标准 C 函数和启动环境。下面详细分析：

---

## 1️⃣ CRT 的主要功能

C 运行时库主要分两部分

1. **程序启动/初始化部分**（startup code）

   * 设置堆栈、全局变量、TLS
   * 初始化 `stdin/stdout/stderr`
   * 调用用户的 `main` 函数
   * 处理 `exit()` 和 `_exit()`
   * 在动态链接环境中，调用动态链接器加载共享库

2. **标准库函数实现**（libc API）

   * 字符串操作：`strlen`, `strcpy`
   * 数学函数：`sqrt`, `pow`
   * 内存管理：`malloc`, `free`
   * I/O 函数：`printf`, `fopen`, `read`
   * 系统调用封装：`open`, `read`, `write`, `fork`

---

## 2️⃣ 常见的 CRT / libc 实现

| 名称                        | 特点                              | 维护者 / 来源   |
| ------------------------- | ------------------------------- | ---------- |
| **glibc (GNU C Library)** | 最流行的 Linux libc                 | GNU 项目     |
| **musl**                  | 小巧、静态链接友好，兼容 POSIX              | musl 开发者社区 |
| **uClibc / uClibc-ng**    | 适合嵌入式 Linux                     | uClibc 社区  |
| **BSD libc**              | FreeBSD / NetBSD / OpenBSD 自己实现 | BSD 项目     |
| **libc++abi + libstdc++** | C++ 运行时库，部分函数依赖 libc            | LLVM / GNU |

---

## 3️⃣ 启动部分

以 Linux/glibc 为例：

* **crt0.o / crt1.o / crti.o / crtn.o**

  * 由 glibc 提供
  * 放在链接器默认路径下（`/usr/lib/x86_64-linux-gnu` 等）
  * 作用：

    * `_start` 入口
    * 调用初始化函数 `_init`、`__libc_start_main`
    * 设置 `argc/argv/envp`
  * 典型流程：

    ```text
    _start:
        setup stack
        call __libc_start_main(main, argc, argv, init, fini, rtld_fini)
    ```

* **`__libc_start_main`**

  * 由 glibc 提供
  * 调用 `.init_array`
  * 最终执行用户的 `main()`


## crt*.o

| 文件         | 主要作用                             | 典型位置                        |
| ---------- | -------------------------------- | --------------------------- |
| **crt0.o** | 最早版本启动文件，提供 `_start`             | 古老 UNIX / 教学示例              |
| **crt1.o** | 程序入口 `_start`，初始化 libc、调用 `main` | `/usr/lib/x86_64-linux-gnu` |
| **crti.o** | `.init` 段函数表开头，用于初始化             | `/usr/lib/x86_64-linux-gnu` |
| **crtn.o** | `.fini` 段函数表结尾，用于结束              | `/usr/lib/x86_64-linux-gnu` |

现代 glibc 一般不使用 crt0.o，而是用 crt1.o + crti.o + crtn.o 组合.

# 参考
1. Linux内核源码
2. ChatGPT
3. https://maskray.me/blog/2021-11-07-init-ctors-init-array
