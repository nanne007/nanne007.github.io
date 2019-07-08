---
title: "[Structured Programming with Goto Statement] 笔记"
date: 2014-10-10T23:04:30+08:00
tags:
- goto
---

[Structured Programming with `goto` Statements](http://cs.sjsu.edu/~mak/CS185C/KnuthStructuredProgrammingGoTo.pdf) 是 Donald E. Knuth 于 1974 年的写作的一篇文章。
原始的 pdf 格式效果不好，互联网上存在一个 [html 版本](http://www.kohala.com/start/papers.others/knuth.dec74.html)。

文章试图以一种易于理解的方式说明 `goto` 的正反两面。
同时，为了说明 `goto` 不同的使用情况，文章还讨论了很多例子程序。

### goto 历史

从60年代开始，程序员开始避免使用 `goto`，尤其是在 Edsger Dijkstra 的 [A Case against the GO TO Statement](http://www.cs.utexas.edu/~EWD/transcriptions/EWD02xx/EWD215.html)之后，`goto` 是有害的这一论调已经达成了广泛的共识。
然而程序员和语言的设计者仍然会需要 "goes/jump to" 这种类似 `goto` 的编程方式。


### 一个搜索的例子

一个典型的程序：假设我们想搜索一个表 `A[1] ... A[m]`，其中的值各不相同；如果值 `x` 不在表中出现，我们想把它插入到其中。
另外，还假设另一个有数组 `B`，其中 `B[i]` 记录了我们搜索 `A[i]` 的次数。
我们可能会通过如下的程序来解决这个问题：

_Example 1_

```
for i:=1 step 1 until m do
  if A[i] = x then goto found fi;
not found:
  i:= m + 1;
  m := i;
  A[i] := x;
  B[i] := 0;
found:
  B[i] := B[i] + 1;
```

如果禁止使用 `goto`，可以这样做（把 for 循环展开为赋值和 while 循环）：

_Example 1a_

```
i := 1;
while i <= m and A[i] != x do
  i := i + 1;
if i > m then
  m := i;
  A[i] := x;
  B[i] := 1;
else
  B[i] := B[i] + 1;
fi;
```

下面列出的方法通过修改数据结构达成了更好的结果：

_Example 2_

```
A[m+1] := x;
i:= 1;
while A[i] != x do
  i := i + 1;
if i > m then
  m := i;
  B[i] := 1;
else
  B[i] := B[i] + 1;
fi;
```

### 效率

经验表明：非 IO-bound 程序源代码的 3% 决定了其绝大部分的运行时间。内循环的运行速度影响着整个程序的运行速度。

**尽可能的去修改程序和数据结构来优化内循环的行为**。这么做出于以下三个目的：
1. 优化过程不会花费很长时间，因为内循环一般都很简短。
2. 取得的效果是可观的。
3. 程序的其他部分可以不那么有效率，以换取可读性和调试的简便。

_Example 1_ 中，就可能将循环从 `1` 到 `m` 变为从 `m` 到 `1`，因为与零比较来的更简单。
_Example 2_ 可以通过加速来提高效率。

_Example 2a_

```
A[m+1] := x;
i := 1;
goto test;
loop:
  i := i + 2;
test:
  if A[i] = x then goto found fi;
  if A[i+1] != x then goto loop fi;
  i := i + 1;
found:
  if i > m then
    m := i;
    B[i] := 1;
  else
    B[i] := B[i] + 1;
  fi;
```

### 错误退出

上述的例子漏掉了一个重要的问题，没有对 `m` 做检查（`m := i` 赋值可能出现溢出）。
软件工程中，**对数据有效性的检查是非常重要的。**
现在说明这个情况，是因为 _error exit_ 是一类重要的 `goto`。

有时候，错误退出会跨越多层控制栈，而最优雅的方式是直接使用 `goto` 或与之等效的方法；
这样，程序的中间层就可以假设没有错误发生。


### 下标检查

在测试期，把下表检查编译成机器码，而在真正的运行期，又取消或这抑制这样的检查；
这就好比，一个水手在训练的时候穿着他的救生服，而下水远航时，却把救生服落在一边。
**下标检查是很有必要的，但不是必须的。**
比如，在 for 循环里面做检查增加了程序负担，因为非常确定在没有到达循环结束之前，下标都是有效的。
拿水手类比，如果水手知道救生服很贵，或者他本身是一个非常优秀的水手，那相比其他危险，救生服就不那么重要了。

### 再一个例子：哈希

假设上述的例子基于标准的哈希技术：`h(x)` 是一个产生值在 `1` 和 `m` 之间的哈希函数，并且 `x != 0`，
并且 `m` 大于表中条目的个数以保证表中至少含有一个“空位置”，“空位置”用零表示。

_Example 3_

```
i := h(x);
while A[i] != 0 do
  begin
    if A[i] = x then goto found fi;
    i := i - 1;
    if i = 0 then i := m fi;
  end;
not found:
  A[i] := x;
  B[i] := 0;
found:
  B[i] := B[i] + 1;
```

这样一来，_Example 2_ 的小技巧就不适用了。
然而，可以用 _Example 1a_ 的方法来消除 `goto`：`while A[i] != 0 and A[i] != x do ...`，
之后，再做一个简单的条件终止检测。

```
i := h(x)
while A[i] != 0 and A[i] != x do
  begin
    i := i - 1;
    if i = 0 then i :=m fi;
  end;
if A[i] = 0 then
  A[i] := x;
  B[i] := 1;
else
  B[i] := B[i] + 1;
fi;
```

这个版本可能更容易阅读，然而，相比 _Example 3_，它做了多余的条件终止检测；
程序的关键部分，应当避免这种情况。

为什么要担心这个多余的检测呢？毕竟它并不在 while-loop 里面。
这是因为，while-loop 并不真的会表现为一个循环：对于一个合适的 `h` 和 `m`，操作 `i := i - 1` 会很少执行，
相较而言，这个多余的检测就不能再被忽略了，尤其是当这个搜索代码在整个程序起主导作用。

多数情况下，由 `A[i] = x` 终止循环发生的更频繁，洞察了这一行为，代码可以变换如下：

_Example 3a_

```
i := h(x)
while A[i] != x do
  begin
    if A[i] = 0 then
      A[x] := x;
      B[i]: = 0;
      goto found;
    fi;
    i := i - 1;
    if i = 0 then i := m fi;
  end
found: B[i] := B[i] + 1;
```

这里就是一个延时优化的例子：当获得了更多关于程序行为的信息，再去优化程序。

如果 while-loop 变成了一个影响整个程序的内循环，那情况又变了。
多余的那个条件终止检测就可以忽略不计，而 `if i = 0` 就影响颇大，我们可能想避免这个几乎总是为 false 的检测。
加上这个假设，程序又应该加以如下的变换：

_Example 3b_

```
A[0] = 0 // add a new element
i := h(x)
while A[i] != x do
    if A[i] != 0 then
      i := i - 1
    else if i = 0 then
      i := m
    else
      A[x] := x;
      B[i]: = 0;
      goto found;
    fi;
  end
found: B[i] := B[i] + 1;
```

### 一个关于文本扫描的例子

_Example 4_

```
x := read-char;
if x = slash then
  x:= read-char;
  if x = slash then
    carriage-return;
    goto char-processed;
  else
    tabulate;
  fi;
fi;
write-char(x);
if x = period then
  write-char space fi;
fi;
char-processed:
```


### 树搜索

_Example 5_

```
compare:
  if A[i] < x then
    if L[i] != 0 then
      i := L[i];
      goto compare;
    else
      L[i] := j;
      goto insert;
    fi;
  else
    if R[i] != 0 then
      i := R[i];
      goto compare;
    else
      R[i] := j;
      goto insert;
    fi;
  fi;
insert:
  A[j] := x;
L[j] := 0;
R[j] := 0;
j := j+1;
```

### 系统化的消除 goto

大多数情况下，程序结构可以很好地用 _顺序组合_，_条件_，_迭代_以及 _case 多路分支_ 来描述;
无约束的 `goto` 使得程序结构难以描述，并且往往是程序形式化出错的症状。
但 `goto` 存在与否并不是关键点，重要的是冰山下的程序结构。
人们总是过于强调去消除 `goto` ，而忽略了真正的问题，
程序员们真正想要的是将程序组织成易于理解的代码，而避免那种会搞乱程序的东西只是会有利于达到这一目的而已。
构造一个程序而无需想到要使用 `goto` 才是程序员真正要做到的。

### 事件表征

通过事件来代替 `goto label`

```
A)      loop until <event>1 or ... or <event>n:
                  <statement list>0;
        repeat;
        then <event>1 => <statement list>1;
                        . . .
             <event>n => <statement list>n;
        fi;
B)      begin until <event>1 or ... or <event>n;
                  <statement list>0;
        end;
        then <event>1 => <statement list>1;
                        . . .
             <event>n => <statement list>n;
        fi;
```

_Example 5b_

```
loop until left leaf hit or right leaf hit:
    if A[i] < x
    then if L[i] != 0 then i := L[i];
        else left leaf hit fi;
    else if R[i] != 0 then i := R[i];
        else right leaf hit fi;
    fi;
repeat;
then left leaf hit => L[i] : = j;
          right leaf hit => R[i]: = j;
fi
A[j] := x; L[j] := 0; R[j] := 0; j := j+1;
```

_Example 5c_

```
loop until leaf replaced:
    if A[i] < x
    then if L[i] != 0 then i := L[i]
        else L[i] := j; leaf replaced fi;
    else if R[i] != 0 then i := R[i]
        else R[i] := j; leaf replaced fi;
    fi;
repeat
A[j] := x; L[j] := 0; R[j] := 0; j := j+1;
```

如果事件还可以扩展从而带有参数，_Example 1 ~ 3_ 就可以通过事件的方式来抽象化：

```
begin until found:
    search table for x and insert it if not present;
end;
then found (integer j) => B[j] := B[j]+1;
fi;
```

这和回调机制类似。

### 消除递归 ###

以中序遍历作为例子。

_Example 6：基本的递归方法_

```
procedure treeprint(t); integer t; value t;
  if t != 0 then
    treeprint(L[t]);
    print(A[t]);
    treeprint(R[t]);
  fi;
```

_Example 6a：利用 goto 消除函数尾的递归调用_

```
procedure treeprint(t); integer t; value t
L: if t != 0 then
      treeprint(L[t]);
      print(A[t]);
      t := R[t];
      go to L;
    fi;
```

_Example 6b: 消除 goto_

```
procedure treeprint(t); integer t; value t;
  loop while t != 0:
    treeprint (L[t]);
    print (A[t]);
    t := R[t];
  repeat;
```

_Example 6c: 再消除内部的递归_

```
procedure treeprint(t); integer t; value t;
  begin
    integer stack S; S := empty;
L1: loop while t != 0:
      S <= t; t := L[t]; go to L1;
L2:   t <= S;
      print(A[t]);
      t := R[t];
    repeat
    if nonempty(S) then go to L2 fi;
  end
```

转换过程中，使用了两个重要的消除原则：
1. 对于尾调用来说，只需简单的 goto 到其开始执行位置。从而，有 _Example 6_ 到 _Example 6a_ 的转换。
2. 对于内嵌的递归调用，保存每一次递归的局部变量，到达递归深度之后，再从底向上执行。从而，有 _Example 6b_ 到 _Example 6c_ 的转换。

可以发现，再进一步对 _Example 6c_ 进行 `goto` 消除，可以实现基于栈的中序遍历，所达到的执行效果是一样的。


### 程序转换

程序设计中，一种可行的解决方式是：首先完成一个结构优美但效率可能很低的程序 P；然后等价的转换程序，直至达到一定的效率，变成最终的程序 Q，当然这些转换过程必须是直观且可信的。这个过程之所以可行，是因为程序员知道每一步转换应该优化程序的哪个部分，以及应该优化到什么程度，这一点和延迟优化也是有关系的。

目前为止，这篇文章已经提到了几种转换：
1. 加速循环过程（_Example 2a_）
2. 把尾调用变成 `goto` 或其等价的形式（_Example 6a_）
3. 使用栈来模拟递归（_Example 6c_）

另外一种著名的方法是 _将不变的子表达式从循环中移除_。
