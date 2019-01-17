# my-LLVM-Kaleidoscope-practice
Some notes about LLVM's Kaleidoscope tutorial.

硬啃龙书感觉不是很有意思，感觉可以跟着LLVM官方的教程尝试一下撸一个玩具编译器出来，了解一下书里讲的每一部分内容具体是怎样的，毕竟编译原理也算是理论和实际结合的一门课。

首先，目标语言（以下简称K语言）是一种顺序执行的语言，可以定义函数、使用条件语句、进行数学运算等。
语言的数据类型固定位64位的浮点数（也就是C中的double）。

```python
# Compute the x'th fibonacci number.
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2)

# This expression will compute the 40th number.
fib(40)
```

//这个语法……看上去有点像Python啊。

K语言还能调用标准库函数（流啤，文档上说是由于LLVM JIT的存在，虽然JIT是个啥我不是很懂……），总之就是很腻害啦，语言的实用性一下子就提高了很多，看起来不那么像是个玩具了。调用方法为先声明再调用，如下：

```c++
extern sin(arg);
extern cos(arg);
extern atan2(arg1 arg2);

atan2(sin(.4), cos(42))
```

//这个声明的架势看起来又有点像C/C++……

