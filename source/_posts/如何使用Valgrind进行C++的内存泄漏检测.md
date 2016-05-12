---
title: (译)如何使用Valgrind进行C/C++的内存泄漏检测
tags: [Memory]
category: Testing
date: 2015-06-02
---

### 简介 ###

在系统编程上，一个重要的方向就是要有效地处理内存问题。你的工作越接近操作系统，你就要面对越多的内存问题。

有时候，这些问题十分琐碎，但很多时候它会成为一个调试内存问题的噩梦。所以，在实际应用中，我们会用到许多调试内存问题的工具。

在本文中，我们会讨论最流行的开源内存管理框架`VALGRIND`。

<!--more-->

>摘自Valgrind.org  
Valgrind 是一个用于构建动态分析工具的测试框架。它包含有一个工具集，每个工具执行某种调试，分析，或者类似的任务，来帮助你改进你的程序。Valgrind是模块化的架构， 因此新的工具能够轻松地被创造而不会妨碍到现有的结构。

许多有用的工具是作为标准包而提供的。

1. `Memcheck`是一个内存错误检测器。它有助于使你的程序，特别是以C和C++写的程序，更加正确。
2. `Cachegrind`是一个缓存和分支预测分析器。它有助于使你的程序运行地更快。
3. `Callgrind` 是一个调用图缓存生成分析器。它与`Cachegrind`有部分功能重叠，但也收集了一个`Cachegrind`没有收集的信息。
4. `Helgrind`是一个线程错误检测器。它有助于使你的多线程程序更加准确。
5. `DRD`也是一个线程错误检测器。它与`Helgrind`相似，但用了不同的分析技术，因此可能找到不同的问题。
6. `Massif`是一个堆分析器。它有助于使你的程序占用更少的内存。
7. `DHAT`是一个不同的堆分析器。它有助于你理解代码块的生存周期、代码块的使用和布局的低效问题。
8. `SGcheck`是一个实验性的工具，用于检测堆和全局数组的溢出。它与`Memcheck`功能互补：`SGcheck`找到`Memcheck`找不到的问题，反之依然。
9. `BBV`是一个ie实验性的`SimPoint`基本块矢量生成器。它对于计算机架构的研究和开发很有帮助。

还有一些对大多数用户没有用处的小工具：`Lackey`是一个示例工具，用于演示仪器基本原理;`Nulgrind`是一个最小化的`Valgrind`，不做分析或者测试，仅用于测试目的。<u>(译者：这里的两个测试原文，前者用的是instrumentation，后者用的是testing，暂时我分不清区别)</u> 



---

### 使用 Valgrind Memcheck ###

memcheck工具使用方式如下：

~~~
    valgrind --tool=memcheck ./a.out
~~~

从上面的命令可以清楚看到，主要的命令是`valgrind`，而我们要用的工具由选项`-tool`来指定。上述的`a.out`是我们想要进行memcheck的可执行文件。

该工具可以检测以下的内存问题：

+ 未初始化内存的使用

+ 对被`free`释放后的内存进行读取/写入

+ 对`malloc`分配的内存块尾部进行读取/写入

+ 内存泄漏

+ 不匹配的使用了malloc/new/new[]和free/delete/delete[]

注意：上述列表并不全面，但包含了该工具能检测到的常见问题。

让我们来一个一个地讨论上述的情景。

注意：下面描述的所有测试代码都应该用gcc并且加上-g选项(用于在memcheck输出中生成行号)进行编译，就像我们之前讨论过的C程序被编译成可执行文件，它需要经历4个不同阶段。

---

### 1.未初始化内存的使用 ###
Code:

~~~c++
#include <stdio.h>
#include <stdlib.h> 

int main(void)
{
    char *p; 

    char c = *p; 

    printf("\n [%c]\n",c); 

    return 0;
}
~~~

在上述的代码中，我们尝试使用未被初始化的指针`p`。  
让我们运行Memcheck看看结果。

~~~
	$ valgrind --tool=memcheck ./val
	==2862== Memcheck, a memory error detector
	==2862== Copyright (C) 2002-2009, and GNU GPL'd, by Julian Seward et al.
	==2862== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==2862== Command: ./val
	==2862==
	==2862== Use of uninitialised value of size 8
	==2862==    at 0x400530: main (valgrind.c:8)
	==2862==
	
	[#]
	==2862==
	==2862== HEAP SUMMARY:
	==2862==     in use at exit: 0 bytes in 0 blocks
	==2862==   total heap usage: 0 allocs, 0 frees, 0 bytes allocated
	==2862==
	==2862== All heap blocks were freed -- no leaks are possible
	==2862==
	==2862== For counts of detected and suppressed errors, rerun with: -v
	==2862== Use --track-origins=yes to see where uninitialized values come from
	==2862== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 4 from 4)
~~~

