---
layout: post
title: "Gcc命令"
date:    2024-08-20
tags: [C/C++]
comments: true
author: mazezen
---


> 简介:
> GCC 全称:GNU Compiler Collection. 是 GNU 工具链的主要组成部分，是 GPL 和 LGPL 许可证发布的程序语言编译器自由软件，由 Richard Stallman 于 1985 年开始开发。

GCC 原名为 GNU C语言编译器，它原本只能处理 C 语言，但现在的 GCC 不仅可以编译 C、C++ 和 Objective-C，还可以通过不同的前端模块支持各种语言，包括 Java、Fortran、Ada、Pascal、Go 和 D 语言等等。

GCC 的编译过程分为四个阶段：预处理（Pre-Processing）、编译（Compiling）、汇编（Assembling）以及链接（Linking）。

#### 语法：
```shell
gcc [options] file...
```
##### 选项:
- <options> 传递给预处理器（preprocessor）。
- -Wl,<options> ：将逗号分隔的 <options> 传递给链接器（linker）。
- -Xassembler <arg> ：将 <arg> 传递给汇编器（assembler）。
- -Xpreprocessor <arg> ：将 <arg> 传递给预处理器（preprocessor）。
- -Xlinker <arg> ：将 <arg> 传递给链接器（linker）。
- -save-temps ：不用删除中间文件。
- -save-temps=<arg> ：不用删除指定的中间文件。
- -no-canonical-prefixes ：在构建其他 gcc 组件的相对前缀时，不要规范化路径。
- -pipe ：使用管道而不是中间文件。
- -time ：为每个子流程的执行计时。
- -specs=<file> ：使用 <file> 的内容覆盖内置规范。
- -std=<standard> ：假设输入源为 <standard>。
- --sysroot=<directory> ：使用 <directory> 作为头文件和库的根目录。
- -B <directory> ：将 <directory> 添加到编译器的搜索路径。
- -v ：显示编译器调用的程序。
- -### ：与 -v 类似，但引用的选项和命令不执行。
- -E ：仅执行预处理（不要编译、汇编或链接）。
- -S ：只编译（不汇编或链接）。
- -c ：编译和汇编，但不链接。
- -o <file> ：指定输出文件。
- -pie ：创建一个动态链接、位置无关的可执行文件。
- -I ：指定头文件的包含路径。
- -L ：指定链接库的包含路径。
- -shared ：创建共享库/动态库。
- -static ：使用静态链接。
- --help ：显示帮助信息。
- --version ：显示编译器版本信息。

#### 示例
##### 阶段编译
假设有文件 hello.c，内容如下：

#include <stdio.h>
 int main(void)
 {
     printf("Hello, world\n");
     return 0;
 }
编译 hello.c，默认输出 a.out
```shell
gcc hello.c
```

编译 hello.c 并指定输出文件为 hello
```shell
gcc hello.c -o hello
```

只执行预处理，输出 hello.i 源文件
```shell
gcc -E hello.c -o hello.i
```

只执行预处理和编译，输出 hello.s 汇编文件
```shell
gcc -S hello.c
```

也可以由 hello.i 文件生成 hello.s 汇编文件
```shell
gcc -S hello.i -o hello.s
```

只执行预处理、编译和汇编，输出 hello.o 目标文件
```shell
gcc -c hello.c
```

也可以由 hello.i 或 hello.s 生成目标文件 hello.o
```shell
gcc -c hello.i -o hello.o
gcc -c hello.s -o hello.o
```

由 hello.o 目标文件链接成可执行文件 hello
```shell
gcc hello.o -o hello
```
