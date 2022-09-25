---
title: LD文件分析
date: 2022-09-25 21:43:43
tags: LD
categories: LD
top_img: 
cover:
---

# LD文件分析

## 1.概述

每一个链接过程都由链接脚本(linker script, 一般以lds作为文件的后缀名) 控制. 链接脚本主要用于规定如何把输入文件内的section放入输出文件内, 并控制输出文件内各部分在程序地址空间内的布局. 但你也可以用连接命令做一些其他事情.

连接器有个默认的内置连接脚本, 可用ld --verbose查看. 连接选项`-r`和`-N`可以影响默认的连接脚本(如何影响?).

`-T`选项用以指定自己的链接脚本, 它将代替默认的连接脚本。你也可以使用<暗含的连接脚本>以增加自定义的链接命令.

以下没有特殊说明，连接器指的是静态连接器. 



链接可以执行于编译时，也就是在源代码被翻译成机器代码时，也可以执行于加载时，也就是在程序在被加载器加载到内存并执行时；甚至执行于运行时，也就是由应用程序来执行。


---

## 2.基本概念

链接器把一个或多个输入文件合成一个输出文件.

输入文件: 目标文件或链接脚本文件.
输出文件: 目标文件或可执行文件.

目标文件(包括可执行文件)具有固定的格式, 在UNIX或GNU/Linux平台下, 一般为ELF格式. 若想了解更多, 可参考 UNIX/Linux平台可执行文件格式分析

有时把输入文件内的section称为输入section(input section), 把输出文件内的section称为输出section(output sectin).

目标文件的每个section至少包含两个信息: **名字和大小**。 大部分section还包含与它相关联的一块数据, 称为section contents(section内容). **一个section可被标记为“loadable(可加载的)”或“allocatable(可分配的)”.**

`loadable section`: 在输出文件运行时, 相应的section内容将被载入进程地址空间中。

`allocatable section:` 内容为空的section可被标记为“可分配的”. 在输出文件运行时, 在进程地址空间中空出大小同section指定大小的部分. 某些情况下, 这块内存必须被置零。

如果一个section不是“可加载的”或“可分配的”, 那么该section通常包含了调试信息. 可用objdump -h命令查看相关信息.

每个“可加载的”或“可分配的”输出section通常包含两个地址: `VMA(virtual memory address虚拟内存地址或程序地址空间地址)和LMA(load memory address加载内存地址或进程地址空间地址)`。 通常VMA和LMA是相同的.

在目标文件中, loadable或allocatable的输出section有两种地址: `VMA(virtual Memory Address)`和`LMA(Load Memory Address`)。 **VMA是执行输出文件时section所在的地址，而LMA是加载输出文件时section所在的地址**。一般而言，某section的VMA == LMA。 但在嵌入式系统中, 经常存在加载地址和执行地址不同的情况： **比如将输出文件加载到开发板的flash中(由LMA指定)，而在运行时将位于flash中的输出文件复制到SDRAM中(由VMA指定)**。

可这样来理解VMA和LMA, 假设:
(1) .data section对应的VMA地址是0x08050000，该section内包含了3个32位全局变量，i、j和k，分别为1，2，3。
(2) .text section内包含由"`printf( "j=%d ", j );`"程序片段产生的代码.

连接时指定`.data section`的VMA为0x08050000，产生的printf指令是将地址为0x08050004处的4字节内容作为一个整数打印出来。

如果`.data section`的LMA为0x08050000，显然结果是j=2。

```
LMA为0x08050000则加载输出文件的起始地址是0x08050000。 i、j 和 k 的地址为：
0x08050000、0x08050004、0x08050008。（32位全局变量）
```

如果`.data section`的LMA为0x08050004，显然结果是 j=1。

```
LMA为0x08050004则加载输出文件的起始地址是0x08050004。 i、j 和 k 的地址为：
0x08050004、0x08050008、0x0805000C。打印 0x08050004地址的值，实际上是 i 的值 1。
```

---

## 3.脚本格式

链接脚本由一系列命令组成, 每个命令由一个关键字(一般在其后紧跟相关参数)或一条对符号的赋值语句组成. 命令由分号‘`;`’分隔开.

文件名或格式名内如果包含分号’`;`'或其他分隔符, 则要用引号‘`"`’将名字全称引用起来. 无法处理含引号的文件名.
`/* */`之间的是注释。

---

## 4.简单例子

在介绍链接描述文件的命令之前, 先看看下述的简单例子:

