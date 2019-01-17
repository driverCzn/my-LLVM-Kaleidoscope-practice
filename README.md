# my-LLVM-Kaleidoscope-practice
Some notes about [LLVM's Kaleidoscope tutorial](https://llvm.org/docs/tutorial/index.html#kaleidoscope-implementing-a-language-with-llvm).

硬啃龙书感觉不是很有意思，感觉可以跟着LLVM官方的教程尝试一下撸一个玩具编译器出来，了解一下书里讲的每一部分内容具体是怎样的，毕竟编译原理是理论和实践联系紧密的一门课。

## 概览：

首先，目标语言（Kaleidoscope，以下简称K语言）是一种顺序执行的语言，可以定义函数、使用条件语句、进行数学运算等。
语言的数据类型固定为64位的浮点数（也就是64位机器上C中的double）。

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

## 词法分析：

接下来描述了词法分析器（Lexer，用于将字符流转化为一个个token，即记号，用于后续语法分析）的设计以及关键代码：

```cpp
// The lexer returns tokens [0-255] if it is an unknown character, otherwise one
// of these for known things.
enum Token {
  tok_eof = -1,

  // commands
  tok_def = -2,
  tok_extern = -3,

  // primary
  tok_identifier = -4,
  tok_number = -5,
};

static std::string IdentifierStr; // Filled in if tok_identifier
static double NumVal;             // Filled in if tok_number
```

先是设计了一个枚举(enum,enumerate)类型的Token向量，其中每个分量分别是一个特定的token类型（例如tok_identifier)以及对应的值（-4）。还定义了两个变量，一个string型变量IdentifierStr用于存储标识符字符串，一个double型变量NumVal用于存储数值（K语言中只有64位浮点数这一种数值类型）。

数据结构定义好了，至于如何将字符流转化为token，仅用了一个函数gettok()，函数返回值为一个整数，整数值为上述Token类型定义的几个值。

gettok()函数如下：

```cpp
/// gettok - Return the next token from standard input.
static int gettok() {
  static int LastChar = ' ';

  // Skip any whitespace.
  while (isspace(LastChar)) // 直到接收到一个非空格字符时跳出while循环。
    LastChar = getchar();	// 注意getchar的返回类型是int。

  // 标识符处理部分：
  if (isalpha(LastChar)) { // identifier: [a-zA-Z][a-zA-Z0-9]*
    IdentifierStr = LastChar;
    while (isalnum((LastChar = getchar())))
      IdentifierStr += LastChar;

    if (IdentifierStr == "def")
      return tok_def;
    if (IdentifierStr == "extern")
      return tok_extern;
    return tok_identifier; // 如果不是上述两个关键词def和extern，则返回接收到的标识符。
  }
  
  // 数值处理部分：
  if (isdigit(LastChar) || LastChar == '.') { // Number: [0-9.]+
    std::string NumStr;
    do {
      NumStr += LastChar;
      LastChar = getchar();
    } while (isdigit(LastChar) || LastChar == '.');

    NumVal = strtod(NumStr.c_str(), nullptr);
    return tok_number;
  }

  // 注释处理部分：
  if (LastChar == '#') {
    // Comment until end of line.
    do
      LastChar = getchar();
    while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

    if (LastChar != EOF)
      return gettok(); // 一行注释读取结束后，因为没有读取到任何有效的token（gettok函数的目的就在于将字符流转换成token序列，然鹅此处没有读取到有效的token），所以再次调用gettok()函数继续读取下一行字符流。
  }

  // Check for end of file.  Don't eat the EOF.
  // EOF处理。
  if (LastChar == EOF)
    return tok_eof;

  // Otherwise, just return the character as its ascii value.
  // 忽略其他情况，即如果出现其他情况，就将接收到的字符返回（对照Token类型，可以发现就是不处理），并继续读入后续字符。
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}
```

// 一点题外话：这里说完了简单的词法分析，没想到真的如此简单，虽然各种边界情况都没有考虑，但是看着上面花花绿绿的语法高亮，突然感觉这是不是应该叫……词法高亮？毕竟就我日常使用体验来说，大部分的语法高亮功能确实就是“词法高亮”而已。

// 又去查了一下相关信息，确实有不少简易的语法高亮只是“词法高亮”，但是现代的IDE都是用的正确的语法分析，就是要

> 利用语法分析器（parser）把由词法分析生成的token序列分析（parse）成一颗抽象语法树（AST）,然后把AST中出现的符号及其所引用的实体之间的对应关系记录在符号表（symbol table）里。

具体可以参考[这个](https://www.zhihu.com/question/39441111/answer/81626593)。

