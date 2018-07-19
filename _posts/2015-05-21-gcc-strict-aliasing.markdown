---
title:  "GCC 优化选项之 strict aliasing"
date:   2015-05-21 20:14:11 +0800
author: elxy
categories: C
---

GCC 优化方式中有一个 `-fstrict-aliasing` 选项。什么是 `strict aliasing` 呢？简单解释就是：“不同类型”的指针，一定不会引用同一个内存区域。c11 标准 6.5 节对别名的具体说明如下：

> An object shall have its stored value accessed only by an lvalue expression that has one of the following types:
>
> — a type compatible with the effective type of the object,
>
> — a qualified version of a type compatible with the effective type of the object,
>
> — a type that is the signed or unsigned type corresponding to the effective type of the object,
>
> — a type that is the signed or unsigned type corresponding to a qualified version of the effective type of the object,
>
> — an aggregate or union type that includes one of the aforementioned types among its members (including, recursively, a member of a subaggregate or contained union), or
>
> — a character type.

在 `strict aliasing` 规则下，编译器可以进行相应的优化。看下面的例子：

```c
int n;

int foo(float *ptr)
{
      n = 1;
      *ptr = 3;
      return n;
}

int main()
{
      printf("%d\n", foo((float *) &n));
      printf("%d\n", n); 
      return 0;
}
```

编译并运行：

```console
$ gcc test.c && ./a.out 
1077936128
1077936128
$ gcc -O2 test.c && ./a.out 
1
1077936128
$ gcc -O2 -fno-strict-aliasing test.c && ./a.out 
1077936128
1077936128
```

在加上 `-O2` 选项后（`strict-aliasing` 优化在 `O2` 以上才开启[^gcc_optimize_options]），第一次输出竟然是 `1`，难道 `*ptr = 3` 没有执行？其实是执行了的，不信看看汇编代码（`google-code-prettify` 不支持 ASM 高亮，凑合看吧）：

```nasm
; gcc -S -O2 test.c
        .file   "strict_aliasing.c"
        .text
        .p2align 4,,15
        .globl  foo
        .type   foo, @function
foo:
.LFB11:
        .cfi_startproc
        movl    $1, n(%rip) ; 将 1 保存在变量 n 中
        movl    $1, %eax ; 直接将返回值 1 存在 EAX 寄存器中
        movl    $0x40400000, (%rdi) ; 将直接数 3 (int) 按 float 类型保存到第一个参数所指地址中
        ret
        .cfi_endproc
.LFE11:
        .size   foo, .-foo
        .section        .rodata.str1.1,"aMS",@progbits,1
.LC1:
        .string "%d\n"
        .section        .text.startup,"ax",@progbits
        .p2align 4,,15
        .globl  main
        .type   main, @function
main:
.LFB12:
        .cfi_startproc
        subq    $8, %rsp ; 栈顶上移 8 字节，这之前还有 8 字节保存返回地址
        .cfi_def_cfa_offset 16 ; 一共移动了 16 字节，相对于 CFA
        movl    $1, %esi ; 这是第二个参数，由于 foo 函数被优化内联了，故直接赋值 1
        movl    $.LC1, %edi ; EDI 指向 "%d\n" 字符串，是第一个参数
        xorl    %eax, %eax ; 将 EAX 置零，不明白啥意思
        movl    $0x40400000, n(%rip) ; 此时才将值 3 (int) 按 float 类型保存到变量 n 中
        call    printf ; 调用 printf
        movl    n(%rip), %esi ; 将 n 的值作为第二个参数
        movl    $.LC1, %edi ; 第一个参数还是指向字符串
        xorl    %eax, %eax ; 将 EAX 置零，不明白啥意思
        call    printf ; 调用 printf
        xorl    %eax, %eax ; 将 EAX 置零，main 函数返回为 0
        addq    $8, %rsp ; 栈顶下移 8 字节，此时 RSP 指向返回地址
        .cfi_def_cfa_offset 8 ; 这时相对于 CFA 就还有 8 字节
        ret
        .cfi_endproc
.LFE12:
        .size   main, .-main
        .comm   n,4,4 ; 存储符号链接，分别是符号名称，长度，和对齐参数
        .ident  "GCC: (SUSE Linux) 4.8.3 20140627 [gcc-4_8-branch revision 212064]"
        .section        .note.GNU-stack,"",@progbits
```

比较这两个文件的第 `12` 行，会发现启用了 `strict aliasing` 优化的汇编代码（含 `-O2`，不含 `-fno-strict-aliasing`），认为 `ptr` 不会是 `n` 的别名，故而函数 `foo()` 会直接返回 `1`。而禁用了 `strict aliasing` 优化的汇编代码，函数 `foo()` 多了一次访存：`movl n(%rip), %eax`。

由于 `-O2` 会将小函数内联[^gcc_optimize_options]，第 `29` 行赋值给 printf 的参数用的就是直接数，而没用变量来存了。

```nasm
; gcc -S -O2 -fno-strict-aliasing test.c
        .file   "strict_aliasing.c"
        .text
        .p2align 4,,15
        .globl  foo
        .type   foo, @function
foo:
.LFB11:
        .cfi_startproc
        movl    $1, n(%rip)
        movl    $0x40400000, (%rdi)
        movl    n(%rip), %eax ; 这里会取出变量 n 的值放到返回值中，相比直接数赋值多了一步读取内存的操作
        ret
        .cfi_endproc
.LFE11:
        .size   foo, .-foo
        .section        .rodata.str1.1,"aMS",@progbits,1
.LC1:
        .string "%d\n"
        .section        .text.startup,"ax",@progbits
        .p2align 4,,15
        .globl  main
        .type   main, @function
main:
.LFB12:
        .cfi_startproc
        subq    $8, %rsp
        .cfi_def_cfa_offset 16
        movl    $1077936128, %esi ; 函数因优化而内联，将 n 的值直接作为参数给 printf
        movl    $.LC1, %edi
        xorl    %eax, %eax
        movl    $0x40400000, n(%rip)
        call    printf
        movl    n(%rip), %esi
        movl    $.LC1, %edi
        xorl    %eax, %eax
        call    printf
        xorl    %eax, %eax
        addq    $8, %rsp
        .cfi_def_cfa_offset 8
        ret
        .cfi_endproc
.LFE12:
        .size   main, .-main
        .comm   n,4,4
        .ident  "GCC: (SUSE Linux) 4.8.3 20140627 [gcc-4_8-branch revision 212064]"
        .section        .note.GNU-stack,"",@progbits
```

由于 `-O2` 默认包含 `-fstrict-aliasing` 优化，如果不能确保自己的指针使用能像 c11 标准 6.5 节中规定得那样严格，建议加上 `-fno-strict-aliasing` 选项。

提问时间
--------

* 为什么赋值是 `3`，最后输出是 `1077936128`？

参考资料
--------

[GCC strict aliasing][2]

脚注
----

 [^gcc_optimize_options]: [GCC Optimize Options][1] 

 [1]: https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html

 [2]: http://www.dutor.net/index.php/2012/07/gcc-strict-aliasing/
