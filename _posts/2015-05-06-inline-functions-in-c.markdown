---
layout: post
title:  "C 语言中的 inline 函数"
date:   2015-05-06 18:02:04 +0800
author: elxy
categories: C
---

`inline`函数是从 C99 才被引入标准 C 中，GNU C（还有一些其他的编译器）在更早的时候就已引入该特性（比如 GCC 1988 年的 1.21 版本）。

把函数变为`inline`函数是为了提示编译器尽可能快速地调用该函数。通常是将函数的代码嵌入调用的地方。

C99 inline 规则
---------------

对`inline`的规定在 C99 标准（ISO/IEC 9899:1999) 的 6.7.4 节。包含以下几种使用情况：

 - 函数声明（包括定义）中只有`inline`，没有`extern`。不会生成一个非内联的版本（汇编代码），不过可以调用一个非内联的版本（这个非内联的版本在在其他编译单元中，务必要定义一个，详见下面的注意事项）。本编译单元只能有一次定义，非内联版本与内联版本的代码必须是一样的。

 - 函数声明为`extern inline`。该函数会生成非内联版本，可被其他编译单元调用。如果看得到该函数的定义，即，在同一个编译单元，使用的是内联版本；如果看不到该函数的定义，则使用的是非内联版本，即和普通函数一样。

 - 函数定义为`static inline`。不会生成外部可见的非内联本，可能会生成一个`static`的非内联版本。

GNU C inline 规则
-----------------

GNU C89 定义了`inline`规则：

 - 函数声明（包括定义）中只有`inline`，没有`extern`。和 C99 的`extern inline`规则相同，

 - 函数定义为`extern inline`。和 C99 的`inline`规则相同。

 - 函数定义为`static inline`。与 C99 规则相同，唯一一个可以在 GNU C89 和 C99 之间可移植的。

inline 使用注意事项
-------------------

下面按 C99 标准说明。

`inline`只是作为对编译器优化的提示，当使用`-O0`选项禁止优化时，`inline`函数只会被当作普通函数对待，不会被展开。除非定义了`always_inline`属性。

{% highlight C %}
/* 仅适用于 GCC 编译器 */
inline void foo (const char) __attribute__((always_inline));
{% endhighlight %}

当一个函数标注了`inline`表示其定义只能被用作内联（展开），并且还会有一个非内联函数定义在程序的其他地方。当使用`-O0`选项编译下面的代码时，编译器在链接时会提示第 10 行找不到`add`。即便使用了 `-O`优化选项，编译器仍会提示找不到`add`，此时虽然第 10 行会被展开，但第 11 行使用的函数的地址，`add`会被当作外部引用，仍然会报错。

{% highlight C %}
/* C99 中缺少非内联定义时，链接出错 */
inline int add(int i, int j)
{
    return i + j;
}

int main(void)
{
...
    a = add(1, 2);
    printf("add points %p\n", add);
...
}
{% endhighlight %}

对上述错误，可以改用`static inline`修复。需要注意的是，如果编译器需要生成非内联版本，可能会浪费一些空间；在两个编译单元中比较函数地址时，结果不会是相等。

另外，还可以添加一个`non-inline`的定义，这里的函数定义与`inline`的定义必须相同。或者在头文件中补充`extern inline`声明，虽然其他编译单元用的是非内联版本。

编译 C 语言时，GCC[^gccstd]和 Clang[^clangstd]都默认采用增加了 GNU 拓展的标准
，可以通过`-std=c11`、`fgnu89-inline`、`fno-gnu89-inline`等选项控制`inline`规则。由于 GCC 从 4.3 版本开始才正式支持 c99 中的`inline`标准[^c99status]，而目前使用的 Cavium SDK 1.7.3 中的 GCC 版本为 4.1.2，即便使用`-std=c99`也仍然是 GNU 的`inline`规则，在海力上编程要注意。

宏与 inline 函数的区别
----------------------

 - 宏只做简单的字符串替换，函数则是参数传递，会进行参数类型检查。
 - 宏不经计算直接替换参数，函数调用则将参数表达式求值后再传递。
 - 宏在编译前进行，函数在编译后执行。

延伸阅读
--------

[内联函数能改善性能么？][1]

[Myth and reality about inline in C99][2]

参考资料
--------

[An Inline Function is As Fast As a Macro][3]

[Inline Functions In C][4]

脚注
----

 [^gccstd]: [GCC Standard][5]

 [^clangstd]: [Clang Compatibility][6]

 [^c99status]: [C99 Status in GCC][7]
 
 [1]: http://www.sunistudio.com/cppfaq/inline-functions.html#[9.3]
 
 [2]: https://gustedt.wordpress.com/2010/11/29/myth-and-reality-about-inline-in-c99/
 
 [3]: https://gcc.gnu.org/onlinedocs/gcc/Inline.html
 
 [4]: http://www.greenend.org.uk/rjk/tech/inline.html
 
 [5]: https://gcc.gnu.org/onlinedocs/gcc/Standards.html
 
 [6]: http://clang.llvm.org/compatibility.html
 
 [7]: https://gcc.gnu.org/c99status.html
