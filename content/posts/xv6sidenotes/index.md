---
title: "xv6 side notes"
subtitle:
draft: false
date: 2024-06-19
description:
keywords: xv6 riscv
license:
comment: false
weight: 0
tags:
  - OS
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
hiddenFromRelated: false
summary: "yet another time killer"
resources:
toc: true
math: true
lightgallery: false
password:
message:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

<!--more-->

### 指令集

> 1.3 RISC-V ISA Overview[^1]
>
>	A RISC-V ISA is defined as a base integer ISA, which must be present in any implementation, plus optional extensions to the base ISA.

riscv 的指令集相当模块化，「通用 ISA 」包含一个 Base ISA 加上若干个标准扩展（Chapter 24 RV32/64G Instruction Set Listings）。

根据 `Makefile`，qemu 模拟的平台 ISA 为 RV64GC。

### assembly

给 objdump 加上 `-M no-aliases` 生成的 "canonical instruction"，可以更准确的去查具体的指令。

对 lab4 traps 中 call.asm 的一些注释：

```asm
int g(int x) {
   0:   1141                    addi    sp,sp,-16   ; In the standard RISC-V calling convention, the stack pointer sp is always 16-byte aligned.
   2:   e422                    sd      s0,8(sp)        ; Save s0 because it's callee saved. It's also the previous frame pointer.
   4:   0800                    addi    s0,sp,16    ; s0 is the current frame pointer.
  return x+3;
}
   6:   250d                    addiw   a0,a0,3     ; x + 3
   8:   6422                    ld      s0,8(sp)        ; Pointer to stack frame for returned function.
   a:   0141                    addi    sp,sp,16    ; Release stack
   c:   8082                    ret

000000000000000e <f>:

int f(int x) {                                      ; Call to `g` inlined.
   e:   1141                    addi    sp,sp,-16
  10:   e422                    sd      s0,8(sp)
  12:   0800                    addi    s0,sp,16
  return g(x);
}
  14:   250d                    addiw   a0,a0,3
  16:   6422                    ld      s0,8(sp)
  18:   0141                    addi    sp,sp,16
  1a:   8082                    ret

000000000000001c <main>:

void main(void) {
  1c:   1141                    addi    sp,sp,-16
  1e:   e406                    sd      ra,8(sp)
  20:   e022                    sd      s0,0(sp)
  22:   0800                    addi    s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:   4635                    li      a2,13           ; arg[2]
  26:   45b1                    li      a1,12           ; arg[1]
  28:   00000517                auipc   a0,0x0          ; arg[0]
  2c:   7c850513                addi    a0,a0,1992 # 7f0 <malloc+0x108> ; arg[0]
  30:   00000097                auipc   ra,0x0                          ; Set return address
  34:   600080e7                jalr    1536(ra) # 630 <printf>         ; Jump to printf
  exit(0);
  38:   4501                    li      a0,0
  3a:   00000097                auipc   ra,0x0
  3e:   28e080e7                jalr    654(ra) # 2c8 <exit>
```

### 安装 uservec

`ecall` 指令使得用户程序陷入 `trap`，根据 `stvec` 跳转到某个地址。xv6 在用户态使用 `ecall` 会跳转到 `uservec@trapoline.S`。将 `stvec` 配置到 `uservec` 的过程是：

每一个进程在执行 `allocproc()` 过程中，`p->context.ra` 指向 `forkret`，当调度器第一次选中这个进程时，通过 `swtch` 使 `ra` 指向 `p->context.ra` 也即 `forkret`，从而从 `swtch` 中执行 `ret` 会跳转到 `forkret`。`forkret` 会执行 `usertrapret`，在 `usertrapret` 中修改 `stvec` 指向 `uservec@trampoline`。

### pgtbl

即使我按照说明，bug free 地实现了所有要求，执行 usertests 我依然会看到类似下面的信息：

```
test copyout: usertrap(): unexpected scause 0x000000000000000c pid=7
            sepc=0x0000000000062e58 stval=0x0000000000062e58
FAILED
SOME TESTS FAILED
```

来试着研究下为什么会有这种情况。