以下脚本将输出文件的text section定位在0x10000, data section定位在0x8000000:

```
SECTIONS
{
  . = 0x10000;
  .text : { *(.text) }
  . = 0x8000000;
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```

解释一下上述的例子:
`. = 0x10000` : 把定位器符号置为0x10000 (若不指定, 则该符号的初始值为0).

`.text : { *(.text) }` : 将所有(*符号代表任意输入文件)输入文件的.text section合并成一个.text section, 该section的地址由定位器符号的值指定, 即0x10000.

`. = 0x8000000` ：把定位器符号置为0x8000000
`.data : { *(.data) }` : 将所有输入文件的.text section合并成一个.data section, 该section的地址被置为0x8000000.

`.bss : { *(.bss) }` : 将所有输入文件的.bss section合并成一个.bss section，该section的地址被置为`0x8000000 + .data section`的大小.

连接器每读完一个section描述后, 将定位器符号的值*增加*该section的大小. 注意: 此处没有考虑对齐约束.

---

## 5.简单脚本命令

### 1.ENTRY（symbol）：将symbol的值设置成入口地址

入口地址(entry point): 进程执行的第一条用户空间的指令在进程地址空间的地址)

ld有多种方法设置进程入口地址, 按一下顺序: (编号越前, 优先级越高)
1, ld命令行的-e选项
2, 连接脚本的ENTRY(SYMBOL)命令
3, 如果定义了start 符号, 使用start符号值
4, 如果存在 .text section , 使用.text section的第一字节的位置值
5, 使用值0

### 2.INCLUDE FILENMAE：包含其他名为FILENAME的链接脚本

相当于c程序内的的#include指令, 用以包含另一个链接脚本.

脚本搜索路径由-L选项指定。 INCLUDE指令可以嵌套使用, **最大深度为10**. 即: 文件1内INCLUDE文件2, 文件2内INCLUDE文件3… , 文件10内INCLUDE文件11. 那么文件11内不能再出现 INCLUDE指令了.

### 3.INPUT(FILES)：将括号内的文件作为链接过程的输入文件

ld首先在当前目录下寻找该文件, 如果没找到, 则在由-L指定的搜索路径下搜索. file可以为 -lfile形式，就象命令行的-l选项一样. 如果该命令出现在暗含的脚本内, 则该命令内的file在链接过程中的顺序由该暗含的脚本在命令行内的顺序决定.

### 4.GROUP(FILES)：指定需要重复搜索符号定义的多个输入文件

file必须是库文件，且file文件作为一组被ld重复扫描，知道不再有新的未定义的引用出现

### 5.OUTPUT(FILENAME)：定义输出文件的名字

同ld的-o选项, 不过-o选项的优先级更高. 所以它可以用来定义默认的输出文件名. 如a.out

### 6.SEARCH_DIR(PATH)：定义搜索路径

同ld的-L选项, 不过由-L指定的路径要比它定义的优先被搜索。

### 7.STARUP(FILENAME)：指定FILENAME为第一个输入文件

在链接过程中，每个输入文件是有顺序的，此命令设置文件filename为第一个输入文件

### 8.OUTPUT_FORMAT(BFDNAME)：设置输出文件使用的BFD格式

同ld选项-o format BFDNAME, 不过ld选项优先级更高.

### 9.OUTPUT_FORMAT(DEFAULT,BIG,LITTLE)：定义三种输出文件的格式（大小端）

若有命令行选项-EB, 则使用第2个BFD格式; 若有命令行选项-EL，则使用第3个BFD格式.否则默认选第一个BFD格式.

`TARGET(BFDNAME)`：设置输入文件的BFD格式

同ld选项-b BFDNAME. 若使用了TARGET命令, 但未使用OUTPUT_FORMAT命令, 则最用一个TARGET命令设置的BFD格式将被作为输出文件的BFD格式.

### 另外还有一些:

`ASSERT(EXP, MESSAGE)`：如果EXP不为真，终止连接过程

`EXTERN(SYMBOL SYMBOL ...)`：在输出文件中增加未定义的符号，如同连接器选项-u

`FORCE_COMMON_ALLOCATION`：为common symbol(通用符号)分配空间，即使用了-r连接选项也为其分配

`NOCROSSREFS(SECTION SECTION ...)`：检查列出的输出section，如果发现他们之间有相互引用，则报错。对于某些系统，特别是内存较紧张的嵌入式系统，某些section是不能同时存在内存中的，所以他们之间不能相互引用。

