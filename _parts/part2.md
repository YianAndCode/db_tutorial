---
title: 第二部分 - 世界上最简单的 SQL 编译器和虚拟机
date: 2017-08-31
---

我们开始复刻 sqlite，sqlite 的“前端”是一个 SQL 编译器，它可以解析字符串然后输出一个内部表示，我们将这个内部表示称之为字节码。

字节码将会被传递给虚拟机，然后被虚拟机执行。

{% include image.html url="assets/images/arch2.gif" description="SQLite Architecture (https://www.sqlite.org/arch.html)" %}

像这样分成两步会有很多好处：
- 降低了每一部分的复杂度（例如：虚拟机无需关心语法错误）
- 允许将常见查询编译一次后就缓存起来以提高性能

按照这个思路，我们来重构一下 main 函数和新增两个关键字：

```diff
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);

-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```

非 SQL 语句，例如 `.exit` 我们称之为“元命令”，这些命令以 `.` 开头，我们只需要判断第一个字符是否为 `.`，然后交给单独的函数来处理这些命令。

接下来，我们添加一个步骤，将用户输入行转换为语句的内部表示。我们这个 sqlite 的前端实现得比较 hacky（译者注：指 demo 级的代码）。

最后，我们将预处理过的语句传递给 `execute_statement`，这个函数最终将会成为我们的虚拟机。

请注意，我们的两个新函数的返回值是枚举值，用于指示是成功或是失败：

```c
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;

typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
```

“未识别的语句（UNRECOGNIZED_STATEMENT）”？有点类似于异常（exception），不过[exception是糟糕的](https://www.youtube.com/watch?v=EVhCUSgNbzo)（并且 C 语言也不支持异常），所以只要可以我就用枚举值来标记。如果我的 switch 语句没有处理每一个枚举值，C 编译器就会警告，所以我们可以不用担心漏了处理函数返回的任一值。尤其是在以后添加了更多的返回值。

`do_meta_command` 是对现有函数的封装，为以后增加更多的命令做准备：

```c
MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
  if (strcmp(input_buffer->buffer, ".exit") == 0) {
    exit(EXIT_SUCCESS);
  } else {
    return META_COMMAND_UNRECOGNIZED_COMMAND;
  }
}
```

我们的“预处理语句”现在只有一个有两个成员的枚举，因为我们允许在语句中传参，所以后续将会包含更多数据。

```c
typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;

typedef struct {
  StatementType type;
} Statement;
```

`prepare_statement` （我们的“SQL 编译器”）目前还不能解析 SQL，它现在只认识两个关键词：
```c
PrepareResult prepare_statement(InputBuffer* input_buffer,
                                Statement* statement) {
  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
    statement->type = STATEMENT_INSERT;
    return PREPARE_SUCCESS;
  }
  if (strcmp(input_buffer->buffer, "select") == 0) {
    statement->type = STATEMENT_SELECT;
    return PREPARE_SUCCESS;
  }

  return PREPARE_UNRECOGNIZED_STATEMENT;
}
```

注意，我们在判断“insert”的时候用的是 `strncmp` 函数，因为 `insert` 关键词后面一般都会跟着要插入的数据（例如： `insert 1 cstack foo@bar.com`）

最后，`execute_statement` 包含了一些桩（译者注：桩程序，指对将要开发的代码的一种临时替代）：
```c
void execute_statement(Statement* statement) {
  switch (statement->type) {
    case (STATEMENT_INSERT):
      printf("This is where we would do an insert.\n");
      break;
    case (STATEMENT_SELECT):
      printf("This is where we would do a select.\n");
      break;
  }
}
```

函数没有返回任何错误码，因为目前为止这个函数也没有什么可以报错的。

改造之后，我们的程序现在可以识别两个新的关键词了！
```command-line
~ ./db
db > insert foo bar
This is where we would do an insert.
Executed.
db > delete foo
Unrecognized keyword at start of 'delete foo'.
db > select
This is where we would do a select.
Executed.
db > .tables
Unrecognized command '.tables'
db > .exit
~
```

我们的数据库框架已经初具雏形了...如果它能存储数据岂不是更棒？在下一章，我们将实现 `insert` 和 `select`，虽然会是世界上最糟糕的数据存储。

下面是本章的代码的完整变更：

```diff
@@ -10,6 +10,23 @@ struct InputBuffer_t {
 } InputBuffer;
 
+typedef enum {
+  META_COMMAND_SUCCESS,
+  META_COMMAND_UNRECOGNIZED_COMMAND
+} MetaCommandResult;
+
+typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
+
+typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;
+
+typedef struct {
+  StatementType type;
+} Statement;
+
 InputBuffer* new_input_buffer() {
   InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
   input_buffer->buffer = NULL;
@@ -40,17 +57,67 @@ void close_input_buffer(InputBuffer* input_buffer) {
     free(input_buffer);
 }
 
+MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
+  if (strcmp(input_buffer->buffer, ".exit") == 0) {
+    close_input_buffer(input_buffer);
+    exit(EXIT_SUCCESS);
+  } else {
+    return META_COMMAND_UNRECOGNIZED_COMMAND;
+  }
+}
+
+PrepareResult prepare_statement(InputBuffer* input_buffer,
+                                Statement* statement) {
+  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+    statement->type = STATEMENT_INSERT;
+    return PREPARE_SUCCESS;
+  }
+  if (strcmp(input_buffer->buffer, "select") == 0) {
+    statement->type = STATEMENT_SELECT;
+    return PREPARE_SUCCESS;
+  }
+
+  return PREPARE_UNRECOGNIZED_STATEMENT;
+}
+
+void execute_statement(Statement* statement) {
+  switch (statement->type) {
+    case (STATEMENT_INSERT):
+      printf("This is where we would do an insert.\n");
+      break;
+    case (STATEMENT_SELECT):
+      printf("This is where we would do a select.\n");
+      break;
+  }
+}
+
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);
 
-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      close_input_buffer(input_buffer);
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```
