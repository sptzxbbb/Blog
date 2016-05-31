---
title: Elisp Notes 1 - Elisp Basics
tags: 
    - Emacs
categories: Coding 
date: 2016-05-26 12:14:58
---


要计算一个elisp表达式的值，首先把光标移动到表达式闭合括号之后, 然后`ctrl+x ctrl+e`调用`eval-last-sexp`。

<!--more-->

## Printing

输出到buffer：`(message "My list is %s %S" "Vicky" (list 8 2 3))`

- `%d` 用于数字
- `%s` 用于字符串
- `%S` 用于任意表达式


## Arithmetic Functions

`(+ 4 5 1)` => 10
`(- 9 2)` => 7
`(* 2 3)` => 6
`(/ 7 2)` => 3 (integer part of quotient)
`(/ 7 2.0)` => 3.5
`(% 7 4)` => 3 (mod, remainder)
`(expt 2 3)` => 8 (power, expoential)


elisp的浮点数必须要有**小数点**和**小数点后的数字**, 如2.0是浮点数, 2.和2则是整数

`(integerp 3.)` => t
`(integerp 3.0)` => nil

名字以`p`结尾(predicate)的函数经常是布尔函数，返回`t`或者`nil`

## Converting String and Numbers

- `(string-to-numbser "3")`
- `(number-to-string 3)`

## True & False

在elisp当中，符号`nil`表示False，同时`nil`等价于空list`()`，因此`()`也表示False；其余都表示True, 习惯上用`t`表示。

`(if [] "yes" "no")` => "yes"
`(if () "yes" "no")` => "no"
`(if 0 "yes" "no")` => "yes"


### Boolean Functions

逻辑运算：

`(and t nil)`
`(or t nil)`
`(and t nil t t t)` ： 接受多个参数

比较数字：

`(< 3 4)`
`(> 3 4)`
`(<= 3 4)`
`(>= 3 4)`
`(= 3 3)`
`(= 3 3.0)` => `t`
`(/= 3 4)` => `t`, 不等于用`/=`

比较字符串：

`(equal "abc" "abc")`
`(strin-equal "abc" "abc")`
`(string-equal "abc" "Abc")` => `nil`
`(string-equal "abc" 'abc')` => `t`, 比较string和symbol。

``abc`表示abc这个symbol，symbol与指针概念相近，指memory的某个地址。

泛式比较用`equal`, 比较两个变量是否有相同`datatype`和`value`。

`(equal 3 3.0)` => `nil`，`type`不同
`(equal '(3 4 5) '(3 4 5))` => `t`
`(equal "e" "e")` => `t`
`(equal 'abc 'abc)` => `t`

不等比较：

使用`(not expr)`。

`(not (= 3 4))` => `t`
`(not (equal 3 4))` => `t`

### Even, odd

- `(= (% n 2) 0)`
- `(= (% n 2) 1)`

## Variables

### Global Variables

elisp使用`setq`来设置变量，变量无需声明，而且是全局性`global`的.


`(setq x 1)`
`(setq a 3 b 2 c 7)`, 进行复数变量赋值。

### Local Variables

要定义`local`变量，使用`let`。例如`(let ((var1 var2…) body)`


```
(let (a b)
(setq a 3)
(setq b 4)
(+ a b)
)
```

还有另外一种语法，不用写若干`setq`，`(let ((var1 val1) (var2 val2)…) body)`

```
(let ((a 3) (b 4))
(+ a b)
)
```

## If Then Else

`if`语句的语法是`(if test body)`。

如果想要else，`(if test true_body false_body)`。

```
(if (< 3 2) (message "yes"))
(if (< 3 2) 7 8)
```

如果不需要else，elisp推荐使用`when`来代替`if`，因为这样语意更清晰。`when`语法是`(when test expr1 expr2...)`, 等同于`(if test (progn expr1 expr2...))`。

## Block of Expressions

代码块的功能在elisp中用`progn`实现。

```
(progn (message "a") (message "b"))
;; is equivalent to 
(message "a") (message "b")
```

`progn`用于把若干表达式塞进一个单一表达式里面，大多数时候用在`if`里面。

```
（if something
    (progn ;true
    ... 
    )
    (
    progn ;false
    ...
    )
)
```

`progn`返回主干里最后一个表达式的值。

### Iteration

elisp大多数循环用while实现，`(while test body)`, body是一或多个表达式。


```
(setq x 0)
(while (< x 4)
  (print (format "day %d" x))
  (setq x (+ 1 x))
  )
```

```
(let ((x 32))
  (while (< x 127)
    (ucs-insert x)
    (setq x (+ x 1))
    )
  )
```

## Defining a Function

基本函数定义的语法是`(defun function_name (param1 param2...) "doc_string" body)`。

```
(defun myFunction () "testing" (message "Yay!"))
```

当一个函数被调用时，body最后一个表达式的值会被返回，也就是elisp函数没有`return`语句。

### Defining Commands

要让一个函数能交互式使用，把`(interactive)`放在doc string之后。

Evaluate以下代码，然后就可以通过`Alt + x`，`execute-exteanded-command`来调用它。

```
(defun yay ()
  "Insert “Yay！” at cursor position."
  (interactive)
  (insert "Yay!")
  )
```

下面是一个基本函数的定义，从`universal-argument`, `ctrl + u`取出1个参数。可以通过`ctrl+u 7 alt+x myFunction`来调用。

```
(defun myFunction (myArg)
  "Prints the argument"
  (interactive "p")
  (message "Your argument is: %d" myArg)
)
```

下面是一个基础函数定义，需要一个`region`作为参数。注意`(interactive "r")`, `r`告诉emacs函数会接受`buffer`的开始和结束位置作为参数。

```
(defun myFunction2 (myStart myEnd)
  "Prints region start and end positions"
  (interactive "r")
  (message "Region begin at: %d, end at: %d" myStart myEnd)
  )
```

总结：
- `(interactive)`从句使得函数可交互式调用和交互式提供参数。
- 含有`(interactive)`的函数是一个可调用**命令**, 可通过`alt+x`, `execute-exteanded-command`来调用。

`(interactive "x...")`需要一个单字母`code`来表明函数会如何从用户取得参数。
`interactive`大约有30个`code`，但以下列出的是最有用的：
- `(interactive)`, 不需要参数的命令
- `(interactive "n")`, `n`代表number，提示用户输入一个数字作为参数（提示字符串可紧跟`"n"`之后作为字符串的一部分。例如，`(interactive "nWhat is your age?")`。）
- `(interactive "s")`，`s`代表string，提示用户输入一个字符串作为参数。
- `(interactive "r")`, `r`代表region, 用于需要两个参数的命令，两个参数分别是当前区域的开始位置和结束位置。这种命令通常会对选定的区域进行操作。

大多数elisp命令的函数定义模板如下：

```
(defun myCommand ()
  "One sentence summary of what this command do.

More detailed documentation here."
  (interactive)
  (let (localVar1 localVar2 …)
    ; do something here …
    ; …
    ; last expression is returned
  )
)
```
