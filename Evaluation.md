# 第零七章 • 计算

## 树型结构

现在，我们可以读取输入，解析并得到表达式的内部结构。但是并不能对它进行计算。本章我们将编写代码，对表达式的内部结构进行计算求值。

所谓的内部结构就是上一章中我们打印出来的内容。它被称为*抽象语法树*(Abstract Syntax Tree，简称 AST)。它用来表示用户输入的表达式的结构。操作数和操作符等需要被处理的实际数据都位于叶子节点上。而非叶子节点上则包含了遍历和求值的信息。

在做解析和求值之前，我们先来看一下数据结构具体的内部表示。在 `mpc.h` 中，可以找到 `mpc_ast_t` 类型的定义，这里面就是我们解析表达式得到的数据结构。

```c
typedef struct mpc_ast_t {
  char* tag;
  char* contents;
  mpc_state_t state;
  int children_num;
  struct mpc_ast_t** children;
} mpc_ast_t;
```

下面来逐一的看看结构体各个字段的含义。

第一个为 `tag` 字段。当我们打印这个树形结构时，`tag` 就是在节点内容之前的信息，它表示了解析这个节点时所用到的所有规则。例如：`expr|number|regex`。

`tag` 字段非常重要，因为它可以让我们知道创建节点时需要用到的规则。

第二个是 `contents` 字段，它包含了节点中具体的内容，例如 `'*'`、`'('`、`'5'`。你会发现，对于表示分支的非叶子节点，这个字段为空。而对于叶子节点，则包含了操作数和操作符。

下一个字段是 `state`。这里面包含了解析器发现这个节点时所处的状态，例如在代码中的行数和列数等。在我们的程序中不会用到这个字段。

最后的两个字段 `children_num` 和 `children` 帮助我们来遍历抽象语法树。前一个字段告诉我们有多少个孩子节点，后一个字段是包含这些节点的数组。

其中，`children` 字段的类型是 `mpc_ast_t**`。这是一个二重指针类型。实际上，它并不像看起来那么可怕，我们会在后面的章节中详细解释它。现在你只需要知道它是孩子节点的列表即可。

我们可以对 `children` 使用数组的语法，在其后使用 `[x]` 来获取某个下标的值。比如，可以用 `children[0]` 来获取第一个孩子节点。注意，在 C 语言中数组是从 `0` 开始计数的。

因为 `mpc_ast_t*` 是指向结构体的指针类型，所以获取其字段的语法有些许不同。我们需要使用 `->` 符号，而不是 `.` 符号。

```c
/* Load AST from output */
mpc_ast_t* a = r.output;
printf("Tag: %s\n", a->tag);
printf("Contents: %s\n", a->contents);
printf("Number of children: %i\n", a->children_num);

/* Get First Child */
mpc_ast_t* c0 = a->children[0];
printf("First Child Tag: %s\n", c0->tag);
printf("First Child Contents: %s\n", c0->contents);
printf("First Child Number of children: %i\n",
  c0->children_num);
```

## 递归

树形结构有个奇怪的地方，它是自身重复的。树的每个孩子节点都是树，每个孩子节点的孩子节点也是树。
