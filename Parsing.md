# 语法分析

## 波兰表达式

为了验证 `mpc` 的威力，本章我们尝试实现一个简单的语法解析器－－[波兰表达式](en.wikipedia.org/wiki/Polish_notation)，它是我们将要实现的 Lisp 的数学运算部分。波兰表达式也是一种数学标记语言，它的运算符在操作数的前面。

举例来说：

| 普通表达式 | 波兰表达式 |
|-----------------|------------------------|
| `1 + 2 + 6` | `+ 1 2 6` |
| `6 + (2 * 9)` | `+ 6 (* 2 9)` |
| `(10 * 2) / (4 + 2)` | `/ (* 10 2) (+ 4 2)` |

现在，我们需要编写这种标记语言的语法规则。我们可以先用白话文来尝试描述它，而后再将其公式化。

我们观察到，波兰表达式总是以操作符开头，后面跟着操作数或其他的包裹在圆括号中的表达式。也就是说，“程序(`Program`)是由一个操作符(`Operator`)加上一个或多个表达式(`Expression`)组成的”，而 “表达式(`Expression`)可以是一个数字，或者是包裹在圆括号中的一个操作符(`Operator`)加上一个或多个表达式(`Expression`)”。

下面是一个更加规整的描述：

| 名称 | 定义 |
|-----------------|------------------------|
| 程序(`Program`) | `开始输入` --> `操作符` --> `一个或多个表达式(Expression)` --> `结束输入` |
| 表达式(Expression) | `数字、左括号 (、操作符` --> `一个或多个表达式` --> `右括号 )` |
| 操作符(Operator) | `'+'、'-'、'*' 、 '/'` |
| 数字(`Number`) | `可选的负号 -` --> `一个或多个 0 到 9 之间的字符` |

## 正则表达式

我们可以使用上一章学过的符号来表示大多数的规则，但是在表示*数字*和*程序*规则时可能会遇到一些问题。这些规则需要用到一些我们没有讲解过的符号。我们还不知道如何表达开始和结束输入、可选字符、字符范围等。

这些规则由正则表达式(Regular Expression)定义。正则表达式适合定义一些小型的语法规则，例如单词或是数字等。正则表达式不支持多规则，但它清晰且精确地界定了输入是否符合规则。下面是正则表达式的基本规则：

| 语法表示 | 作用 |
|------------|-------------------------|
| `.` | 要求任意字符 |
| `a` | 要求字符 `a` |
| `[abcdef]` | 要求 `abcdef` 中的任意一个 |
| `[a-f]` | 要求按照字母顺序，`a` 到 `f` 中的任意一个 |
| `a?` | 要求 `a` 字符或什么都没有，即 `a` 为可选的 |
| `a*` | 要求有 0 个或多个字符 `a` |
| `a+` | 要求有 1 个或多个字符 `a` |
| `^` | 开始输入 |
| `$` | 结束输入 |

这是我们目前需要的所有正则表达式规则。如果你对正则表达式感兴趣，[这里](http://regex.learncodethehardway.org/)是关于它的一个完整的教程。

在 `mpc` 中，我们需要将正则表达式包裹在一对 `/` 中。例如，Number 可以用 `/-?[0-9]+/` 来表示。

## 安装 mpc

在我们正式编写这个语法解析器之前，正如之前在 Linux 和 Mac 上使用 `editline` 库一样，首先需要包含 `mpc` 的头文件，然后链接 `mpc` 库。

你可以直接使用第四章的代码，并将源文件重命名为 `parsing.c`，然后从 `mpc` 的[项目主页](http://github.com/orangeduck/mpc) 下载 `mpc.h` 和  `mpc.c`，放到和你的 `parsing.c` 同目录下。

在 `parsing.c` 的顶部添加 `#include "mpc.h"` 将 `mpc` 包含进来。将 `mpc.c` 放到命令行中来链接它。另外，在 Linux 上，还要加一个 -lm  参数来链接数学库。

Mac 和 Linux：

`cc -std=c99 -Wall parsing.c mpc.c -ledit -lm -o parsing`

Windows：

`cc -std=c99 -Wall parsing.c mpc.c -o parsing`

> 等一下，包含头文件难道不是用 `#include <mpc.h>`？

*事实上，在 C 语言中有两种包含头文件的方式，一种是用尖括号 `<>`，还有一种是用 `""` 双引号。通常，尖括号用来包含系统头文件如 `stdio.h`，双引号用来包含其他的头文件如 `mpc.h`。*

## 波兰表达式语法解析

把上面描述的规则用正式的语言编写，并在必要的地方使用正则表达式。下面就是波兰表达式的最终语法规则。认真读一下下面的代码，验证是否与之前的描述相符。

```c
/* Create Some Parsers */
mpc_parser_t* Number   = mpc_new("number");
mpc_parser_t* Operator = mpc_new("operator");
mpc_parser_t* Expr     = mpc_new("expr");
mpc_parser_t* Lispy    = mpc_new("lispy");

/* Define them with the following Language */
mpca_lang(MPCA_LANG_DEFAULT,
  "                                                     \
    number   : /-?[0-9]+/ ;                             \
    operator : '+' | '-' | '*' | '/' ;                  \
    expr     : <number> | '(' <operator> <expr>+ ')' ;  \
    lispy    : /^/ <operator> <expr>+ /$/ ;             \
  ",
  Number, Operator, Expr, Lispy);
```

我们还需要使用在第四章中编写的交互式的程序中。将上面的代码放在 `main` 函数的开头处，打印版本信息的代码之前。在 `main` 函数的最后，还应该将使用完毕的解析器删除。只需要将下面的代码放在 `return` 语句之前即可。

/* Undefine and Delete our Parsers */
mpc_cleanup(4, Number, Operator, Expr, Lispy);

> 编译的时候得到了一个错误：`undefined reference to `mpc_lang'`

*注意函数的名字为 `mpca_lang`，`mpc` 后面有个 `a` 字母。*
