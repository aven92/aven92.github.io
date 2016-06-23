---
layout: post
title: C语言side effect和sequence point
categories: C
---

<!--more-->

  C 语言中副作用 ( side effect ) : 是指对数据对象或者文件的修改。例如，语句 var = 99 ; 的副作用是把 var 的值修改成 99 。对表达式求值也可能产生副作用，例如，对表达式求 se = 100;
求值所产生的副作用就是 se 的值被修改成 100。
  序列点 ( sequence point ) : 是指程序运行中的一个特殊的时间点，在该点之前的所有副作用已经结束，并且后续的副作用还没发生。C 语句结束标志——分号 ( ; ) 是序列点。标准规定，在两个序列点之间，一个对象所保存的值最多只能被修改一次。

  C 语句中由赋值、自增或者自减等引起的副作用在分号之前必须结束。我们以后会说到一些包含序列点的运算符。任何完整表达式 ( full expression ) 运算结束的那个时间点也是序列点。所谓完整表达式，就是说这个表达式不是子表达式。而所谓的子表达式，则是指表达式中的表达式。例如：f = ++e % 3 这整个表达式就是一个完整表达式。这个表达式中的 ++e、3 和 ++e % 3 都是它的子表达式。

有了序列点的概念，我们下面来分析一下一个很常见的错误 :

    int x = 1, y ;
    y = x++ + x++ ;

这里 y = x++ + x++ 是完整表达式，而 x++ 是它的子表达式。这个完整表达式运算结束的那一点是一个序列点，int x = 1, y ; 中的 ; 也是一个序列点。也就是说 x++ + x++ 位于两个序列点之间。标准规定，在两个序列点之间，一个对象所保存的值最多只能被修改一次。但是我们清楚可以看到，上面这个例子中，x 的值在两个序列点之间被修改了两次。这显然是错误的！这段代码在不同的编译器上编译可能会导致 y 的值有所不同。比较常见的结果是 y 的值最后被修改为 2 或者 3 。在此，我不打算就这个问题作更深入的分析，各位只要记住这是错误的，别这么用就可以了。有兴趣的话，可以看看以下列出的相关资料。

C 语言标准对副作用和序列点的定义如下：
Accessing a volatile object, modifying an object, modifying a file, or calling a function that does any of those operations are all side effects, which are changes in the state of the execution environment. Evaluation of an expression may produce side effects. At certain specified points in the execution sequence called sequence points, all side effects of previous evaluations shall be complete and no side effects of subsequent evaluations shall have taken place.
翻译如下：

访问易变对象，修改对象或文件，或者调用包含这些操作的函数都是副作用，它们都会改变执行环境的状态。计算表达式也会引起副作用。执行序列中某些特定的点被称为序列点。在序列点上，该点之前所有运算的副作用都应该结束，并且后继运算的副作用还没发生。

顺序点有:

1. The point of calling a function, after evaluating its arguments.
2. The end of the first operand of the && operator.
3. The end of the first operand of the || operator.
4. The end of the first operand of the ?: conditional operator.
5. The end of the each operand of the comma operator.
6. Completing the evaluation of a full expression. They are the following:
7. Evaluating the initializer of an auto object.
8. The expression in an ‘ordinary’ statement—an expression followed by semicolon.
9. The controlling expressions in do , while , if , switch or for statements.
10. The other two expressions in a for statement.
11. The expression in a return statement.

编译时可以加上 " -Wsequence-point " 让编译器帮我们检查可能的关于检查点的错误。

    $ vim test_sequence_point.c 
    ＃include <stdio.h> 
    int main() {
        int i = 12 ;
        i = i-- ;
        printf( "the i is %d/n" , i ) ;
        return 0 ;   
    }

    $ gcc -Wsequence-point test_sequence_point.c
    test_sequence_point.c: In function `main':  
    test_sequence_point.c:10: warning: operation on `i' may be undefined

