---
title: 第一部分 - 导论和 REPL
date: 2017-08-30
---

作为一名 web 开发者，我每天都在和关系数据库打交道，但是对于我而言，数据库就像是个黑盒，对此我充满了许多疑问：
- 数据是以怎样的格式存储在内存和磁盘中的？
- 它们又是什么时候从内存持久化到磁盘的？
- 为什么一张表只能有一个主键？
- 怎样回滚一个事务？
- 索引格式是怎样的？
- 什么场景会触发全表扫描？
- 预处理语句的储存格式是什么样的？

换句话说，数据库是怎样**工作**的？

为了找到答案，我正在从零开始编写一个数据库，我将参考 sqlite，因为跟 MySQL 或者 PostgreSQL 比起来，sqlite 更简单（并且整个数据库是保存在单个文件中的！），我更有信心能彻底搞懂这些问题。

# Sqlite

在 sqlite 的官网上已经有很多[关于 sqlite 内部构造的文档](https://www.sqlite.org/arch.html)，并且我也有一本 [SQLite Database System: Design and Implementation](https://play.google.com/store/books/details?id=9Z6IQQnX1JEC)。（译者注：需要用 Android 手机在 Google Play 中才能打开）

{% include image.html url="assets/images/arch1.gif" description="sqlite 架构 (https://www.sqlite.org/zipvfs/doc/trunk/www/howitworks.wiki)" %}

一条查询语句在查询或者修改数据的时候会经过一系列组件的处理，其中**前端**包含了以下部分：
- 分词器（tokenizer）
- 解析器（parser）
- 代码生成器（code generator）

前端接收的输入是一条 SQL 查询，处理后的输出是 sqlite 虚拟机字节码（本质上是已经编译好的可以操作数据库的程序）。

_后端_ 包含了以下部分：
- 虚拟机
- B-tree
- 分页管理器
- OS 接口

**虚拟机** 将前端生成的字节码作为指令，然后对一个或多个表、对索引进行操作，每个表或索引都存储在 B-tree 这种数据结构中。虚拟机本质上是一个庞大的分支处理语句，其中的分支就是字节码指令。

每一个 **B-tree** 都包含了许多节点，每个节点的长度是一个页。B-tree 可以通过向分页管理器发送指令，从磁盘中读取一页数据或者将一页数据写回到磁盘中。

**分页管理器**通过接收到的指令来决定是读取还是写入一页又一页的数据。它负责通过正确的偏移量从数据库文件中读/写数据，并且它会将最近访问的页缓存中和决定何时将这些页回写到磁盘中。

**OS 接口**是根据编译 sqlite 的操作系统不同而不同的“层”（译者注：这个层是向上提供统一的接口，屏蔽不同操作系统的不同底层操作）。在本教程中，我不打算去做跨平台的兼容。

千里之行始于足下，让我们从简单的事情开始迈出第一步：REPL。

## 编写一个简单的 REPL 程序

当你从命令行执行 sqlite 的时候，sqlite 就会进入一个 `读取-执行-打印` 的循环（译者注：REPL 全称为 Read-Eval-Print Loop，即“读取-求值-输出”循环，也被叫做`交互式顶层构件`）：

```shell
~ sqlite3
SQLite version 3.16.0 2016-11-04 19:09:39
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> create table users (id int, username varchar(255), email varchar(255));
sqlite> .tables
users
sqlite> .exit
~
```

要做到这种效果，我们的 main 函数就会有一个无限循环：打印提示、读取一行输入、处理输入的内容：

```c
int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```

定义一个 `InputBuffer` ，里面会封装一些信息，作为我们的代码与 [getline()](http://man7.org/linux/man-pages/man3/getline.3.html)（稍后会详细介绍）交互时的载体。
```c
typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = (InputBuffer*)malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}
```

接下来，定义一个 `print_prompt()` 用于用户输入前打印提示信息。

```c
void print_prompt() { printf("db > "); }
```

然后使用 [getline()](http://man7.org/linux/man-pages/man3/getline.3.html) 来读取用户输入的一行信息：
```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```
`lineptr`：这是一个指针，它指向的地址里的值，是我们用于指向读取缓冲区的变量的指针。如果它是 `NULL`，那么它会由 `getline` 申请内存空间，最后哪怕是执行失败了，也必须由我们来调用 `free` 以释放这部分空间。

`n`：一个指针，表示的是 `*lineptr` 指向的空间的大小

`stream`：要读取的输入流。在这里我们将从标准输入中读取。

`return value`：已读取的字节数，它可能会比 `*n` 要小。

我们调用 `getline` 时会传入 `input_buffer->buffer` 和 `input_buffer->buffer_length`，作为将要读取的内容的存储空间和存放该空间大小的变量，接着我们将它的返回值保存到 `input_buffer->input_length` 中。

`buffer` 在初始时是空的，因此 `getline` 会申请足够的内存空间来存储一整行输入，并且将 `buffer` 指向这段空间。

```c
void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}
```

接下来我们应该定义一个函数，用于释放 `InputBuffer *` 实例和其中的 `buffer` 元素的内存空间（在 `read_input` 中调用 `getline` 时为 `input_buffer->buffer` 申请了内存空间）。

```c
void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}
```

最后，我们来解析和执行命令。在这一章，我们只实现一个退出命令：`.exit`。对于其他输入，我们就打印一个错误信息，然后进入下一圈循环就行。

```c
if (strcmp(input_buffer->buffer, ".exit") == 0) {
  close_input_buffer(input_buffer);
  exit(EXIT_SUCCESS);
} else {
  printf("Unrecognized command '%s'.\n", input_buffer->buffer);
}
```

让我们将代码跑起来看看！
```shell
~ ./db
db > .tables
Unrecognized command '.tables'.
db > .exit
~
```

好了，我们已经得到一个可以正确跑起来的 REPL 程序了。在下一章，我们将开始开发我们的“指令语言”。下面是本章的完整代码：

```c
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}

void print_prompt() { printf("db > "); }

void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}

void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}

int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```