我这里直接跳过经过检查没有问题的部分，直接说明我是怎么找到问题的。先来看下出问题的测试是什么：

```c
// what if you pass ridiculous pointers to system calls
// that write user memory with copyout?
void copyout(char *s) {
  uint64 addrs[] = {0LL, 0x80000000LL, 0xffffffffffffffff};

  for (int ai = 0; ai < 2; ai++) {
    uint64 addr = addrs[ai];

    int fd = open("README", 0);
    if (fd < 0) {
      printf("open(README) failed\n");
      exit(1);
    }
    int n = read(fd, (void *)addr, 8192);
    if (n > 0) {
      printf("read(fd, %p, 8192) returned %d, not -1 or 0\n", addr, n);
      exit(1);
    }
    close(fd);

    int fds[2];
    if (pipe(fds) < 0) {
      printf("pipe() failed\n");
      exit(1);
    }
    n = write(fds[1], "x", 1);
    if (n != 1) {
      printf("pipe write failed\n");
      exit(1);
    }
    n = read(fds[0], (void *)addr, 8192);
    if (n > 0) {
      printf("read(pipe, %p, 8192) returned %d, not -1 or 0\n", addr, n);
      exit(1);
    }
    close(fds[0]);
    close(fds[1]);
  }
}
```

可以知道 `addrs` 中的三个地址都应该是不可写的，从而 read 调用应该返回 -1 或 0。gdb 一下看看和预期是否一致。

为了简单，通过 `CPUS=1` 限制下虚拟机的 CPU 数量：
```shell
# VM
# 1.
make CPUS=1 qemu-gdb
# 4. (qemu)
usertests

# 2.
riscv64-linux-gnu-gdb -x ./.gdbinit
# 3. (gdb)
file user/_usertests
break copyout
c
# 5. (gdb)
n
```

在 5. 处执行到 copyout(usertests.c)，然后逐行执行，注意到第一次执行到 read 之后，程序崩溃：

```plaintext
(gdb)
89          int n = read(fd, (void*)addr, 8192);
(gdb)
90          if(n > 0){
(gdb)
0x0000000000062e58 in ?? ()
=> 0x0000000000062e58:
Cannot access memory at address 0x62e58
```

也就是测试尝试向地址 0 写入一些内容的时候，程序崩溃了。下一步类似的，去看下内核态发生了什么事情，这次 break 到内核的 copyout(vm.c)。

```plaintext
Breakpoint 2, copyout (pagetable=0x800fb000, dstva=dstva@entry=0,
    src=src@entry=0x800148e8 <bcache+13456> "xv6 is a re-implementation of Dennis Ritchie's and Ken Thompson's Unix\nVersion 6 (v6).  xv
6 loosely follows the structure and style of v6,\nbut is implemented for a modern RISC-V multiprocessor using A"...,
    len=len@entry=1024) at kernel/vm.c:364
364       while(len > 0){
(gdb) n
365         va0 = PGROUNDDOWN(dstva);
(gdb)
366         if(va0 >= MAXVA)
(gdb)
368         pte = walk(pagetable, va0, 0);
(gdb)
369         if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0 ||
(gdb)
374         if(n > len)
(gdb)
376         memmove((void *)(pa0 + (dstva - va0)), src, n);
(gdb) disassemble pa0 pa0+64
A syntax error in expression, near `pa0+64'.
(gdb) disassemble pa0,pa0+64
Dump of assembler code from 0x82d11000 to 0x82d11080:
   0x0000000082d11000:  addi    sp,sp,-16
   0x0000000082d11002:  sd      ra,8(sp)
   0x0000000082d11004:  sd      s0,0(sp)
   0x0000000082d11006:  addi    s0,sp,16
   0x0000000082d11008:  li      a1,513
   0x0000000082d1100c:  li      a0,1
   0x0000000082d1100e:  slli    a0,a0,0x1f
   0x0000000082d11010:  auipc   ra,0x6
   0x0000000082d11014:  jalr    -278(ra) # 0x82d16efa
   0x0000000082d11018:  bgez    a0,0x82d11038
   0x0000000082d1101c:  li      a1,513
   0x0000000082d11020:  li      a0,-1
   0x0000000082d11022:  auipc   ra,0x6
   0x0000000082d11026:  jalr    -296(ra) # 0x82d16efa
   0x0000000082d1102a:  li      a1,-1
   0x0000000082d1102c:  bgez    a0,0x82d1103c
   0x0000000082d11030:  ld      ra,8(sp)
   0x0000000082d11032:  ld      s0,0(sp)
   0x0000000082d11034:  addi    sp,sp,16
   0x0000000082d11036:  ret
   0x0000000082d11038:  li      a1,1
   0x0000000082d1103a:  slli    a1,a1,0x1f
   0x0000000082d1103c:  mv      a2,a0