`OUTPUT_ARCH(BFDARCH)`：设置输出文件的machine architecture(体系结构)，BFDARCH为被BFD库使用的名字之一。可以用命令objdump -f查看。

可通过 man -S 1 ld查看ld的联机帮助, 里面也包括了对这些命令的介绍.

---

## 6.链接文件的过程

### 1.查看详细编译过程

main.c

```c
int sum(int *a, int n);
int array[2] = {1,2};
int main()
{
	int val =sum(array,2);
    return val;
}
```

sum.c

```c
int sum(int *a, int n)
{
    int i = 0;
    int s = 0;
    for (i = 0; i < n; i++) {
        s += a[i];
    }
    return s;
}
```

在linux中写下上面两个文件，执行命令`gcc -v -Og -o prog main.c sum.c`可以看到详细的编译过程

```shell
yuh@yuh:~/code/other/link_demo$ gcc -v  -Og -o prog main.c sum.c
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-linux-gnu/11/lto-wrapper
OFFLOAD_TARGET_NAMES=nvptx-none:amdgcn-amdhsa
OFFLOAD_TARGET_DEFAULT=1
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 11.2.0-19ubuntu1' --with-bugurl=file:///usr/share/doc/gcc-11/README.Bugs --enable-languages=c,ada,c++,go,brig,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-11 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --disable-werror --enable-cet --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-offload-targets=nvptx-none=/build/gcc-11-gBFGDP/gcc-11-11.2.0/debian/tmp-nvptx/usr,amdgcn-amdhsa=/build/gcc-11-gBFGDP/gcc-11-11.2.0/debian/tmp-gcn/usr --without-cuda-driver --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu --with-build-config=bootstrap-lto-lean --enable-link-serialization=2
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.2.0 (Ubuntu 11.2.0-19ubuntu1)
COLLECT_GCC_OPTIONS='-v' '-Og' '-o' 'prog' '-mtune=generic' '-march=x86-64' '-dumpdir' 'prog-'
 /usr/lib/gcc/x86_64-linux-gnu/11/cc1 -quiet -v -imultiarch x86_64-linux-gnu main.c -quiet -dumpdir prog- -dumpbase main.c -dumpbase-ext .c -mtune=generic -march=x86-64 -Og -version -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -o /tmp/cczXoFXB.s
GNU C17 (Ubuntu 11.2.0-19ubuntu1) version 11.2.0 (x86_64-linux-gnu)
        compiled by GNU C version 11.2.0, GMP version 6.2.1, MPFR version 4.1.0, MPC version 1.2.1, isl version isl-0.24-GMP

GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
ignoring nonexistent directory "/usr/local/include/x86_64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/../../../../x86_64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc/x86_64-linux-gnu/11/include
 /usr/local/include
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
GNU C17 (Ubuntu 11.2.0-19ubuntu1) version 11.2.0 (x86_64-linux-gnu)
        compiled by GNU C version 11.2.0, GMP version 6.2.1, MPFR version 4.1.0, MPC version 1.2.1, isl version isl-0.24-GMP

GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
Compiler executable checksum: ead6677a8de2192bf1e5ee7b28d13856
COLLECT_GCC_OPTIONS='-v' '-Og' '-o' 'prog' '-mtune=generic' '-march=x86-64' '-dumpdir' 'prog-'
 as -v --64 -o /tmp/ccVyxzU3.o /tmp/cczXoFXB.s
GNU assembler version 2.38 (x86_64-linux-gnu) using BFD version (GNU Binutils for Ubuntu) 2.38
COLLECT_GCC_OPTIONS='-v' '-Og' '-o' 'prog' '-mtune=generic' '-march=x86-64' '-dumpdir' 'prog-'
 /usr/lib/gcc/x86_64-linux-gnu/11/cc1 -quiet -v -imultiarch x86_64-linux-gnu sum.c -quiet -dumpdir prog- -dumpbase sum.c -dumpbase-ext .c -mtune=generic -march=x86-64 -Og -version -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -o /tmp/cczXoFXB.s
GNU C17 (Ubuntu 11.2.0-19ubuntu1) version 11.2.0 (x86_64-linux-gnu)
        compiled by GNU C version 11.2.0, GMP version 6.2.1, MPFR version 4.1.0, MPC version 1.2.1, isl version isl-0.24-GMP

GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
ignoring nonexistent directory "/usr/local/include/x86_64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/../../../../x86_64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc/x86_64-linux-gnu/11/include
 /usr/local/include
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
GNU C17 (Ubuntu 11.2.0-19ubuntu1) version 11.2.0 (x86_64-linux-gnu)
        compiled by GNU C version 11.2.0, GMP version 6.2.1, MPFR version 4.1.0, MPC version 1.2.1, isl version isl-0.24-GMP

GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
Compiler executable checksum: ead6677a8de2192bf1e5ee7b28d13856
COLLECT_GCC_OPTIONS='-v' '-Og' '-o' 'prog' '-mtune=generic' '-march=x86-64' '-dumpdir' 'prog-'
 as -v --64 -o /tmp/ccbV57VD.o /tmp/cczXoFXB.s
GNU assembler version 2.38 (x86_64-linux-gnu) using BFD version (GNU Binutils for Ubuntu) 2.38
COMPILER_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/
LIBRARY_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../../lib/:/lib/x86_64-linux-gnu/:/lib/../lib/:/usr/lib/x86_64-linux-gnu/:/usr/lib/../lib/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-v' '-Og' '-o' 'prog' '-mtune=generic' '-march=x86-64' '-dumpdir' 'prog.'
 /usr/lib/gcc/x86_64-linux-gnu/11/collect2 -plugin /usr/lib/gcc/x86_64-linux-gnu/11/liblto_plugin.so -plugin-opt=/usr/lib/gcc/x86_64-linux-gnu/11/lto-wrapper -plugin-opt=-fresolution=/tmp/ccqQCgOu.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --build-id --eh-frame-hdr -m elf_x86_64 --hash-style=gnu --as-needed -dynamic-linker /lib64/ld-linux-x86-64.so.2 -pie -z now -z relro -o prog /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/Scrt1.o /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/11/crtbeginS.o -L/usr/lib/gcc/x86_64-linux-gnu/11 -L/usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/11/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/11/../../.. /tmp/ccVyxzU3.o /tmp/ccbV57VD.o -lgcc --push-state --as-needed -lgcc_s --pop-state -lc -lgcc --push-state --as-needed -lgcc_s --pop-state /usr/lib/gcc/x86_64-linux-gnu/11/crtendS.o /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/crtn.o
COLLECT_GCC_OPTIONS='-v' '-Og' '-o' 'prog' '-mtune=generic' '-march=x86-64' '-dumpdir' 'prog.'
```