从上面输出的可以看到，`Valgrind`检测到了未被初始化的变量和给出了一个警告(上面加粗的几行)。(译者：然而我并没有看到**加粗**。)

---

### 2.对被free释放后的内存进行读取/写入 ###
Code：

~~~c++
#include <stdio.h>
#include <stdlib.h> 

int main(void)
{
    char *p = malloc(1);
    *p = 'a'; 

    char c = *p; 

    printf("\n [%c]\n",c); 

    free(p);
    c = *p;
    return 0;
}
~~~

在上述代码片段中，我们释放了一个指针`p`，然后我们尝试再次通过该指针访问值。  
让我们运行memcheck来看一下Valgrind对此情况作何反应。

~~~
    $ valgrind --tool=memcheck ./val
	==2849== Memcheck, a memory error detector
	==2849== Copyright (C) 2002-2009, and GNU GPL'd, by Julian Seward et al.
	==2849== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==2849== Command: ./val
	==2849== 
	
	[a]
	==2849== Invalid read of size 1
	==2849==    at 0x400603: main (valgrind.c:30)
	==2849==  Address 0x51b0040 is 0 bytes inside a block of size 1 free'd
	==2849==    at 0x4C270BD: free (vg_replace_malloc.c:366)
	==2849==    by 0x4005FE: main (valgrind.c:29)
	==2849==
	==2849==
	==2849== HEAP SUMMARY:
	==2849==     in use at exit: 0 bytes in 0 blocks
	==2849==   total heap usage: 1 allocs, 1 frees, 1 bytes allocated
	==2849==
	==2849== All heap blocks were freed -- no leaks are possible
	==2849==
	==2849== For counts of detected and suppressed errors, rerun with: -v
	==2849== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 4 from 4)
~~~

正如上述输出可以看到，该工具检测到了无效读取和给出了警告`Invalid read of size 1`。

---

### 3.对malloc分配的内存块尾部进行读取/写入 ###
Code：

~~~
    #include <stdio.h>
	#include <stdlib.h> 
	
	int main(void)
	{
        char *p = malloc(1);
        *p = 'a'; 
	
	    char c = *(p+1); 
	
	    printf("\n [%c]\n",c); 
	
	    free(p);
	    return 0;
	}
~~~

在上述代码片段中，我们分配了一个字节的内存大小给`p`，但我们将值读入`c`中的时候用的地址是p+1.  
现在我们用Valgrind运行上述代码片。

~~~
	$ valgrind --tool=memcheck ./val
	==2835== Memcheck, a memory error detector
	==2835== Copyright (C) 2002-2009, and GNU GPL'd, by Julian Seward et al.
	==2835== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==2835== Command: ./val
	==2835==
	==2835== Invalid read of size 1
	==2835==    at 0x4005D9: main (valgrind.c:25)
	==2835==  Address 0x51b0041 is 0 bytes after a block of size 1 alloc'd
	==2835==    at 0x4C274A8: malloc (vg_replace_malloc.c:236)
	==2835==    by 0x4005C5: main (valgrind.c:22)
	==2835== 
	
	[]
	==2835==
	==2835== HEAP SUMMARY:
	==2835==     in use at exit: 0 bytes in 0 blocks
	==2835==   total heap usage: 1 allocs, 1 frees, 1 bytes allocated
	==2835==
	==2835== All heap blocks were freed -- no leaks are possible
	==2835==
	==2835== For counts of detected and suppressed errors, rerun with: -v
	==2835== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 4 from 4)
~~~

同样，Valgrind在这种情况下也检测到了无效的读取操作。

---

### 4.内存泄漏 ###

~~~c++
#include <stdio.h>
#include <stdlib.h> 

int main(void)
{
    char *p = malloc(1);
    *p = 'a'; 

    char c = *p; 

    printf("\n [%c]\n",c); 

    return 0;
}
~~~

在这段代码中，我们分配了1字节大小的内存而没有释放。现在让我们运行Valgrind看看会怎样。

~~~
    $ valgrind --tool=memcheck --leak-check=full ./val
	==2888== Memcheck, a memory error detector
	==2888== Copyright (C) 2002-2009, and GNU GPL'd, by Julian Seward et al.
	==2888== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==2888== Command: ./val
	==2888== 
	
	[a]
	==2888==
	==2888== HEAP SUMMARY:
	==2888==     in use at exit: 1 bytes in 1 blocks
	==2888==   total heap usage: 1 allocs, 0 frees, 1 bytes allocated
	==2888==
	==2888== 1 bytes in 1 blocks are definitely lost in loss record 1 of 1
	==2888==    at 0x4C274A8: malloc (vg_replace_malloc.c:236)
	==2888==    by 0x400575: main (valgrind.c:6)
	==2888==
	==2888== LEAK SUMMARY:
	==2888==    definitely lost: 1 bytes in 1 blocks
	==2888==    indirectly lost: 0 bytes in 0 blocks
	==2888==      possibly lost: 0 bytes in 0 blocks
	==2888==    still reachable: 0 bytes in 0 blocks
	==2888==         suppressed: 0 bytes in 0 blocks
	==2888==
	==2888== For counts of detected and suppressed errors, rerun with: -v
	==2888== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 4 from 4)