End of assembler dump.
(gdb) n
378         len -= n;
(gdb) p /x *pte
$2 = 0x2148ec5f
(gdb) disassemble 0x82d11000,0x82d11040
Dump of assembler code from 0x82d11000 to 0x82d11040:
   0x0000000082d11000:  ld      a4,232(a2)
   0x0000000082d11002:  fld     ft0,328(sp)
   0x0000000082d11004:  lui     t1,0xffffa
   0x0000000082d11006:  ld      s0,64(a0)
   0x0000000082d11008:  ld      s0,96(a2)
   0x0000000082d1100a:  addiw   s10,s10,25
   0x0000000082d1100c:  lui     s10,0x1a
   0x0000000082d1100e:  ld      a2,216(s0)
   0x0000000082d11010:  lui     s10,0x19
   0x0000000082d11012:  lui     t3,0x19
   0x0000000082d11014:  ld      a3,192(a0)
   0x0000000082d11016:  ld      a3,208(a0)
   0x0000000082d11018:  jal     t3,0x82d1770a
   0x0000000082d1101c:  fld     ft0,88(sp)
   0x0000000082d1101e:  ld      s1,136(a0)
   0x0000000082d11020:  ld      t3,216(sp)
   0x0000000082d11022:  lui     t1,0xffffa
   0x0000000082d11024:  lw      s0,96(a2)
   0x0000000082d11026:  lui     s0,0xffffa
   0x0000000082d11028:  bltu    s2,s6,0x82d11678
   0x0000000082d1102c:  .insn   4, 0x61207327
   0x0000000082d11030:  ld      s0,216(sp)
   0x0000000082d11032:  lw      s0,80(a4)
   0x0000000082d11034:  lui     t3,0x19
   0x0000000082d11036:  lw      s0,104(s0)
   0x0000000082d11038:  ld      a0,216(a4)
   0x0000000082d1103a:  c.lui   zero,0xffffb
   0x0000000082d1103c:  csrrsi  t5,0x276,28
End of assembler dump.
```

可以发现，copyout 成功地向虚拟地址 0 写入了数据（`memmove((void *)(pa0 + (dstva - va0)), src, n);`没有报错），这附近应该是程序的文本段(user/user.ld)，不应该是可写的，但是 `p /x *pte` 确实告诉我们这里实际上是可写的。对比 `user/usertests.asm` 和上面第一次 disassemble 的输出可以印证 0 确实是被载入了程序的可执行部分。

到这里为止，受不了啦，向课程 stuff 求助！惊喜的是，只经过 1 个周出头，我真的收到了回复邮件，甚至地址是大名鼎鼎的 **Robert Morris** 在 MIT 的邮箱地址，😲。anwyay，至少从邮件可以确认，我使用的 xv6 的仓库代码 "slightly broken"。在邮件中教授提供了简单的 fix：直接跳过无法通过的测试，也就是我的仓库中的提交 23cc5b6。

进一步的找原因，`objdump --header usertests.o` 的结果告诉我：
```
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00005be6  0000000000000000  0000000000000000  00000040  2**1
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
```
可以看到这里这个文件中的 `.text` 段的权限还是 `READONLY`，但是 `readelf --header _usertests` 中有
```
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  RISCV_ATTRIBUT 0x000000000002b1f3 0x0000000000000000 0x0000000000000000
                 0x0000000000000053 0x0000000000000000  R      0x1
  LOAD           0x0000000000001000 0x0000000000000000 0x0000000000000000
                 0x000000000000a990 0x00000000000111c8  RWE    0x1000
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10

 Section to Segment mapping:
  Segment Sections...
   00     .riscv.attributes
   01     .text .rodata .eh_frame .data .bss
   02