### 2.手工控制编译过程

选项	功能
-v	查看gcc编译器的版本，显示gcc执行时的详细过程
-o	指定输出文件名为file，这个名称不能跟源文件名同名
-E	只预处理，不会编译、汇编、链接
-S	只编译，不会汇编、链接
-c	编译和汇编，不会链接
一个c/c++文件要经过预处理、编译、汇编和链接才能变成可执行文件。

（1）预处理

C/C++源文件中，以“#”开头的命令被称为预处理命令，如包含命令“#include”、宏定义命令“#define”、条件编译命令“#if”、“#ifdef”等。
预处理就是将要包含(include)的文件插入原文件中、将宏定义展开、根据条件编译命令选择要使用的代码，最后将这些东西输出到一个“.i”文件中等待进一步处理。

> 预处理不处理语法错误，只是会执行预处理指令（包含头文件，宏定义，预处理条件开关）

（2）编译

编译就是把C/C++代码(比如上述的“.i”文件)“翻译”成汇编代码。

> 编译的时候编译器会检查语法错误。

（3）汇编

汇编就是将第二步输出的汇编代码翻译成符合一定格式的机器代码，在Linux系统上一般表现为ELF目标文件(OBJ文件)。

**“反汇编”**是指将机器代码转换为汇编代码，这在调试程序时常常用到。

（4）链接

链接就是将上步生成的OBJ文件和系统库的OBJ文件、库文件链接起来，最终生成了可以在特定平台运行的可执行文件。
### 3.手动编译链接

hello.c(预处理)->hello.i(编译)->hello.s(汇编)->hello.o(链接)->hello.exe(windows下)
详细的每一步命令如下：

``` shell
gcc -E -o hello.i hello.c
gcc -S -o hello.s hello.i
gcc -c -o hello.o hello.s
gcc -o hello hello.o
```



