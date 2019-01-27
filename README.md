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

接下来描述了词法分析器（Lexer，用于将字符流转化为token序列，即记号流，用于后续语法分析）的设计以及关键代码：

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
// 看到这里发现对static的用法不是很清楚，详情见此https://www.cnblogs.com/stoneJin/archive/2011/09/21/2183313.html。
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
      // 一行注释读取结束后，因为没有读取到任何有效的token（gettok函数的目的就在于将字符流转换成token序列，然鹅此处没有读取到有效的token），所以再次调用gettok()函数继续读取下一行字符流。
      return gettok();
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

## 语法分析：

使用上述lexer（TODO）产生token序列以后，将token序列通过parser分析成AST。先看一下AST长啥样（下面代码中的一个个class就是AST中不同的结点类，除了ExprAST作为基类不直接使用。）：

```cpp
/// ExprAST - Base class for all expression nodes.
class ExprAST {
public:
  // 虽然只是在析构函数前加了个不起眼的virtual，然而原理却还是要查一下才能明白，就是为了在用基类指针销毁子类对象时可以正确地调用析构函数（子类中不写析构函数，也就是调用基类的析构函数来销毁子类的对象，不加virtual则基类的析构函数不会被调用）：https://blog.csdn.net/starlee/article/details/619827。
  virtual ~ExprAST() {}
};

/// NumberExprAST - Expression class for numeric literals like "1.0".
// 这个子类就比较好理解，构造函数获取一个double类型的值作为该结点的值，但没有写析构函数的原因见上面为什么要在析构函数前加virtual。
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
};

/// VariableExprAST - Expression class for referencing a variable, like "a".
class VariableExprAST : public ExprAST {
  std::string Name;

public:
  VariableExprAST(const std::string &Name) : Name(Name) {}
};

/// BinaryExprAST - Expression class for a binary operator.
class BinaryExprAST : public ExprAST {
  char Op;
  // 使用unique_ptr的原因:请教了大佬，得知auto_ptr属于C++的智能指针的旧式用法，现在已经被unique_ptr取代，并且unique_ptr做得更好。在现在，只要学习unique_ptr和shared_ptr之间的联系区别就可以了。
  std::unique_ptr<ExprAST> LHS, RHS;

public:
  BinaryExprAST(char op, std::unique_ptr<ExprAST> LHS,
                std::unique_ptr<ExprAST> RHS)
    // 之所以要使用std::move()（这里感觉是不得不使用），是因为unique_ptr不可以被复制，而用std::move()可以将unique_ptr的所有权转移，将参数LHS所指向的内存转让给instant variable LHS，也就不是复制了，于是可以完成赋值的操作，但相应的原对象也无法再使用。关于使用std::move()可以参考以下资料：https://blog.csdn.net/CPriLuke/article/details/79462388
    : Op(op), LHS(std::move(LHS)), RHS(std::move(RHS)) {}
};

/// CallExprAST - Expression class for function calls.
class CallExprAST : public ExprAST {
  std::string Callee;
  std::vector<std::unique_ptr<ExprAST>> Args;

public:
  CallExprAST(const std::string &Callee,
              std::vector<std::unique_ptr<ExprAST>> Args)
    : Callee(Callee), Args(std::move(Args)) {}
};

// 函数原型的定义
/// PrototypeAST - This class represents the "prototype" for a function,
/// which captures its name, and its argument names (thus implicitly the number
/// of arguments the function takes).
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;

public:
  PrototypeAST(const std::string &name, std::vector<std::string> Args)
    : Name(name), Args(std::move(Args)) {}

  const std::string &getName() const { return Name; }
};
// 函数的定义
/// FunctionAST - This class represents a function definition itself.
class FunctionAST {
  std::unique_ptr<PrototypeAST> Proto;
  std::unique_ptr<ExprAST> Body;

public:
  FunctionAST(std::unique_ptr<PrototypeAST> Proto,
              std::unique_ptr<ExprAST> Body)
    : Proto(std::move(Proto)), Body(std::move(Body)) {}
};

```

以上就定义完了所有的AST结点类，接下来开始定义分析(parse)函数：

```cpp
// 这里处理了两种情况，一种是读入的token仅仅是一个标识符而不是函数，另一种情况是读入的token是函数调用；这两种情况的区分通过提前读入一个token，看这个token是不是'('而确定。
/// identifierexpr
///   ::= identifier
///   ::= identifier '(' expression* ')'
static std::unique_ptr<ExprAST> ParseIdentifierExpr() {
  std::string IdName = IdentifierStr;

  getNextToken();  // eat identifier.

  if (CurTok != '(') // Simple variable ref.
    return llvm::make_unique<VariableExprAST>(IdName);

  // Call.
  getNextToken();  // eat (
  std::vector<std::unique_ptr<ExprAST>> Args;
  if (CurTok != ')') {
    while (1) {
      if (auto Arg = ParseExpression())
        Args.push_back(std::move(Arg));
      else
        return nullptr;

      if (CurTok == ')')
        break;

      if (CurTok != ',')
        return LogError("Expected ')' or ',' in argument list");
      getNextToken();
    }
  }

  // Eat the ')'.
  getNextToken();

  return llvm::make_unique<CallExprAST>(IdName, std::move(Args));
}

// 上面所展示的对于几种token的分析的结果都可以称为是primary expr（即主要的表达式，如标识符、数字、括号，相对应的有后面扩扩展内容中的user-defined unary operator），下面的代码作为一个统一的入口函数，判断了要解析的token属于哪一类表达式。
/// primary
///   ::= identifierexpr // 标识符开头的表达式
///   ::= numberexpr // 数字……
///   ::= parenexpr // 括号……
static std::unique_ptr<ExprAST> ParsePrimary() {
  switch (CurTok) {
  default:
    return LogError("unknown token when expecting an expression");
  case tok_identifier:
    return ParseIdentifierExpr();
  case tok_number:
    return ParseNumberExpr();
  case '(':
    return ParseParenExpr();
  }
}
```

其实从上面词法分析和语法分析两小节的代码可以看出来，词法分析器（lexer）是作为语法分析器(parser)的一个子模块被调用（getNextToken()）的，只有当语法分析进行到需要token的时候，词法分析器才会进行分析，而不是一开始进行完整的词法分析。也可以用龙书上的一张图解释一下：

![](.\assets\Snipaste_2019-01-22_22-00-30.png)

下面开始较为复杂一点的二元表达式的解析：

如果不考虑优先级的话，二元表达式是有二义性的，也就是`3+4*5`可以理解成`(3+4)*5`，也可以理解成`3+(4*5)`，因此说具有二义性。但是考虑运算优先级的话，应该是`3+(4*5)`才对，所以这里使用了一种叫做“运算符优先级解析（[Operator-Precedence Parsing](http://en.wikipedia.org/wiki/Operator-precedence_parser)）”的方法解决运算优先级的问题。

使用运算符优先级解析，首先就需要一张运算符优先级表：

```cpp
/// BinopPrecedence - This holds the precedence for each binary operator that is
/// defined.
static std::map<char, int> BinopPrecedence;

/// GetTokPrecedence - Get the precedence of the pending binary operator token.
static int GetTokPrecedence() {
  if (!isascii(CurTok))
    return -1;

  // Make sure it's a declared binop.
  int TokPrec = BinopPrecedence[CurTok];
  if (TokPrec <= 0) return -1;
  return TokPrec;
}

int main() {
  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40;  // highest.
  ...
}
```

