-----
title: windbg利用UnhandledExceptionFilter
-----

## 前言
windbg是非常强大的调试工具，具体怎么使用，网上已经有很多的教程了，这里就不复述了。

今天遇到一个崩溃，对于我来说，还是一个新的崩溃，没有这么调试过。所以这里记下来，以便后续回顾。

## 配置
首先，将pdb文件以及源码在windbg中配置好，同时将截取的dmp文件加载到windbg中。

### 64位与32位切换
现在越来越多的电脑使用64位的操作系统，但是大多数的软件还是32位的，64位系统使用wow64来帮助运行32位程序。如果一个64位系统上的32位程序崩溃了，得到的dump在不经过转换的情况下，用windbg也看不见有用的信息。
- 用windbg加载dmp，并配好pdb以及源码
- 使用`.load wow64exts`加载`wow64exts`模块，将64位dump转换成32位的dump
- 输入`!sw`进行转换，命令执行完后提示`Switched to 32bit mode`
- 最后就可以分析这个dump了

## 调试
当然了，可以使用自动分析的命令开始分析`!analyze -v`，结果如下图所示：
![自动分析结果](/images/201608/调试-自动分析.png)

这里我经过仔细分析，这里应该不会出问题的。其次，我们来仔细看下这个线程的堆栈信息，如下图所示：
![线程堆栈1](/images/201608/线程堆栈1.png)

我们可看出，没有可能导致线程崩溃的地方，分析发现这个线程就是一直在等待，并没有发现崩溃的地方。所以，我在这里浪费了很多时间。我们再看其他线程的堆栈，如下图所示：
![其他线程堆栈](/images/201608/其他线程堆栈.png)

在上图中，其他线程中并没有发现我自己写的代码，所以当时就认定出错的线程一定是第一个线程的堆栈。泪流满面，仔细分析得出第一个线程不可能出错，再次仔细看看4号线程的堆栈，其中有个API `UnhandledExceptionFilter`，这个API明显就是和异常有关。下面看下这个API的参数：
``` bash
LONG WINAPI UnhandledExceptionFilter(
  _In_ struct _EXCEPTION_POINTERS *ExceptionInfo
);
```
再看下该参数中结构体的定义：
``` bash
typedef struct _EXCEPTION_POINTERS {
  PEXCEPTION_RECORD ExceptionRecord;
  PCONTEXT          ContextRecord;
} EXCEPTION_POINTERS, *PEXCEPTION_POINTERS;
```
参数一是`EXCEPTION_RECORD`结构体的指针，注意第二个参数是异常上下文信息`CONTEXT`的指针。

所以我们在windbg中切换到4号线程中`~4s`，使用`kb`查看相关API的参数，如下图所示
![UnhandledExceptionFilter](/images/201608/UnhandledExceptionFilter.png)

然后使用`dd 0414f1dc`查看`UnhandledExceptionFilter`结构体参数的内存，如下图所示，红色标记为结构体中第一个参数地址，蓝色标记的是结构体中第二个参数，也就是异常上下文信息。
![参数](/images/201608/参数.png)

这里因为我们要找到异常的上下文信息，也就是出错时线程的堆栈信息，所以我们接着进入到异常上下文中。这里使用`.exr 记录地址`显示一个异常记录的详细内容，这里的命令为`.exr 0414f36c`，结果如下图所示（字段含义，自己可以查查）：
![异常详细信息](/images/201608/异常详细信息.png)

使用`cxr 0414f36c`查看出错上下文信息，命令执行完后就可以直接看到出错的位置了，如下图所示：
![cxr](/images/201608/cxr.png)

最后可以使用命令`k`、`dv`等查看，或者View-Local，查看局部变量，会发现`fileItem`指针为空，这里我们就发现了出错的原因了，也找到了出错的真正的线程上下文了，如下图所示：
![空指针](/images/201608/空指针.png)

## 总结
- 正在被分析的线程，如果没有发现出错的地方，同时，也没有发现和异常相关的API时，需要仔细分析其他线程
- 注意UnhandledExceptionFilter API，清楚该API的参数以及其结构

如果需要此文分析的dmp以及pdb文件，请留言，我会及时回复的。

整理的比较辛苦，如果本文对你有用，请留下你来过的痕迹，转载请注明出处! 2016年8月15号于北京  https://soygrow.github.io