~~~

输出行(上述加粗的)表明该工具能够检测泄漏的内存。
注意：在这个例子中，我们增加了一个额外选项`-leak-check=full`来获取内存泄漏的详细细节。

---

### 5. 不匹配的使用了malloc/new/new[]和free/delete/delete[]###

~~~c++
#include <stdio.h>
#include <stdlib.h>
#include<iostream> 

int main(void)
{
    char *p = (char*)malloc(1);
    *p = 'a'; 

    char c = *p; 

    printf("\n [%c]\n",c);
    delete p;
    return 0;
}
~~~

上面的代码中，我们使用了malloc()来分配内存，但使用了delete操作符来释放内存。  
注意：使用g++来编译上面的代码，因为delete是在C++中引进的，而要编译C++需要使用g++。  
让我们运行来看一下：

~~~
    $ valgrind --tool=memcheck --leak-check=full ./val
	==2972== Memcheck, a memory error detector
	==2972== Copyright (C) 2002-2009, and GNU GPL'd, by Julian Seward et al.
	==2972== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==2972== Command: ./val
	==2972== 
	
	[a]
	==2972== Mismatched free() / delete / delete []
	==2972==    at 0x4C26DCF: operator delete(void*) (vg_replace_malloc.c:387)
	==2972==    by 0x40080B: main (valgrind.c:13)
	==2972==  Address 0x595e040 is 0 bytes inside a block of size 1 alloc'd
	==2972==    at 0x4C274A8: malloc (vg_replace_malloc.c:236)
	==2972==    by 0x4007D5: main (valgrind.c:7)
	==2972==
	==2972==
	==2972== HEAP SUMMARY:
	==2972==     in use at exit: 0 bytes in 0 blocks
	==2972==   total heap usage: 1 allocs, 1 frees, 1 bytes allocated
	==2972==
	==2972== All heap blocks were freed -- no leaks are possible
	==2972==
	==2972== For counts of detected and suppressed errors, rerun with: -v
	==2972== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 4 from 4)
~~~

从上面的输出行(加粗行)，Valgrind明确的说明`不匹配的使用了free()/delete/delete[]`

你可以在测试代码中尝试使用`new`和`free`来看看会有什么结果。

---

### 6.两次释放内存 ###
Code：
~~~c++
#include <stdio.h>
#include <stdlib.h> 

int main(void)
{
    char *p = (char*)malloc(1);
    *p = 'a'; 

    char c = *p;
    printf("\n [%c]\n",c);
    free(p);
    free(p);
    return 0;
}
~~~
在上面的代码片中，我们两次释放了`p`指针。现在，让我们运行memcheck看看。

~~~
    $ valgrind --tool=memcheck --leak-check=full ./val
	==3167== Memcheck, a memory error detector
	==3167== Copyright (C) 2002-2009, and GNU GPL'd, by Julian Seward et al.
	==3167== Using Valgrind-3.6.0.SVN-Debian and LibVEX; rerun with -h for copyright info
	==3167== Command: ./val
	==3167== 
	
	[a]
	==3167== Invalid free() / delete / delete[]
	==3167==    at 0x4C270BD: free (vg_replace_malloc.c:366)
	==3167==    by 0x40060A: main (valgrind.c:12)
	==3167==  Address 0x51b0040 is 0 bytes inside a block of size 1 free'd
	==3167==    at 0x4C270BD: free (vg_replace_malloc.c:366)
	==3167==    by 0x4005FE: main (valgrind.c:11)
	==3167==
	==3167==
	==3167== HEAP SUMMARY:
	==3167==     in use at exit: 0 bytes in 0 blocks
	==3167==   total heap usage: 1 allocs, 2 frees, 1 bytes allocated
	==3167==
	==3167== All heap blocks were freed -- no leaks are possible
	==3167==
	==3167== For counts of detected and suppressed errors, rerun with: -v
	==3167== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 4 from 4)
~~~

从上面的输出可以看到(加粗行)，该工具检测到了我们对同一指针进行了两次释放内存的操作。

在本文中，我们把注意力放在了内存管理框架Valgrind上，然后使用了memcheck(由此框架提供)来描述它是如何减轻经常与内存打交道的程序员的工作难度。这个工具能检测到许多手动检测不到的内存问题。

译者：附上[原文地址](http://www.thegeekstuff.com/2011/11/valgrind-memcheck/)。  
<u>第一次翻译，水平有限，如有错误，请留言告知</u>