1. 执行`gcc -E -o hello.i hello.c`从源文件生成.i预处理文件

   我们先看源码`hello.c`

   ```c
     #include <stdio.h>                                                                                                                                                                        2
     #define HELLO_WORLD "hello world"
     int main(void)
     {
           printf("%s\n",HELLO_WORLD);
         return 0;
     }
   ```

   再看生成的`hello.i`文件

   ```c
   # 4 "hello.c"
    int main(void)
    {
   
     printf("%s\n","hello world");
     return 0;
    }   
   ```

   可以看到源码中的宏被替换掉了

   

2. 执行`gcc -S -o hello.s hello.i`

   查看`hello.s`文件

   ```assembly
        .file   "hello.c"                                                                                                                                                                     2     .text
        .section    .rodata
    .LC0:
        .string "hello world"
        .text
        .globl  main
        .type   main, @function
    main:
    .LFB0:
        .cfi_startproc
        endbr64
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        leaq    .LC0(%rip), %rax
        movq    %rax, %rdi
        call    puts@PLT
        movl    $0, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
    .LFE0:
        .size   main, .-main
        .ident  "GCC: (Ubuntu 11.2.0-19ubuntu1) 11.2.0"
        .section    .note.GNU-stack,"",@progbits
        .section    .note.gnu.property,"a"
        .align 8
        .long   1f - 0f
        .long   4f - 1f
        .long   5
    0:
        .string "GNU"
    1:
        .align 8
        .long   0xc0000002
        .long   3f - 2f
    2:
        .long   0x3
    3:
        .align 8
    4:
        
   ```

   

