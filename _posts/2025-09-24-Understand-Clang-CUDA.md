---
layout: post
title: Understand Clang CUDA
tags: ["Clang", "CUDA"]
mathjax: true
---

- [Clang \& NVCC](#clang--nvcc)
  - [背景](#背景)
  - [Clang如何编译cuda程序](#clang如何编译cuda程序)
  - [NVCC如何编译cuda程序](#nvcc如何编译cuda程序)
  - [区别](#区别)
  - [Cuda程序如何执行](#cuda程序如何执行)
  - [CUDA Toolkit](#cuda-toolkit)
  - [LLVM PTX backend](#llvm-ptx-backend)
  - [PTX分析](#ptx分析)
  - [参考](#参考)
  - [附录](#附录)
    - [CUDA源码](#cuda源码)
    - [PTX汇编](#ptx汇编)
    - [SASS](#sass)
    - [Fatbinary](#fatbinary)
    - [Host Object Code](#host-object-code)
    - [CUDA JIT](#cuda-jit)


# Clang & NVCC
本文档探索以下：
1.	探索clang & nvcc编译cuda程序的流程以及区别
2.	cuda程序执行流程
3.	CUDA toolkit分析
4.	LLVM PTX backend
5.  PTX分析

## 背景
1.	Cuda 版本
    +	CUDA Tookit工具链版本，可以用gcc 11/12这种版本理解，决定API/库版本，以及支持的compute capability。
2.	Ptx版本
    +	对应CUDA Toolkit支持的指令集，前向兼容
    +	可以理解为LLVM IR/Java Bitcode version
3.	SASS版本
    +	GPU硬件的二进制指令，绑定到SM 架构号(sm_70)
4.	Compute capability（SM version）
    + 每代GPU架构定义的版本号，类似Neoverse-N1,N2这种
    + 硬件指令(SAAS)Nvidia不开源，所以是直接绑定到了SM version上，可以理解为-march=，指明了GPU的硬件架构以及可用的指令集。
    + --cuda-gpu-arch=sm_80

5.	Driver Version
> https://developer.nvidia.com/cuda-gpus
 ToolKit对其做强制要求。


## Clang如何编译cuda程序
https://llvm.org/docs/CompileCudaWithLLVM.html

```
clang++ demo.cu -o demo --cuda-gpu-arch=sm_61 -lcudart_static -ldl -lrt -pthread  -L /usr/local/cuda/lib64
```

背景: Clang Driver: 处理输入参数，生成Jobs, 起子进程执行Job。

```
clang++ main.cu cuda.cu -o demo --cuda-gpu-arch=sm_61 -lcudart_static -ldl -lrt -pthread  -L /usr/local/cuda/lib64/ -###

clang version 20.1.2 (git@gitlab.lecarc.local:armcompiler/xclang.git 58df0ef89dd64126512e4ee27b4ac3fd8ddf6247)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /home/kindles/build/bin
Build config: +unoptimized, +assertions

 "/home/kindles/build/bin/clang-20" "-cc1" "-triple" "nvptx64-nvidia-cuda" "-aux-triple" "x86_64-unknown-linux-gnu" "-S" "-dumpdir" "demo-" "-disable-free" "-clear-ast-before-backend" "-main-file-name" "main.cu" "-mrelocation-model" "static" "-mframe-pointer=all" "-fno-rounding-math" "-no-integrated-as" "-aux-target-cpu" "x86-64" "-fcuda-is-device" "-mllvm" "-enable-memcpyopt-without-libcalls" "-fno-threadsafe-statics" "-fcuda-allow-variadic-functions" "-mlink-builtin-bitcode" "/usr/local/cuda-11.8/nvvm/libdevice/libdevice.10.bc" "-target-sdk-version=11.8" "-target-cpu" "sm_61" "-target-feature" "+ptx78" "-debugger-tuning=gdb" "-fno-dwarf-directory-asm" "-fdebug-compilation-dir=/home/kindles/cuda" "-resource-dir" "/home/kindles/build/lib/clang/20" "-internal-isystem" "/home/kindles/build/lib/clang/20/include/cuda_wrappers" "-include" "__clang_cuda_runtime_wrapper.h" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-internal-isystem" "/usr/local/cuda-11.8/include" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-fdeprecated-macro" "-fno-autolink" "-ferror-limit" "19" "--offload-new-driver" "-pthread" "-fgnuc-version=4.2.1" "-fskip-odr-check-in-gmf" "-fcxx-exceptions" "-fexceptions" "-fcolor-diagnostics" "-cuid=eeaa04177902016" "-D__GCC_HAVE_DWARF2_CFI_ASM=1" "-o" "/tmp/main-sm_61-cf6aa2.s" "-x" "cuda" "main.cu"
 
 "/usr/local/cuda-11.8/bin/ptxas" "-m64" "-O0" "--gpu-name" "sm_61" "--output-file" "/tmp/main-sm_61-bf5478.o" "/tmp/main-sm_61-cf6aa2.s"
 
 "/usr/local/cuda-11.8/bin/fatbinary" "-64" "--create" "/tmp/main-24e3a4.fatbin" "--image=profile=sm_61,file=/tmp/main-sm_61-bf5478.o"
 
 "/home/kindles/build/bin/clang-20" "-cc1" "-triple" "x86_64-unknown-linux-gnu" "-target-sdk-version=11.8" "-fcuda-allow-variadic-functions" "-aux-triple" "nvptx64-nvidia-cuda" "-emit-obj" "-dumpdir" "demo-" "-disable-free" "-clear-ast-before-backend" "-main-file-name" "main.cu" "-mrelocation-model" "pic" "-pic-level" "2" "-pic-is-pie" "-mframe-pointer=all" "-fmath-errno" "-ffp-contract=on" "-fno-rounding-math" "-mconstructor-aliases" "-funwind-tables=2" "-target-cpu" "x86-64" "-tune-cpu" "generic" "-debugger-tuning=gdb" "-fdebug-compilation-dir=/home/kindles/cuda" "-fcoverage-compilation-dir=/home/kindles/cuda" "-resource-dir" "/home/kindles/build/lib/clang/20" "-internal-isystem" "/home/kindles/build/lib/clang/20/include/cuda_wrappers" "-include" "__clang_cuda_runtime_wrapper.h" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-internal-isystem" "/usr/local/cuda-11.8/include" "-fdeprecated-macro" "-ferror-limit" "19" "--offload-new-driver" "-pthread" "-fgnuc-version=4.2.1" "-fskip-odr-check-in-gmf" "-fcxx-exceptions" "-fexceptions" "-fcolor-diagnostics" "-fcuda-include-gpubinary" "/tmp/main-24e3a4.fatbin" "-cuid=eeaa04177902016" "-faddrsig" "-D__GCC_HAVE_DWARF2_CFI_ASM=1" "-o" "/tmp/main-c1d440.o" "-x" "cuda" "main.cu"
 
 "/home/kindles/build/bin/clang-20" "-cc1" "-triple" "nvptx64-nvidia-cuda" "-aux-triple" "x86_64-unknown-linux-gnu" "-S" "-dumpdir" "demo-" "-disable-free" "-clear-ast-before-backend" "-main-file-name" "cuda.cu" "-mrelocation-model" "static" "-mframe-pointer=all" "-fno-rounding-math" "-no-integrated-as" "-aux-target-cpu" "x86-64" "-fcuda-is-device" "-mllvm" "-enable-memcpyopt-without-libcalls" "-fno-threadsafe-statics" "-fcuda-allow-variadic-functions" "-mlink-builtin-bitcode" "/usr/local/cuda-11.8/nvvm/libdevice/libdevice.10.bc" "-target-sdk-version=11.8" "-target-cpu" "sm_61" "-target-feature" "+ptx78" "-debugger-tuning=gdb" "-fno-dwarf-directory-asm" "-fdebug-compilation-dir=/home/kindles/cuda" "-resource-dir" "/home/kindles/build/lib/clang/20" "-internal-isystem" "/home/kindles/build/lib/clang/20/include/cuda_wrappers" "-include" "__clang_cuda_runtime_wrapper.h" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-internal-isystem" "/usr/local/cuda-11.8/include" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-fdeprecated-macro" "-fno-autolink" "-ferror-limit" "19" "--offload-new-driver" "-pthread" "-fgnuc-version=4.2.1" "-fskip-odr-check-in-gmf" "-fcxx-exceptions" "-fexceptions" "-fcolor-diagnostics" "-cuid=b7a5c2548887d1a7" "-D__GCC_HAVE_DWARF2_CFI_ASM=1" "-o" "/tmp/cuda-sm_61-b09510.s" "-x" "cuda" "cuda.cu"
 
 "/usr/local/cuda-11.8/bin/ptxas" "-m64" "-O0" "--gpu-name" "sm_61" "--output-file" "/tmp/cuda-sm_61-1df779.o" "/tmp/cuda-sm_61-b09510.s"
 
 "/usr/local/cuda-11.8/bin/fatbinary" "-64" "--create" "/tmp/cuda-938e7f.fatbin" "--image=profile=sm_61,file=/tmp/cuda-sm_61-1df779.o"
 
 "/home/kindles/build/bin/clang-20" "-cc1" "-triple" "x86_64-unknown-linux-gnu" "-target-sdk-version=11.8" "-fcuda-allow-variadic-functions" "-aux-triple" "nvptx64-nvidia-cuda" "-emit-obj" "-dumpdir" "demo-" "-disable-free" "-clear-ast-before-backend" "-main-file-name" "cuda.cu" "-mrelocation-model" "pic" "-pic-level" "2" "-pic-is-pie" "-mframe-pointer=all" "-fmath-errno" "-ffp-contract=on" "-fno-rounding-math" "-mconstructor-aliases" "-funwind-tables=2" "-target-cpu" "x86-64" "-tune-cpu" "generic" "-debugger-tuning=gdb" "-fdebug-compilation-dir=/home/kindles/cuda" "-fcoverage-compilation-dir=/home/kindles/cuda" "-resource-dir" "/home/kindles/build/lib/clang/20" "-internal-isystem" "/home/kindles/build/lib/clang/20/include/cuda_wrappers" "-include" "__clang_cuda_runtime_wrapper.h" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/x86_64-redhat-linux" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../include/c++/12/backward" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-internal-isystem" "/home/kindles/build/lib/clang/20/include" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../x86_64-redhat-linux/include" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-internal-isystem" "/usr/local/cuda-11.8/include" "-fdeprecated-macro" "-ferror-limit" "19" "--offload-new-driver" "-pthread" "-fgnuc-version=4.2.1" "-fskip-odr-check-in-gmf" "-fcxx-exceptions" "-fexceptions" "-fcolor-diagnostics" "-fcuda-include-gpubinary" "/tmp/cuda-938e7f.fatbin" "-cuid=b7a5c2548887d1a7" "-faddrsig" "-D__GCC_HAVE_DWARF2_CFI_ASM=1" "-o" "/tmp/cuda-7935de.o" "-x" "cuda" "cuda.cu"
 
 "/home/kindles/build/bin/clang-linker-wrapper" "--cuda-path=/usr/local/cuda-11.8" "--host-triple=x86_64-unknown-linux-gnu" "--linker-path=/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../bin/ld" "--hash-style=gnu" "--eh-frame-hdr" "-m" "elf_x86_64" "-pie" "-dynamic-linker" "/lib64/ld-linux-x86-64.so.2" "-o" "demo" "/lib/../lib64/Scrt1.o" "/lib/../lib64/crti.o" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/crtbeginS.o" "-L/usr/local/cuda/lib64/" "-L/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12" "-L/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/../../../../lib64" "-L/lib/../lib64" "-L/usr/lib/../lib64" "-L/lib" "-L/usr/lib" "/tmp/main-c1d440.o" "/tmp/cuda-7935de.o" "-lcudart_static" "-ldl" "-lrt" "-lstdc++" "-lm" "-lgcc_s" "-lgcc" "-lpthread" "-lc" "-lgcc_s" "-lgcc" "/opt/rh/gcc-toolset-12/root/usr/lib/gcc/x86_64-redhat-linux/12/crtendS.o" "/lib/../lib64/crtn.o"
```

从上面的Job中可以看到,Clang编译cuda程序分为以下几步:
1. 编译CUDA源码中Device部分至PTX汇编
2. 利用CUDA Toolkit中ptxas汇编器将第一步中PTX编译为SASS指令
3. 利用CUDA Toolkit中fatbinary将SASS文件和PTX文件合并为fatbinary
4. 编译CUDA源码中Host部分至Native code,并且将fatbinary合共至Native Code
5. 最后链接多个fatbin,然后host链接并输出可执行文件（都在链接器中完成）

## NVCC如何编译cuda程序
```
nvcc -v main.cu add.cu mul.cu -o main

#$ _NVVM_BRANCH_=nvvm
#$ _NVVM_BRANCH_SUFFIX_=
#$ _SPACE_= 
#$ _CUDART_=cudart
#$ _HERE_=/usr/local/cuda-12.4/bin
#$ _THERE_=/usr/local/cuda-12.4/bin
#$ _TARGET_SIZE_=
#$ _TARGET_DIR_=
#$ _TARGET_DIR_=targets/x86_64-linux
#$ TOP=/usr/local/cuda-12.4/bin/..
#$ NVVMIR_LIBRARY_DIR=/usr/local/cuda-12.4/bin/../nvvm/libdevice
#$ LD_LIBRARY_PATH=/usr/local/cuda-12.4/bin/../lib:/usr/local/cuda-12.4/lib64:/usr/local/cuda-12.4/lib64:/usr/local/cuda-12.4/lib64
#$ PATH=/usr/local/cuda-12.4/bin/../nvvm/bin:/usr/local/cuda-12.4/bin:/home/kindles/build/bin/:/root/build/bin/:/usr/local/cuda-12.4/bin:/root/miniconda3/bin:/root/.vscode-server/cli/servers/Stable-384ff7382de624fb94dbaf6da11977bba1ecd427/server/bin/remote-cli:/usr/local/cuda-12.4/bin:/root/miniconda3/bin:/usr/share/man:/root/.nvm/versions/node/v22.14.0/bin:/usr/local/cuda-12.4/bin:/root/miniconda3/bin:/usr/share/Modules/bin:/usr/condabin:/usr/lib64/ccache:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
#$ INCLUDES="-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"  
#$ LIBRARIES=  "-L/usr/local/cuda-12.4/bin/../targets/x86_64-linux/lib/stubs" "-L/usr/local/cuda-12.4/bin/../targets/x86_64-linux/lib"
#$ CUDAFE_FLAGS=
#$ PTXAS_FLAGS=
#$ gcc -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -E -x c++ -D__CUDACC__ -D__NVCC__  "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"    -D__CUDACC_VER_MAJOR__=12 -D__CUDACC_VER_MINOR__=4 -D__CUDACC_VER_BUILD__=99 -D__CUDA_API_VER_MAJOR__=12 -D__CUDA_API_VER_MINOR__=4 -D__NVCC_DIAG_PRAGMA_SUPPORT__=1 -include "cuda_runtime.h" -m64 "main.cu" -o "/tmp/tmpxft_00041631_00000000-5_main.cpp4.ii" 
#$ cudafe++ --c++14 --gnu_version=80500 --display_error_number --orig_src_file_name "main.cu" --orig_src_path_name "/home/kindles/cuda/test/main.cu" --allow_managed  --m64 --parse_templates --gen_c_file_name "/tmp/tmpxft_00041631_00000000-6_main.cudafe1.cpp" --stub_file_name "tmpxft_00041631_00000000-6_main.cudafe1.stub.c" --gen_module_id_file --module_id_file_name "/tmp/tmpxft_00041631_00000000-4_main.module_id" "/tmp/tmpxft_00041631_00000000-5_main.cpp4.ii" 
#$ gcc -D__CUDA_ARCH__=520 -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -E -x c++  -DCUDA_DOUBLE_MATH_FUNCTIONS -D__CUDACC__ -D__NVCC__  "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"    -D__CUDACC_VER_MAJOR__=12 -D__CUDACC_VER_MINOR__=4 -D__CUDACC_VER_BUILD__=99 -D__CUDA_API_VER_MAJOR__=12 -D__CUDA_API_VER_MINOR__=4 -D__NVCC_DIAG_PRAGMA_SUPPORT__=1 -include "cuda_runtime.h" -m64 "main.cu" -o "/tmp/tmpxft_00041631_00000000-17_main.cpp1.ii" 
#$ cicc --c++14 --gnu_version=80500 --display_error_number --orig_src_file_name "main.cu" --orig_src_path_name "/home/kindles/cuda/test/main.cu" --allow_managed   -arch compute_52 -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name "tmpxft_00041631_00000000-3_main.fatbin.c" -tused --module_id_file_name "/tmp/tmpxft_00041631_00000000-4_main.module_id" --gen_c_file_name "/tmp/tmpxft_00041631_00000000-6_main.cudafe1.c" --stub_file_name "/tmp/tmpxft_00041631_00000000-6_main.cudafe1.stub.c" --gen_device_file_name "/tmp/tmpxft_00041631_00000000-6_main.cudafe1.gpu"  "/tmp/tmpxft_00041631_00000000-17_main.cpp1.ii" -o "/tmp/tmpxft_00041631_00000000-6_main.ptx"
#$ ptxas -arch=sm_52 -m64  "/tmp/tmpxft_00041631_00000000-6_main.ptx"  -o "/tmp/tmpxft_00041631_00000000-18_main.sm_52.cubin" 
#$ fatbinary -64 --cicc-cmdline="-ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 " "--image3=kind=elf,sm=52,file=/tmp/tmpxft_00041631_00000000-18_main.sm_52.cubin" "--image3=kind=ptx,sm=52,file=/tmp/tmpxft_00041631_00000000-6_main.ptx" --embedded-fatbin="/tmp/tmpxft_00041631_00000000-3_main.fatbin.c" 
#$ rm /tmp/tmpxft_00041631_00000000-3_main.fatbin
#$ gcc -D__CUDA_ARCH__=520 -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -c -x c++  -DCUDA_DOUBLE_MATH_FUNCTIONS -Wno-psabi "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"   -m64 "/tmp/tmpxft_00041631_00000000-6_main.cudafe1.cpp" -o "/tmp/tmpxft_00041631_00000000-19_main.o" 
#$ gcc -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -E -x c++ -D__CUDACC__ -D__NVCC__  "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"    -D__CUDACC_VER_MAJOR__=12 -D__CUDACC_VER_MINOR__=4 -D__CUDACC_VER_BUILD__=99 -D__CUDA_API_VER_MAJOR__=12 -D__CUDA_API_VER_MINOR__=4 -D__NVCC_DIAG_PRAGMA_SUPPORT__=1 -include "cuda_runtime.h" -m64 "add.cu" -o "/tmp/tmpxft_00041631_00000000-9_add.cpp4.ii" 
#$ cudafe++ --c++14 --gnu_version=80500 --display_error_number --orig_src_file_name "add.cu" --orig_src_path_name "/home/kindles/cuda/test/add.cu" --allow_managed  --m64 --parse_templates --gen_c_file_name "/tmp/tmpxft_00041631_00000000-10_add.cudafe1.cpp" --stub_file_name "tmpxft_00041631_00000000-10_add.cudafe1.stub.c" --gen_module_id_file --module_id_file_name "/tmp/tmpxft_00041631_00000000-8_add.module_id" "/tmp/tmpxft_00041631_00000000-9_add.cpp4.ii" 
#$ gcc -D__CUDA_ARCH__=520 -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -E -x c++  -DCUDA_DOUBLE_MATH_FUNCTIONS -D__CUDACC__ -D__NVCC__  "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"    -D__CUDACC_VER_MAJOR__=12 -D__CUDACC_VER_MINOR__=4 -D__CUDACC_VER_BUILD__=99 -D__CUDA_API_VER_MAJOR__=12 -D__CUDA_API_VER_MINOR__=4 -D__NVCC_DIAG_PRAGMA_SUPPORT__=1 -include "cuda_runtime.h" -m64 "add.cu" -o "/tmp/tmpxft_00041631_00000000-20_add.cpp1.ii" 
#$ cicc --c++14 --gnu_version=80500 --display_error_number --orig_src_file_name "add.cu" --orig_src_path_name "/home/kindles/cuda/test/add.cu" --allow_managed   -arch compute_52 -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name "tmpxft_00041631_00000000-7_add.fatbin.c" -tused --module_id_file_name "/tmp/tmpxft_00041631_00000000-8_add.module_id" --gen_c_file_name "/tmp/tmpxft_00041631_00000000-10_add.cudafe1.c" --stub_file_name "/tmp/tmpxft_00041631_00000000-10_add.cudafe1.stub.c" --gen_device_file_name "/tmp/tmpxft_00041631_00000000-10_add.cudafe1.gpu"  "/tmp/tmpxft_00041631_00000000-20_add.cpp1.ii" -o "/tmp/tmpxft_00041631_00000000-10_add.ptx"
#$ ptxas -arch=sm_52 -m64  "/tmp/tmpxft_00041631_00000000-10_add.ptx"  -o "/tmp/tmpxft_00041631_00000000-21_add.sm_52.cubin" 
#$ fatbinary -64 --cicc-cmdline="-ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 " "--image3=kind=elf,sm=52,file=/tmp/tmpxft_00041631_00000000-21_add.sm_52.cubin" "--image3=kind=ptx,sm=52,file=/tmp/tmpxft_00041631_00000000-10_add.ptx" --embedded-fatbin="/tmp/tmpxft_00041631_00000000-7_add.fatbin.c" 
#$ rm /tmp/tmpxft_00041631_00000000-7_add.fatbin
#$ gcc -D__CUDA_ARCH__=520 -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -c -x c++  -DCUDA_DOUBLE_MATH_FUNCTIONS -Wno-psabi "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"   -m64 "/tmp/tmpxft_00041631_00000000-10_add.cudafe1.cpp" -o "/tmp/tmpxft_00041631_00000000-22_add.o" 
#$ gcc -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -E -x c++ -D__CUDACC__ -D__NVCC__  "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"    -D__CUDACC_VER_MAJOR__=12 -D__CUDACC_VER_MINOR__=4 -D__CUDACC_VER_BUILD__=99 -D__CUDA_API_VER_MAJOR__=12 -D__CUDA_API_VER_MINOR__=4 -D__NVCC_DIAG_PRAGMA_SUPPORT__=1 -include "cuda_runtime.h" -m64 "mul.cu" -o "/tmp/tmpxft_00041631_00000000-13_mul.cpp4.ii" 
#$ cudafe++ --c++14 --gnu_version=80500 --display_error_number --orig_src_file_name "mul.cu" --orig_src_path_name "/home/kindles/cuda/test/mul.cu" --allow_managed  --m64 --parse_templates --gen_c_file_name "/tmp/tmpxft_00041631_00000000-14_mul.cudafe1.cpp" --stub_file_name "tmpxft_00041631_00000000-14_mul.cudafe1.stub.c" --gen_module_id_file --module_id_file_name "/tmp/tmpxft_00041631_00000000-12_mul.module_id" "/tmp/tmpxft_00041631_00000000-13_mul.cpp4.ii" 
#$ gcc -D__CUDA_ARCH__=520 -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -E -x c++  -DCUDA_DOUBLE_MATH_FUNCTIONS -D__CUDACC__ -D__NVCC__  "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"    -D__CUDACC_VER_MAJOR__=12 -D__CUDACC_VER_MINOR__=4 -D__CUDACC_VER_BUILD__=99 -D__CUDA_API_VER_MAJOR__=12 -D__CUDA_API_VER_MINOR__=4 -D__NVCC_DIAG_PRAGMA_SUPPORT__=1 -include "cuda_runtime.h" -m64 "mul.cu" -o "/tmp/tmpxft_00041631_00000000-23_mul.cpp1.ii" 
#$ cicc --c++14 --gnu_version=80500 --display_error_number --orig_src_file_name "mul.cu" --orig_src_path_name "/home/kindles/cuda/test/mul.cu" --allow_managed   -arch compute_52 -m64 --no-version-ident -ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 --include_file_name "tmpxft_00041631_00000000-11_mul.fatbin.c" -tused --module_id_file_name "/tmp/tmpxft_00041631_00000000-12_mul.module_id" --gen_c_file_name "/tmp/tmpxft_00041631_00000000-14_mul.cudafe1.c" --stub_file_name "/tmp/tmpxft_00041631_00000000-14_mul.cudafe1.stub.c" --gen_device_file_name "/tmp/tmpxft_00041631_00000000-14_mul.cudafe1.gpu"  "/tmp/tmpxft_00041631_00000000-23_mul.cpp1.ii" -o "/tmp/tmpxft_00041631_00000000-14_mul.ptx"
#$ ptxas -arch=sm_52 -m64  "/tmp/tmpxft_00041631_00000000-14_mul.ptx"  -o "/tmp/tmpxft_00041631_00000000-24_mul.sm_52.cubin" 
#$ fatbinary -64 --cicc-cmdline="-ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 " "--image3=kind=elf,sm=52,file=/tmp/tmpxft_00041631_00000000-24_mul.sm_52.cubin" "--image3=kind=ptx,sm=52,file=/tmp/tmpxft_00041631_00000000-14_mul.ptx" --embedded-fatbin="/tmp/tmpxft_00041631_00000000-11_mul.fatbin.c" 
#$ rm /tmp/tmpxft_00041631_00000000-11_mul.fatbin
#$ gcc -D__CUDA_ARCH__=520 -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -c -x c++  -DCUDA_DOUBLE_MATH_FUNCTIONS -Wno-psabi "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"   -m64 "/tmp/tmpxft_00041631_00000000-14_mul.cudafe1.cpp" -o "/tmp/tmpxft_00041631_00000000-25_mul.o" 
#$ nvlink -m64 --arch=sm_52 --register-link-binaries="/tmp/tmpxft_00041631_00000000-15_main_dlink.reg.c"    "-L/usr/local/cuda-12.4/bin/../targets/x86_64-linux/lib/stubs" "-L/usr/local/cuda-12.4/bin/../targets/x86_64-linux/lib" -cpu-arch=X86_64 "/tmp/tmpxft_00041631_00000000-19_main.o" "/tmp/tmpxft_00041631_00000000-22_add.o" "/tmp/tmpxft_00041631_00000000-25_mul.o"  -lcudadevrt  -o "/tmp/tmpxft_00041631_00000000-26_main_dlink.sm_52.cubin" --host-ccbin "gcc"
#$ fatbinary -64 --cicc-cmdline="-ftz=0 -prec_div=1 -prec_sqrt=1 -fmad=1 " -link "--image3=kind=elf,sm=52,file=/tmp/tmpxft_00041631_00000000-26_main_dlink.sm_52.cubin" --embedded-fatbin="/tmp/tmpxft_00041631_00000000-16_main_dlink.fatbin.c" 
#$ rm /tmp/tmpxft_00041631_00000000-16_main_dlink.fatbin
#$ gcc -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -c -x c++ -DFATBINFILE="\"/tmp/tmpxft_00041631_00000000-16_main_dlink.fatbin.c\"" -DREGISTERLINKBINARYFILE="\"/tmp/tmpxft_00041631_00000000-15_main_dlink.reg.c\"" -I. -D__NV_EXTRA_INITIALIZATION= -D__NV_EXTRA_FINALIZATION= -D__CUDA_INCLUDE_COMPILER_INTERNAL_HEADERS__  -Wno-psabi "-I/usr/local/cuda-12.4/bin/../targets/x86_64-linux/include"    -D__CUDACC_VER_MAJOR__=12 -D__CUDACC_VER_MINOR__=4 -D__CUDACC_VER_BUILD__=99 -D__CUDA_API_VER_MAJOR__=12 -D__CUDA_API_VER_MINOR__=4 -D__NVCC_DIAG_PRAGMA_SUPPORT__=1 -m64 "/usr/local/cuda-12.4/bin/crt/link.stub" -o "/tmp/tmpxft_00041631_00000000-27_main_dlink.o" 
#$ g++ -D__CUDA_ARCH_LIST__=520 -D__NV_LEGACY_LAUNCH -m64 -Wl,--start-group "/tmp/tmpxft_00041631_00000000-27_main_dlink.o" "/tmp/tmpxft_00041631_00000000-19_main.o" "/tmp/tmpxft_00041631_00000000-22_add.o" "/tmp/tmpxft_00041631_00000000-25_mul.o"   "-L/usr/local/cuda-12.4/bin/../targets/x86_64-linux/lib/stubs" "-L/usr/local/cuda-12.4/bin/../targets/x86_64-linux/lib"  -lcudadevrt  -lcudart_static  -lrt -lpthread  -ldl  -Wl,--end-group -o "main" 

```

1. 分离Host与Device Code
2. 编译device code生成PTX
3. 编译PTX至SASS
4. fatbin合并PTX与SASS
5. 编译Host code 至native code
6. 编译link.stub至native code
7. nvlink链接所有fatbin,（头文件方式供host include)
8. 用host linker将567链接输出可执行文件
   
## 区别
> https://llvm.org/docs/CompileCudaWithLLVM.html#id12
+ nvcc 的 split：历史产物，为了快速把 CUDA 推向市场，最大化复用现有编译器生态 → 工程 pragmatism。
+ clang 的 merged parsing：为了保持 C++ 语义一致性、工具链现代化、可维护性 → 语言 correctness。

## Cuda程序如何执行
> https://llvm.org/docs/NVPTXUsage.html#running-the-kernel （包含JIT)
 
```
main()
 ├─ cudaMalloc()
 │   └─ libcudart.so → cuMemAlloc() [driver API] → GPU alloc
 │
 ├─ add<<<>>>()
 │   └─ __device_stub__add()
 │       ├─ cudaConfigureCall()
 │       ├─ cudaSetupArgument()
 │       └─ cudaLaunch(kernelFunc)
 │           └─ libcudart.so → cuLaunchKernel() [driver API]
 │               └─ GPU: 执行 kernel (SASS/PTX)
 │
 └─ cudaDeviceSynchronize()
     └─ libcudart.so → cuCtxSynchronize() [driver API]
         └─ GPU: 等待任务完成

```

上面的执行过程可以通过反汇编最终得到:
```
llvm-readelf -h main # 得到入口地址

llvm-objdump -d main >& main.s # 根据入口地址即可静态追踪执行过程

cuda-gdb main # 也可以通过cuda-gdb进行调试追踪

```

JIT,请看附录例子。


## CUDA Toolkit
> https://developer.nvidia.com/cuda-toolkit
> https://docs.nvidia.com/cuda/index.html

分析CUDA的开发包里面都有什么，以及作用。

 The toolkit includes GPU-accelerated libraries, debugging and optimization tools, a C/C++ compiler, and a runtime library.

```
| 目录/文件                                       | 说明                                                             |
| ------------------------------------------- | -------------------------------------------------------------- |
| `bin/`                                      | CUDA 命令行工具，比如 `nvcc`、`cuda-gdb`、`fatbinary`、`ptxas` 等          |
| `lib64/`                                    | 64-bit 静态库和动态库（如 `libcudart.so` stubs）            |
| `include/`                                  | CUDA headers（比如 `cuda_runtime.h`、`device_launch_parameters.h`） |
| `nvvm/`                                     | NVVM IR 和编译器库，支持 PTX 生成和 JIT                                   |
                                                          

```


## LLVM PTX backend
> https://llvm.org/docs/NVPTXUsage.html

+ LLVM IR子集(NVVM)
调试clang编译cuda，

```
#0  addPassesToGenerateCode (TM=..., PM=..., DisableVerify=false, MMIWP=...) at /home/kindles/llvm/llvm/lib/CodeGen/CodeGenTargetMachineImpl.cpp:119
#1  0x00000000070df5f9 in llvm::CodeGenTargetMachineImpl::addPassesToEmitFile (this=0x1148c0b0, PM=..., Out=..., DwoOut=0x0, FileType=llvm::CodeGenFileType::AssemblyFile, DisableVerify=false, MMIWP=0xfafe420)
    at /home/kindles/llvm/llvm/lib/CodeGen/CodeGenTargetMachineImpl.cpp:219
#2  0x000000000936b5fd in (anonymous namespace)::EmitAssemblyHelper::AddEmitPasses (this=0x7fffffff2680, CodeGenPasses=..., Action=clang::Backend_EmitAssembly, OS=..., DwoOS=0x0)
    at /home/kindles/llvm/clang/lib/CodeGen/BackendUtil.cpp:624
#3  0x0000000009364971 in (anonymous namespace)::EmitAssemblyHelper::RunCodegenPipeline (this=0x7fffffff2680, Action=clang::Backend_EmitAssembly, Python Exception <class 'ValueError'> Unsupported implementation for unique_ptr: std::__uniq_ptr_data<llvm::raw_pwrite_stream, std::default_delete<llvm::raw_pwrite_stream>, true, true>: 
OS=..., Python Exception <class 'ValueError'> Unsupported implementation for unique_ptr: std::__uniq_ptr_data<llvm::ToolOutputFile, std::default_delete<llvm::ToolOutputFile>, true, true>: 
DwoOS=...)
    at /home/kindles/llvm/clang/lib/CodeGen/BackendUtil.cpp:1204
#4  0x000000000935e10a in (anonymous namespace)::EmitAssemblyHelper::emitAssembly (this=0x7fffffff2680, Action=clang::Backend_EmitAssembly, Python Exception <class 'ValueError'> Unsupported implementation for unique_ptr: std::__uniq_ptr_data<llvm::raw_pwrite_stream, std::default_delete<llvm::raw_pwrite_stream>, true, true>: 
OS=..., BC=0xf92c4c0)
    at /home/kindles/llvm/clang/lib/CodeGen/BackendUtil.cpp:1252
#5  0x000000000935d5db in clang::emitBackendOutput (CI=..., CGOpts=..., TDesc=..., M=0xf9ccf80, Action=clang::Backend_EmitAssembly, VFS=..., Python Exception <class 'ValueError'> Unsupported implementation for unique_ptr: std::__uniq_ptr_data<llvm::raw_pwrite_stream, std::default_delete<llvm::raw_pwrite_stream>, true, true>: 
OS=..., BC=0xf92c4c0)
    at /home/kindles/llvm/clang/lib/CodeGen/BackendUtil.cpp:1416
#6  0x0000000009cb1d87 in clang::BackendConsumer::HandleTranslationUnit (this=0xf92c4c0, C=...) at /home/kindles/llvm/clang/lib/CodeGen/CodeGenAction.cpp:315
#7  0x000000000caa3198 in clang::ParseAST (S=..., PrintStats=false, SkipFunctionBodies=false) at /home/kindles/llvm/clang/lib/Parse/ParseAST.cpp:184
#8  0x000000000a451e27 in clang::ASTFrontendAction::ExecuteAction (this=0xf93f880) at /home/kindles/llvm/clang/lib/Frontend/FrontendAction.cpp:1186
#9  0x0000000009cb5ce0 in clang::CodeGenAction::ExecuteAction (this=0xf93f880) at /home/kindles/llvm/clang/lib/CodeGen/CodeGenAction.cpp:1101
#10 0x000000000a451826 in clang::FrontendAction::Execute (this=0xf93f880) at /home/kindles/llvm/clang/lib/Frontend/FrontendAction.cpp:1072
#11 0x000000000a36d08e in clang::CompilerInstance::ExecuteAction (this=0xf928ac0, Act=...) at /home/kindles/llvm/clang/lib/Frontend/CompilerInstance.cpp:1056
#12 0x000000000a577dae in clang::ExecuteCompilerInvocation (Clang=0xf928ac0) at /home/kindles/llvm/clang/lib/FrontendTool/ExecuteCompilerInvocation.cpp:296
#13 0x0000000005efe37a in cc1_main (Argv=..., Argv0=0x7fffffffcc3f "/home/kindles/build/bin/clang-20", MainAddr=0x5eeee70 <GetExecutablePath[abi:cxx11](char const*, bool)>)
    at /home/kindles/llvm/clang/tools/driver/cc1_main.cpp:290
#14 0x0000000005ef061a in ExecuteCC1Tool (ArgV=..., ToolContext=...) at /home/kindles/llvm/clang/tools/driver/driver.cpp:218
#15 0x0000000005eef323 in clang_main (Argc=96, Argv=0x7fffffffc5c8, ToolContext=...) at /home/kindles/llvm/clang/tools/driver/driver.cpp:259
#16 0x0000000005f25c05 in main (argc=96, argv=0x7fffffffc5c8) at tools/clang/tools/driver/clang-driver.cpp:17
```

llvm/lib/CodeGen/CodeGenTargetMachineImpl.cpp,
整个NVPTX后端pass的添加。
```
static TargetPassConfig *
addPassesToGenerateCode(CodeGenTargetMachineImpl &TM, PassManagerBase &PM,
                        bool DisableVerify,
                        MachineModuleInfoWrapperPass &MMIWP) {
  // Targets may override createPassConfig to provide a target-specific
  // subclass.
  TargetPassConfig *PassConfig = TM.createPassConfig(PM);
  // Set PassConfig options provided by TargetMachine.
  PassConfig->setDisableVerify(DisableVerify);
  PM.add(PassConfig);
  PM.add(&MMIWP);

  if (PassConfig->addISelPasses())
    return nullptr;
  PassConfig->addMachinePasses();
  PassConfig->setInitialized();
  return PassConfig;
}

```

下面对添加的addISelPasses进行分析。

```
bool TargetPassConfig::addISelPasses() {
  if (TM->useEmulatedTLS())
    addPass(createLowerEmuTLSPass()); // 使用运行时库模拟TLS, 在不支持TLS的硬件上。

  PM->add(createTargetTransformInfoWrapperPass(TM->getTargetIRAnalysis())); // TTI信息
  addPass(createPreISelIntrinsicLoweringPass()); // lower intrinsic to ir
  addPass(createExpandLargeDivRemPass()); // lower除法/求余到一个rt库函数实现。
  addPass(createExpandLargeFpConvertPass()); // lower浮点整数相互转换至rt库函数实现。
  addIRPasses(); // IR级别目标相关的优化pass // NVPTX后端重写,虚函数 // llvm/lib/Target/NVPTX/NVPTXTargetMachine.cpp:addIRPasses()
  addCodeGenPrepare(); // codegen之前做清洗/变形/规范化对IR
  addPassesToHandleExceptions(); // 异常处理pass 
  addISelPrepare(); // 指令选择预处理(expand/lower等)

  return addCoreISelPasses(); // 指令选择pass
}
```

对addMachinePasses分析。
```
void TargetPassConfig::addMachinePasses() {
  AddingMachinePasses = true;

  // Add passes that optimize machine instructions in SSA form.
  if (getOptLevel() != CodeGenOptLevel::None) {
    addMachineSSAOptimization(); // MI上SSA形式优化,NVPTX后端重写// llvm/lib/Target/NVPTX/NVPTXTargetMachine.cpp
  } else {
    // If the target requests it, assign local variables to stack slots relative
    // to one another and simplify frame index references where possible.
    addPass(&LocalStackSlotAllocationID);
  }

  if (TM->Options.EnableIPRA)
    addPass(createRegUsageInfoPropPass());

  // Run pre-ra passes.
  addPreRegAlloc(); // 寄存器分配之前，NVPTX后端重写

  // Debugifying the register allocator passes seems to provoke some
  // non-determinism that affects CodeGen and there doesn't seem to be a point
  // where it becomes safe again so stop debugifying here.
  DebugifyIsSafe = false;

  // Add a FSDiscriminator pass right before RA, so that we could get
  // more precise SampleFDO profile for RA.
  if (EnableFSDiscriminator) {
    addPass(createMIRAddFSDiscriminatorsPass(
        sampleprof::FSDiscriminatorPass::Pass1));
    const std::string ProfileFile = getFSProfileFile(TM);
    if (!ProfileFile.empty() && !DisableRAFSProfileLoader)
      addPass(createMIRProfileLoaderPass(ProfileFile, getFSRemappingFile(TM),
                                         sampleprof::FSDiscriminatorPass::Pass1,
                                         nullptr));
  }

  // Run register allocation and passes that are tightly coupled with it,
  // including phi elimination and scheduling.
  if (getOptimizeRegAlloc())
    addOptimizedRegAlloc(); // 寄存器分配，包括PreRA指令调度，寄存器分配// NVPTX后端重写
  else
    addFastRegAlloc();

  // Run post-ra passes.
  addPostRegAlloc(); // 寄存器分配后优化，NVPTX后端重写，包含序言尾言以及peephole

  addPass(&RemoveRedundantDebugValuesID);

  addPass(&FixupStatepointCallerSavedID);

  // Insert prolog/epilog code.  Eliminate abstract frame index references...
  if (getOptLevel() != CodeGenOptLevel::None) {
    addPass(&PostRAMachineSinkingID);
    addPass(&ShrinkWrapID); // 优化插入位置
  }

  // Prolog/Epilog inserter needs a TargetMachine to instantiate. But only
  // do so if it hasn't been disabled, substituted, or overridden.
  if (!isPassSubstitutedOrOverridden(&PrologEpilogCodeInserterID))
      addPass(createPrologEpilogInserterPass());

  /// Add passes that optimize machine instructions after register allocation.
  if (getOptLevel() != CodeGenOptLevel::None)
      addMachineLateOptimization(); // 优化机器指令

  // Expand pseudo instructions before second scheduling pass.
  addPass(&ExpandPostRAPseudosID);

  // Run pre-sched2 passes.
  addPreSched2(); // 虚函数，do nothing

  if (EnableImplicitNullChecks)
    addPass(&ImplicitNullChecksID);

  // Second pass scheduler.
  // Let Target optionally insert this pass by itself at some other
  // point.
  if (getOptLevel() != CodeGenOptLevel::None &&
      !TM->targetSchedulesPostRAScheduling()) {
    if (MISchedPostRA)
      addPass(&PostMachineSchedulerID);
    else
      addPass(&PostRASchedulerID); // RA后指令调度，without SchedModel
  }

  // GC
  addGCPasses(); // 支持的垃圾自动回收机制

  // Basic block placement.
  if (getOptLevel() != CodeGenOptLevel::None)
    addBlockPlacement(); // BB位置优化

  // Insert before XRay Instrumentation.
  addPass(&FEntryInserterID);

  addPass(&XRayInstrumentationID);
  addPass(&PatchableFunctionID);

  addPreEmitPass(); // Emit MC之前,虚函数，do nothing

  if (TM->Options.EnableIPRA)
    // Collect register usage information and produce a register mask of
    // clobbered registers, to be used to optimize call sites.
    addPass(createRegUsageInfoCollector());

  // FIXME: Some backends are incompatible with running the verifier after
  // addPreEmitPass.  Maybe only pass "false" here for those targets?
  addPass(&FuncletLayoutID); // 处理异常处理bb

  addPass(&RemoveLoadsIntoFakeUsesID);
  addPass(&StackMapLivenessID);
  addPass(&LiveDebugValuesID);
  addPass(&MachineSanitizerBinaryMetadataID);

  if (TM->Options.EnableMachineOutliner &&
      getOptLevel() != CodeGenOptLevel::None &&
      EnableMachineOutliner != RunOutliner::NeverOutline) {
    bool RunOnAllFunctions =
        (EnableMachineOutliner == RunOutliner::AlwaysOutline);
    bool AddOutliner =
        RunOnAllFunctions || TM->Options.SupportsDefaultOutlining;
    if (AddOutliner)
      addPass(createMachineOutlinerPass(RunOnAllFunctions));
  }

  if (GCEmptyBlocks)
    addPass(llvm::createGCEmptyBasicBlocksPass());

  if (EnableFSDiscriminator)
    addPass(createMIRAddFSDiscriminatorsPass(
        sampleprof::FSDiscriminatorPass::PassLast));

  // Machine function splitter uses the basic block sections feature.
  // When used along with `-basic-block-sections=`, the basic-block-sections
  // feature takes precedence. This means functions eligible for
  // basic-block-sections optimizations (`=all`, or `=list=` with function
  // included in the list profile) will get that optimization instead.
  if (TM->Options.EnableMachineFunctionSplitter || //拆分函数
      EnableMachineFunctionSplitter) {
    const std::string ProfileFile = getFSProfileFile(TM);
    if (!ProfileFile.empty()) {
      if (EnableFSDiscriminator) {
        addPass(createMIRProfileLoaderPass(
            ProfileFile, getFSRemappingFile(TM),
            sampleprof::FSDiscriminatorPass::PassLast, nullptr));
      } else {
        // Sample profile is given, but FSDiscriminator is not
        // enabled, this may result in performance regression.
        WithColor::warning()
            << "Using AutoFDO without FSDiscriminator for MFS may regress "
               "performance.\n";
      }
    }
    addPass(createMachineFunctionSplitterPass());
    if (SplitStaticData) // 拆分静态数据
      addPass(createStaticDataSplitterPass());
  }
  // We run the BasicBlockSections pass if either we need BB sections or BB
  // address map (or both).
  if (TM->getBBSectionsType() != llvm::BasicBlockSection::None ||
      TM->Options.BBAddrMap) {
    if (TM->getBBSectionsType() == llvm::BasicBlockSection::List) {
      addPass(llvm::createBasicBlockSectionsProfileReaderWrapperPass(
          TM->getBBSectionsFuncListBuf()));
      addPass(llvm::createBasicBlockPathCloningPass());
    }
    addPass(llvm::createBasicBlockSectionsPass()); // 重新对bb布局
  }

  addPostBBSections(); // do nothing

  if (!DisableCFIFixup && TM->Options.EnableCFIFixup)
    addPass(createCFIFixup());

  PM->add(createStackFrameLayoutAnalysisPass());

  // Add passes that directly emit MI after all other MI passes.
  addPreEmitPass2(); // do nothing

  AddingMachinePasses = false;
}
```



## PTX分析
> https://docs.nvidia.com/cuda/parallel-thread-execution/

PTX, a low-level parallel thread execution virtual machine and instruction set architecture (ISA). 





## 参考
https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#rhel-rocky-installation

https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#compute-capability

https://developer.nvidia.com/cuda-gpus

https://developer.nvidia.com/cuda-toolkit-archive

https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html

https://docs.nvidia.com/cuda/parallel-thread-execution/

https://developer.nvidia.com/cuda-toolkit

https://llvm.org/docs/CompileCudaWithLLVM.html

https://llvm.org/docs/NVPTXUsage.html

## 附录

### CUDA源码
```cuda
#include <iostream>

__global__ void VecAdd(float* A, float* B, float* C, int N) {
    int i = threadIdx.x + blockIdx.x * blockDim.x;
    if (i < N)
        C[i] = A[i] + B[i];
}

int main() {
    int N = 256;
    size_t size = N * sizeof(float);

    float *h_A = new float[N];
    float *h_B = new float[N];
    float *h_C = new float[N];

    for (int i = 0; i < N; ++i) {
        h_A[i] = i;
        h_B[i] = 2 * i;
    }

    float *d_A, *d_B, *d_C;
    cudaMalloc(&d_A, size);
    cudaMalloc(&d_B, size);
    cudaMalloc(&d_C, size);

    cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
    cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice);

    VecAdd<<<(N + 255) / 256, 256>>>(d_A, d_B, d_C, N);
    cudaError_t err = cudaGetLastError();
    printf("Launch: %s\n", cudaGetErrorString(err));
    cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost);

    for (int i = 0; i < 10; ++i)
        std::cout << h_C[i] << std::endl;

    cudaFree(d_A);
    cudaFree(d_B);
    cudaFree(d_C);
    delete[] h_A;
    delete[] h_B;
    delete[] h_C;

    return 0;
}


```
### PTX汇编
编译main.cu至PTX
```ptx
//
// Generated by LLVM NVPTX Back-End
//

.version 7.8
.target sm_61
.address_size 64

```

编译cuda.cu至PTX
```
//
// Generated by LLVM NVPTX Back-End
//

.version 7.8
.target sm_61
.address_size 64

	// .globl	_Z9vectorAddPKfS0_Pfi   // -- Begin function _Z9vectorAddPKfS0_Pfi
.global .align 1 .b8 blockIdx[1];
.global .align 1 .b8 blockDim[1];
.global .align 1 .b8 threadIdx[1];
                                        // @_Z9vectorAddPKfS0_Pfi
.visible .entry _Z9vectorAddPKfS0_Pfi(
	.param .u64 .ptr .align 1 _Z9vectorAddPKfS0_Pfi_param_0,
	.param .u64 .ptr .align 1 _Z9vectorAddPKfS0_Pfi_param_1,
	.param .u64 .ptr .align 1 _Z9vectorAddPKfS0_Pfi_param_2,
	.param .u32 _Z9vectorAddPKfS0_Pfi_param_3
)
{
	.local .align 8 .b8 	__local_depot0[32];
	.reg .b64 	%SP;
	.reg .b64 	%SPL;
	.reg .pred 	%p<2>;
	.reg .b32 	%r<9>;
	.reg .f32 	%f<4>;
	.reg .b64 	%rd<18>;

// %bb.0:                               // %entry
	mov.u64 	%SPL, __local_depot0;
	cvta.local.u64 	%SP, %SPL;
	ld.param.u32 	%r1, [_Z9vectorAddPKfS0_Pfi_param_3];
	ld.param.u64 	%rd3, [_Z9vectorAddPKfS0_Pfi_param_2];
	ld.param.u64 	%rd2, [_Z9vectorAddPKfS0_Pfi_param_1];
	ld.param.u64 	%rd1, [_Z9vectorAddPKfS0_Pfi_param_0];
	cvta.to.global.u64 	%rd4, %rd3;
	cvta.global.u64 	%rd5, %rd4;
	cvta.to.global.u64 	%rd6, %rd2;
	cvta.global.u64 	%rd7, %rd6;
	cvta.to.global.u64 	%rd8, %rd1;
	cvta.global.u64 	%rd9, %rd8;
	st.u64 	[%SP], %rd9;
	st.u64 	[%SP+8], %rd7;
	st.u64 	[%SP+16], %rd5;
	st.u32 	[%SP+24], %r1;
	mov.u32 	%r2, %ctaid.x;
	mov.u32 	%r3, %ntid.x;
	mul.lo.s32 	%r4, %r2, %r3;
	mov.u32 	%r5, %tid.x;
	add.s32 	%r6, %r4, %r5;
	st.u32 	[%SP+28], %r6;
	ld.u32 	%r7, [%SP+28];
	ld.u32 	%r8, [%SP+24];
	setp.ge.s32 	%p1, %r7, %r8;
	@%p1 bra 	$L__BB0_2;
	bra.uni 	$L__BB0_1;
$L__BB0_1:                              // %if.then
	ld.u64 	%rd10, [%SP];
	ld.s32 	%rd11, [%SP+28];
	shl.b64 	%rd12, %rd11, 2;
	add.s64 	%rd13, %rd10, %rd12;
	ld.f32 	%f1, [%rd13];
	ld.u64 	%rd14, [%SP+8];
	add.s64 	%rd15, %rd14, %rd12;
	ld.f32 	%f2, [%rd15];
	add.rn.f32 	%f3, %f1, %f2;
	ld.u64 	%rd16, [%SP+16];
	add.s64 	%rd17, %rd16, %rd12;
	st.f32 	[%rd17], %f3;
	bra.uni 	$L__BB0_2;
$L__BB0_2:                              // %if.end
	ret;
                                        // -- End function
}

```

### SASS
编译main PTX 至SASS
```
[root@localhost cuda]#  "/usr/local/cuda-11.8/bin/ptxas" "-m64" "-O0" "--gpu-name" "sm_61" "--output-file" "/tmp/main-sm_61-bf5478.o" "/tmp/main-sm_61-cf6aa2.s"
[root@localhost cuda]# file /tmp/main-sm_61-bf5478.o
/tmp/main-sm_61-bf5478.o: ELF 64-bit LSB executable, NVIDIA CUDA architecture,, statically linked, not stripped
[root@localhost cuda]# llvm-readelf -S  /tmp/main-sm_61-bf5478.o
There are 5 section headers, starting at offset 0x1d8:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .shstrtab         STRTAB          0000000000000000 000040 000041 00      0   0  1
  [ 2] .strtab           STRTAB          0000000000000000 000081 000041 00      0   0  1
  [ 3] .symtab           SYMTAB          0000000000000000 0000c8 000030 18      2   2  8
  [ 4] .nv.rel.action    LOPROC+0xB      0000000000000000 0000f8 0000e0 08      0   0  8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  R (retain), p (processor specific)
```



### Fatbinary
```
[root@localhost cuda]#  "/usr/local/cuda-11.8/bin/fatbinary" "-64" "--create" "/tmp/main-24e3a4.fatbin" "--image=profile=sm_61,file=/tmp/main-sm_61-bf5478.o"
[root@localhost cuda]# file /tmp/main-24e3a4.fatbin
/tmp/main-24e3a4.fatbin: data
```

```

```


### Host Object Code
```
[root@localhost cuda]# llvm-readelf -S /tmp/main-c1d440.o
There are 16 section headers, starting at offset 0xbc8:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 0009a9 00021b 00      0   0  1
  [ 2] .text             PROGBITS        0000000000000000 000040 000223 00  AX  0   0 16
  [ 3] .rela.text        RELA            0000000000000000 000650 000240 18   I 15   2  8
  [ 4] .text.startup     PROGBITS        0000000000000000 000270 00003b 00  AX  0   0 16
  [ 5] .rela.text.startup RELA           0000000000000000 000890 000090 18   I 15   4  8
  [ 6] .bss              NOBITS          0000000000000000 0002ab 000001 00  WA  0   0  1
  [ 7] .rodata.str1.1    PROGBITS        0000000000000000 0002ab 000008 01 AMS  0   0  1
  [ 8] .init_array       INIT_ARRAY      0000000000000000 0002b8 000008 00  WA  0   0  8
  [ 9] .rela.init_array  RELA            0000000000000000 000920 000018 18   I 15   8  8
  [10] .comment          PROGBITS        0000000000000000 0002c0 000070 01  MS  0   0  1
  [11] .note.GNU-stack   PROGBITS        0000000000000000 000330 000000 00      0   0  1
  [12] .eh_frame         X86_64_UNWIND   0000000000000000 000330 000098 00   A  0   0  8
  [13] .rela.eh_frame    RELA            0000000000000000 000938 000060 18   I 15  12  8
  [14] .llvm_addrsig     LLVM_ADDRSIG    0000000000000000 000998 000011 00   E 15   0  1
  [15] .symtab           SYMTAB          0000000000000000 0003c8 000288 18      1  11  8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  R (retain), l (large), p (processor specific)


  [root@localhost cuda]# llvm-readelf -s /tmp/cuda-7935de.o

Symbol table '.symtab' contains 25 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS cuda.cu
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT     2 .text
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT     5 .text._ZN4dim3C2Ejjj
     4: 0000000000000160    77 FUNC    LOCAL  DEFAULT     2 __cuda_register_globals
     5: 0000000000000000    22 OBJECT  LOCAL  DEFAULT     6 .L__unnamed_1
     6: 00000000000001b0    55 FUNC    LOCAL  DEFAULT     2 __cuda_module_ctor
     7: 0000000000000000    24 OBJECT  LOCAL  DEFAULT     8 __cuda_fatbin_wrapper
     8: 0000000000000000     8 OBJECT  LOCAL  DEFAULT    10 __cuda_gpubin_handle
     9: 00000000000001f0    15 FUNC    LOCAL  DEFAULT     2 __cuda_module_dtor
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT     7 .nv_fatbin
    11: 0000000000000000     0 SECTION LOCAL  DEFAULT     8 .nvFatBinSegment
    12: 0000000000000000     0 SECTION LOCAL  DEFAULT    10 .bss
    13: 0000000000000000   173 FUNC    GLOBAL DEFAULT     2 _Z24__device_stub__vectorAddPKfS0_Pfi
    14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaPopCallConfiguration
    15: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND cudaLaunchKernel
    16: 00000000000000b0   175 FUNC    GLOBAL DEFAULT     2 launchVectorAdd
    17: 0000000000000000    40 FUNC    WEAK   DEFAULT     5 _ZN4dim3C2Ejjj
    18: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaPushCallConfiguration
    19: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND cudaDeviceSynchronize
    20: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaRegisterFunction
    21: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaRegisterFatBinary
    22: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaRegisterFatBinaryEnd
    23: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND atexit
    24: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaUnregisterFatBinary
```

注意索引为10的section
```
[root@localhost cuda]# llvm-readelf -S /tmp/cuda-7935de.o
There are 19 section headers, starting at offset 0x1d38:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 001b00 000232 00      0   0  1
  [ 2] .text             PROGBITS        0000000000000000 000040 0001ff 00  AX  0   0 16
  [ 3] .rela.text        RELA            0000000000000000 001880 0001b0 18   I 18   2  8
  [ 4] .group            GROUP           0000000000000000 001620 000008 04     18  17  4
  [ 5] .text._ZN4dim3C2Ejjj PROGBITS     0000000000000000 000240 000028 00 AXG  0   0 16
  [ 6] .rodata.str1.1    PROGBITS        0000000000000000 000268 000016 01 AMS  0   0  1
  [ 7] .nv_fatbin        PROGBITS        0000000000000000 000280 001250 00   A  0   0  8
  [ 8] .nvFatBinSegment  PROGBITS        0000000000000000 0014d0 000018 00  WA  0   0  8
  [ 9] .rela.nvFatBinSegment RELA        0000000000000000 001a30 000018 18   I 18   8  8
  [10] .bss              NOBITS          0000000000000000 0014e8 000008 00  WA  0   0  8
  [11] .init_array       INIT_ARRAY      0000000000000000 0014e8 000008 00  WA  0   0  8
  [12] .rela.init_array  RELA            0000000000000000 001a48 000018 18   I 18  11  8
  [13] .comment          PROGBITS        0000000000000000 0014f0 000070 01  MS  0   0  1
  [14] .note.GNU-stack   PROGBITS        0000000000000000 001560 000000 00      0   0  1
  [15] .eh_frame         X86_64_UNWIND   0000000000000000 001560 0000c0 00   A  0   0  8
  [16] .rela.eh_frame    RELA            0000000000000000 001a60 000090 18   I 18  15  8
  [17] .llvm_addrsig     LLVM_ADDRSIG    0000000000000000 001af0 000010 00   E 18   0  1
  [18] .symtab           SYMTAB          0000000000000000 001628 000258 18      1  13  8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  R (retain), l (large), p (processor specific)

[root@localhost cuda]# llvm-readelf -s /tmp/cuda-7935de.o

Symbol table '.symtab' contains 25 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS cuda.cu
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT     2 .text
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT     5 .text._ZN4dim3C2Ejjj
     4: 0000000000000160    77 FUNC    LOCAL  DEFAULT     2 __cuda_register_globals
     5: 0000000000000000    22 OBJECT  LOCAL  DEFAULT     6 .L__unnamed_1
     6: 00000000000001b0    55 FUNC    LOCAL  DEFAULT     2 __cuda_module_ctor
     7: 0000000000000000    24 OBJECT  LOCAL  DEFAULT     8 __cuda_fatbin_wrapper
     8: 0000000000000000     8 OBJECT  LOCAL  DEFAULT    10 __cuda_gpubin_handle
     9: 00000000000001f0    15 FUNC    LOCAL  DEFAULT     2 __cuda_module_dtor
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT     7 .nv_fatbin
    11: 0000000000000000     0 SECTION LOCAL  DEFAULT     8 .nvFatBinSegment
    12: 0000000000000000     0 SECTION LOCAL  DEFAULT    10 .bss
    13: 0000000000000000   173 FUNC    GLOBAL DEFAULT     2 _Z24__device_stub__vectorAddPKfS0_Pfi
    14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaPopCallConfiguration
    15: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND cudaLaunchKernel
    16: 00000000000000b0   175 FUNC    GLOBAL DEFAULT     2 launchVectorAdd
    17: 0000000000000000    40 FUNC    WEAK   DEFAULT     5 _ZN4dim3C2Ejjj
    18: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaPushCallConfiguration
    19: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND cudaDeviceSynchronize
    20: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaRegisterFunction
    21: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaRegisterFatBinary
    22: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaRegisterFatBinaryEnd
    23: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND atexit
    24: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND __cudaUnregisterFatBinary
```

### CUDA JIT
```
#include <iostream>
#include <fstream>
#include <cassert>
#include "cuda.h"


void checkCudaErrors(CUresult err) {
  assert(err == CUDA_SUCCESS);
  std::cout << __LINE__ << err << "\n";
}

/// main - Program entry point
int main(int argc, char **argv) {
  CUdevice    device;
  CUmodule    cudaModule;
  CUcontext   context;
  CUfunction  function;
  CUlinkState linker;
  int         devCount;

  // CUDA initialization
  checkCudaErrors(cuInit(0));
  checkCudaErrors(cuDeviceGetCount(&devCount));
  checkCudaErrors(cuDeviceGet(&device, 0));

  char name[128];
  checkCudaErrors(cuDeviceGetName(name, 128, device));
  std::cout << "Using CUDA Device [0]: " << name << "\n";

  int devMajor, devMinor;
  checkCudaErrors(cuDeviceComputeCapability(&devMajor, &devMinor, device));
  std::cout << "Device Compute Capability: "
            << devMajor << "." << devMinor << "\n";
  if (devMajor < 2) {
    std::cerr << "ERROR: Device 0 is not SM 2.0 or greater\n";
    return 1;
  }

  std::ifstream t("kernel.ptx");
  if (!t.is_open()) {
    std::cerr << "kernel.ptx not found\n";
    return 1;
  }
  std::string str((std::istreambuf_iterator<char>(t)),
                    std::istreambuf_iterator<char>());

  // Create driver context
  checkCudaErrors(cuCtxCreate(&context, 0, device));

  // Create module for object
  checkCudaErrors(cuModuleLoadDataEx(&cudaModule, str.c_str(), 0, 0, 0));

  // Get kernel function
  checkCudaErrors(cuModuleGetFunction(&function, cudaModule, "kernel"));

  // Device data
  CUdeviceptr devBufferA;
  CUdeviceptr devBufferB;
  CUdeviceptr devBufferC;

  checkCudaErrors(cuMemAlloc(&devBufferA, sizeof(float)*16));
  checkCudaErrors(cuMemAlloc(&devBufferB, sizeof(float)*16));
  checkCudaErrors(cuMemAlloc(&devBufferC, sizeof(float)*16));

  float* hostA = new float[16];
  float* hostB = new float[16];
  float* hostC = new float[16];

  // Populate input
  for (unsigned i = 0; i != 16; ++i) {
    hostA[i] = (float)i;
    hostB[i] = (float)(2*i);
    hostC[i] = 0.0f;
  }

  checkCudaErrors(cuMemcpyHtoD(devBufferA, &hostA[0], sizeof(float)*16));
  checkCudaErrors(cuMemcpyHtoD(devBufferB, &hostB[0], sizeof(float)*16));


  unsigned blockSizeX = 16;
  unsigned blockSizeY = 1;
  unsigned blockSizeZ = 1;
  unsigned gridSizeX  = 1;
  unsigned gridSizeY  = 1;
  unsigned gridSizeZ  = 1;

  // Kernel parameters
  void *KernelParams[] = { &devBufferA, &devBufferB, &devBufferC };

  std::cout << "Launching kernel\n";

  // Kernel launch
  checkCudaErrors(cuLaunchKernel(function, gridSizeX, gridSizeY, gridSizeZ,
                                 blockSizeX, blockSizeY, blockSizeZ,
                                 0, NULL, KernelParams, NULL));

  // Retrieve device data
  checkCudaErrors(cuMemcpyDtoH(&hostC[0], devBufferC, sizeof(float)*16));


  std::cout << "Results:\n";
  for (unsigned i = 0; i != 16; ++i) {
    std::cout << hostA[i] << " + " << hostB[i] << " = " << hostC[i] << "\n";
  }


  // Clean up after ourselves
  delete [] hostA;
  delete [] hostB;
  delete [] hostC;

  // Clean-up
  checkCudaErrors(cuMemFree(devBufferA));
  checkCudaErrors(cuMemFree(devBufferB));
  checkCudaErrors(cuMemFree(devBufferC));
  checkCudaErrors(cuModuleUnload(cudaModule));
  checkCudaErrors(cuCtxDestroy(context));

  return 0;
}

```