```

那么也就是链接阶段出的问题，我们开 make 的 verbose 模式看看 `_usertests` 是怎么生成的：
```
riscv64-linux-gnu-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -gdwarf-2 -DSOL_PGTBL -DLAB_PGTBL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o user/usertests.o user/usertests.c
   Successfully remade target file 'user/usertests.o'.
  Finished prerequisites of target file 'user/_usertests'.
  Must remake target 'user/_usertests'.
riscv64-linux-gnu-ld -z max-page-size=4096 -T user/user.ld -o user/_usertests user/usertests.o user/ulib.o user/usys.o user/printf.o user/umalloc.o
riscv64-linux-gnu-ld: warning: user/_usertests has a LOAD segment with RWX permissions
riscv64-linux-gnu-objdump -S user/_usertests > user/usertests.asm
riscv64-linux-gnu-objdump -t user/_usertests | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > user/usertests.sym
  Successfully remade target file 'user/_usertests'.
```

ld 的参数很少，合理怀疑 user/user.ld 的问题。uhhh，虽然不确定可不可行，我们使用默认的 ld script 尝试过一下链接：
```
> riscv64-linux-gnu-ld -z max-page-size=4096 -o user/_usertests user/usertests.o user/ulib.o user/usys.o user/printf.o user/umalloc.o
riscv64-linux-gnu-ld: warning: cannot find entry symbol _start; defaulting to 0000000000010120
```

虽然有 warning 找不到默认的入口，但是成功生成了可执行文件，我们再看一下 program header：
```
> readelf --header user/_usertests
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  RISCV_ATTRIBUT 0x000000000000a45a 0x0000000000000000 0x0000000000000000
                 0x0000000000000053 0x0000000000000000  R      0x1
  LOAD           0x0000000000000000 0x0000000000010000 0x0000000000010000
                 0x0000000000009f28 0x0000000000009f28  R E    0x1000
  LOAD           0x000000000000a000 0x000000000001a000 0x000000000001a000
                 0x0000000000000448 0x0000000000006c78  RW     0x1000
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10

 Section to Segment mapping:
  Segment Sections...
   00     .riscv.attributes
   01     .text .rodata .eh_frame
   02     .data .sdata .sbss .bss
   03
```
可以看到多了一个 LOAD 分段，两边权限不同，其中一个不带 W，之前的 warning 也消失了。

后面的验证，有点懒得弄了，总之大致弄清楚问题出在哪里了：user.ld 并不完善，导致文本和一些数据被放在了一个段中，这个段被设置了 RWE 权限。

后话：实际上，之前每次 `make qemu` 好像总有 `riscv64-linux-gnu-ld: warning: user/_usertests has a LOAD segment with RWX permissions` 但是这并没有引起我的注意😓。
### cow-fork

实现这个功能，只需要维护用户进程 vm 对 page 的引用。由于多个 CPU 可能同时修改不同进程的页表，我们需要对引用技术加锁进行同步。

对于引用计数，需要解决下面几个问题：
1. 对哪些地址进行计数。
   可以被用户进程使用的地址空间为 memory allocator 的可用地址。(kalloc.c)
2. 有哪些操作在什么地方影响了引用计数。
   mappages 和 uvmunmap。特别地，mappages 同时被内核页表、用户进程页表使用，需要过滤内核页表的操作，只计算用户页表的引用。另一方面，trapframe 和 trampoline 比较特殊，可以不参与 cow。
3. 如何使用锁。
   我的实现中，mappages 和 uvmunmap 均不获取锁，而是要求 caller 获取锁。

### networking

nettests 中有 dns test，里面使用的 dns server 为 8.8.8.8。在国内的网络环境下，由于众所周知的原因请求有几率失败，看起来测试的代码没有「超时」的检测，一旦请求失败测试会卡在这一步。将 dns 服务器换成 119.29.29.29 后没有发现卡在 dns test 这一步骤。

[^1]: riscv-spec-20191213 (https://github.com/riscv/riscv-isa-manual/releases/tag/20240411)