3. 执行`gcc -c -o hello.o hello.s`将汇编文件转化为二进制文件

   这一步将汇编代码转化为机器码，生成的`hello.o`文件可以通过命令`xxd hello.o`来查看

   ```
   00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
   00000010: 0100 3e00 0100 0000 0000 0000 0000 0000  ..>.............
   00000020: 0000 0000 0000 0000 5802 0000 0000 0000  ........X.......
   00000030: 0000 0000 4000 0000 0000 4000 0e00 0d00  ....@.....@.....
   00000040: f30f 1efa 5548 89e5 488d 0500 0000 0048  ....UH..H......H
   00000050: 89c7 e800 0000 00b8 0000 0000 5dc3 6865  ............].he
   00000060: 6c6c 6f20 776f 726c 6400 0047 4343 3a20  llo world..GCC:
   00000070: 2855 6275 6e74 7520 3131 2e32 2e30 2d31  (Ubuntu 11.2.0-1
   00000080: 3975 6275 6e74 7531 2920 3131 2e32 2e30  9ubuntu1) 11.2.0
   00000090: 0000 0000 0000 0000 0400 0000 1000 0000  ................
   000000a0: 0500 0000 474e 5500 0200 00c0 0400 0000  ....GNU.........
   000000b0: 0300 0000 0000 0000 1400 0000 0000 0000  ................
   000000c0: 017a 5200 0178 1001 1b0c 0708 9001 0000  .zR..x..........
   000000d0: 1c00 0000 1c00 0000 0000 0000 1e00 0000  ................
   000000e0: 0045 0e10 8602 430d 0655 0c07 0800 0000  .E....C..U......
   000000f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000100: 0000 0000 0000 0000 0100 0000 0400 f1ff  ................
   00000110: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000120: 0000 0000 0300 0100 0000 0000 0000 0000  ................
   00000130: 0000 0000 0000 0000 0000 0000 0300 0500  ................
   00000140: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000150: 0900 0000 1200 0100 0000 0000 0000 0000  ................
   00000160: 1e00 0000 0000 0000 0e00 0000 1000 0000  ................
   00000170: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000180: 0068 656c 6c6f 2e63 006d 6169 6e00 7075  .hello.c.main.pu
   00000190: 7473 0000 0000 0000 0b00 0000 0000 0000  ts..............
   000001a0: 0200 0000 0300 0000 fcff ffff ffff ffff  ................
   000001b0: 1300 0000 0000 0000 0400 0000 0500 0000  ................
   000001c0: fcff ffff ffff ffff 2000 0000 0000 0000  ........ .......
   000001d0: 0200 0000 0200 0000 0000 0000 0000 0000  ................
   000001e0: 002e 7379 6d74 6162 002e 7374 7274 6162  ..symtab..strtab
   000001f0: 002e 7368 7374 7274 6162 002e 7265 6c61  ..shstrtab..rela
   00000200: 2e74 6578 7400 2e64 6174 6100 2e62 7373  .text..data..bss
   00000210: 002e 726f 6461 7461 002e 636f 6d6d 656e  ..rodata..commen
   00000220: 7400 2e6e 6f74 652e 474e 552d 7374 6163  t..note.GNU-stac
   00000230: 6b00 2e6e 6f74 652e 676e 752e 7072 6f70  k..note.gnu.prop
   00000240: 6572 7479 002e 7265 6c61 2e65 685f 6672  erty..rela.eh_fr
   00000250: 616d 6500 0000 0000 0000 0000 0000 0000  ame.............
   00000260: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000270: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000280: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000290: 0000 0000 0000 0000 2000 0000 0100 0000  ........ .......
   000002a0: 0600 0000 0000 0000 0000 0000 0000 0000  ................
   000002b0: 4000 0000 0000 0000 1e00 0000 0000 0000  @...............
   000002c0: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   000002d0: 0000 0000 0000 0000 1b00 0000 0400 0000  ................
   000002e0: 4000 0000 0000 0000 0000 0000 0000 0000  @...............
   000002f0: 9801 0000 0000 0000 3000 0000 0000 0000  ........0.......
   00000300: 0b00 0000 0100 0000 0800 0000 0000 0000  ................
   00000310: 1800 0000 0000 0000 2600 0000 0100 0000  ........&.......
   00000320: 0300 0000 0000 0000 0000 0000 0000 0000  ................
   00000330: 5e00 0000 0000 0000 0000 0000 0000 0000  ^...............
   00000340: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   00000350: 0000 0000 0000 0000 2c00 0000 0800 0000  ........,.......
   00000360: 0300 0000 0000 0000 0000 0000 0000 0000  ................
   00000370: 5e00 0000 0000 0000 0000 0000 0000 0000  ^...............
   00000380: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   00000390: 0000 0000 0000 0000 3100 0000 0100 0000  ........1.......
   000003a0: 0200 0000 0000 0000 0000 0000 0000 0000  ................
   000003b0: 5e00 0000 0000 0000 0c00 0000 0000 0000  ^...............
   000003c0: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   000003d0: 0000 0000 0000 0000 3900 0000 0100 0000  ........9.......
   000003e0: 3000 0000 0000 0000 0000 0000 0000 0000  0...............
   000003f0: 6a00 0000 0000 0000 2700 0000 0000 0000  j.......'.......
   00000400: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   00000410: 0100 0000 0000 0000 4200 0000 0100 0000  ........B.......
   00000420: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000430: 9100 0000 0000 0000 0000 0000 0000 0000  ................
   00000440: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   00000450: 0000 0000 0000 0000 5200 0000 0700 0000  ........R.......
   00000460: 0200 0000 0000 0000 0000 0000 0000 0000  ................
   00000470: 9800 0000 0000 0000 2000 0000 0000 0000  ........ .......
   00000480: 0000 0000 0000 0000 0800 0000 0000 0000  ................
   00000490: 0000 0000 0000 0000 6a00 0000 0100 0000  ........j.......
   000004a0: 0200 0000 0000 0000 0000 0000 0000 0000  ................
   000004b0: b800 0000 0000 0000 3800 0000 0000 0000  ........8.......
   000004c0: 0000 0000 0000 0000 0800 0000 0000 0000  ................
   000004d0: 0000 0000 0000 0000 6500 0000 0400 0000  ........e.......
   000004e0: 4000 0000 0000 0000 0000 0000 0000 0000  @...............
   000004f0: c801 0000 0000 0000 1800 0000 0000 0000  ................
   00000500: 0b00 0000 0900 0000 0800 0000 0000 0000  ................
   00000510: 1800 0000 0000 0000 0100 0000 0200 0000  ................
   00000520: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000530: f000 0000 0000 0000 9000 0000 0000 0000  ................
   00000540: 0c00 0000 0400 0000 0800 0000 0000 0000  ................
   00000550: 1800 0000 0000 0000 0900 0000 0300 0000  ................
   00000560: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   00000570: 8001 0000 0000 0000 1300 0000 0000 0000  ................
   00000580: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   00000590: 0000 0000 0000 0000 1100 0000 0300 0000  ................
   000005a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
   000005b0: e001 0000 0000 0000 7400 0000 0000 0000  ........t.......
   000005c0: 0000 0000 0000 0000 0100 0000 0000 0000  ................
   000005d0: 0000 0000 0000 0000                      ........
   ```

   

4. 最后执行`gcc -o hello hello.o`将二进制文件生成可执行文件

