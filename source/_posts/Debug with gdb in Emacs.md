---
title: Debug with gdb in Emacs
date: 2016-05-31 14:19:18
categories: Coding
tag: [tool]
---


Emacs is a powerful editor and can be configured as a IDE.One of the must-learn is to debug with gdb.

- - - 

### Compile
To make a program debuggable, we need to compile with parameter -g using gcc/g++. 

```
$ g++ -g main.cpp -o main
```

Warning: do not add other optimization parameters for like -O,-O2, otherwise the source file will be changed.

<!--more-->


### Start gdb in Emacs 

1. `M-x gdb`
Then we need to enter detailed instruction in minibuffer, for example  
`Run gdb (like this): gdb -i=mi main`  

2. The Emacs windows should looks like below.
![single](/images/Debug with gdb in Emacs/single.png)![multi](/images/Debug with gdb in Emacs/multi.png ) 
 

Use `M-x gdb-many-windows` to switch and `M-x gdb-restore-windows` to reset layout.


### Breakpoints ###

To set a breakpoint, just click on the fringe.To make the fringe visible, use `M-x fringe-mode`. Or `C-x C-a C-b`(gud-break) to do the job.  

![gud-break](/images/Debug with gdb in Emacs/gud-break.png ) 

To delete a breakpoint , click the fringe again or `C-x C-a C-d` or `gud-remove`  

In the buffer, click or `RET` the breakpoint will move the cursor to the corresponding breakpoint in program while `SPC` could activate or deactivate a breakpoint.

The breakpoints information will be shown in the buffer.

- - - 

### Run ###

After setting the breakingpoints up, click on the ![gdb-go](/images/Debug with gdb in Emacs/gdb-go.jpg) or `gud-go` in the minibuffer or `r` in gud buffer

Whenever the program encounters a breakpoint, it stops right away and a triangle points to the current line in fringe.


#### Run By Step, Run to the cursor

The most useful function in debugging is run by step. There are two kinds of run by step.

1. Next: run a functon call as a statement.  
Instruction: `gud-next`, or click on ![gud-next](/images/Debug with gdb in Emacs/gud-next.jpg ), `C-x C-a C-n`

2. Step: enter a function if encounter one.  
Instruction: `gud-step`, or click on ![gud-step](/images/Debug with gdb in Emacs/gud-step.jpg ) 
, `C-x C-a C-s`

Besides, `gud-finish` could help us jump out of the current function.The key is `C-x C-a C-f`.

What's more, we can run to the cursor line by `gud-until`, `C-x C-a C-u`. Or we drag the statement point ![gud-point](/images/Debug with gdb in Emacs/gud-point.png) to any line, the program will stop there automatically.

- - - 

### Continue ###
After a break, `gud-go` or ![gdb-go](/images/Debug with gdb in Emacs/gdb-go.jpg) continues the program.We could also use `gud-cont`, the key of which is `C-x C-a C-r`

- - - 

### Examine Variables ###
In `gdb-many-windows` mode, the local variable will be displayed in local buffer in the top right corner.  

![local buffer](/images/Debug with gdb in Emacs/local_buffer.png)  

To include more variables into the list, select them and `gud-watch` or `C-x C-a C-w`.  
The observed variables are also displayed in Speedbar, where `SPC` could expand or shrink var.  

![speedbar](/images/Debug with gdb in Emacs/speedbar.png)  

Another helpful feature is `gud-tooltip-mode` which shows the value of var when mouse hangs over it.  

![gud-tooltip](/images/Debug with gdb in Emacs/gud-tooltip-mode.png)  

- - - 

### Some Tricks about GDB###

#### Pretty STL ####
It's a shame that gdb doesn't provide support for <font color="Red">stl</font>. For example, to print a vector, most freak out for below nonsense feedback.

```
(gdb) p foo2
     $2 = {
      <std::_Vector_base<int, std::allocator<int> >> = {
    _M_impl = {
      <std::allocator<int>> = {
        <__gnu_cxx::new_allocator<int>> = {<No data fields>}, <No data fields>}, 
      members of std::_Vector_base<int, std::allocator<int> >::_Vector_impl: 
      _M_start = 0x604160, 
      _M_finish = 0x604188, 
      _M_end_of_storage = 0x6041a0
    }
    }, <No data fields>}
```


To cope with it, a [script](https://github.com/sptzxbbb/gdb) is needed to print pretty stl. We use different commands to do the job. For example, 

```
    (gdb) pvector foo2
    elem[0]: $7 = 0
    elem[1]: $8 = 1
	elem[2]: $9 = 2
	elem[3]: $10 = 3
	elem[4]: $11 = 4
	elem[5]: $12 = 5
	elem[6]: $13 = 6
	elem[7]: $14 = 7
	elem[8]: $15 = 8
	elem[9]: $16 = 9
	Vector size = 10
	Vector capacity = 16
	Element type = std::_Vector_base<int, std::allocator<int> >::pointer
```

#### Print Array ####
To display an array properly, see below.

```
(gdb) p *foo3@10
     $2 = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
(gdb) p *foo3
	 $3 = 0
	 (gdb) p *(foo3 + 2)
	 $4 = 2
```


Supposing I need to see the content of array a[N], suitable command is`p *a@N` or `p (int [N])*a`.In order to show a specific value, type `p *(a + offset)` just like what we do in c/c++.


#### Set value ####

We can change the value of variable while running temporarily.

`set var key = value` or `print key = value`
	 
```
(gdb) p i
    $9 = 3
(gdb) set var i = 2
(gdb) p i
	$10 = 2
```
    



#### Conditional Break ####
`break line-number if condition` When boolean expression is true, breaks at the line-number.

### More about GDB ###

To know more detailed infomaion about gdb, please refer to [GNU GDB Debugger Command Cheat Sheet](http://www.yolinux.com/TUTORIALS/GDB-Commands.html) 
