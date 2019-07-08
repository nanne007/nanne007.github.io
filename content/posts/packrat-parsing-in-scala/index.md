---
title: "Packrat Parsing in Scala"
date: 2014-05-26T23:01:26+08:00
tags:
- scala
- parsing
---

>  原文 [Packrat Parsing in Scala](http://scala-programming-language.1934581.n4.nabble.com/attachment/1956909/0/packrat_parsers.pdf)

### 解决回溯时重复解析的问题

为 Input 构造一个 Memo Cache，保存不同 Parsers 在 Input 不同位置上的结果；
回溯时，相同的 Parser 在相同 Input 位置上的结果就可以直接从 Memo 中返回。

### 解决直接左递归问题

```
expr = expr '+' '1'
     | '1'
```
1. 完成 seed parsing。
   seed parsing 指的是左递归语法中的基本步骤，比如上述例子中 `expr = '1'`。
   确定 seed parsing 之后，左递归的解析才可以在此基础上推演下去。
2. 在 seed 基础上不断 grow，直到达到终点。

举个例子：上述文法在输入 `1+1+1` 上的表现。
1. input 的初始 Memo 为空，cache = { }
2. 完成 seed parsing 之后，cache = { [expr, 1] = 1 }
3. 再重新用 expr 解析输入，
   这时， `expr = expr '+' '1'` 中右边的 `expr` 解析结果就可以直接从 cache 中返回，
   得到新的结果之后，再次更新 cache = { [expr, 1] = 3 }。
   不断重复这个过程，直到没有新结果，这就是最终 expr 匹配的结果。
   反映在这个例子中是， cache = { [expr, 1] = 5 }。

### 解决直接左递归

```
MEMO: (RULE, Pos) -> MEMOENTRY
MEMOENTRY: (ans: AST or LR, pos: Position)
LR: (detected: Boolean)
```

```
GROW-LR(rule, position, memoEntry, H)
  ... // line A
  while TRUE do
    Pos <- position
    ... // line B
    let ans = EVAL(rule.body)
    if(ans = FAIL or Pos <= memoEntry.pos) then break
    memoEntry.ans <- ans
    memoEntry.pos = Pos
  ... // line C
  Pos <- memoEntry.pos
  return memoEntry.ans
```

```
APPLY-RULE(rule, position)
  let m = MEMO(rule, position)
  if  m = NIL
      then let lr = new LR(FALSE)
           m <- new MEMOENTRY(lr, position)
           MEMO(rule, position) <- m
           // 1. evaluate the body
           let ans = EVAL(rule.body)
           m.ans <- ans
           m.pos <- Pos
           // 3. found a left recursive rule, and a successful seed parse
           if lr.detected = TRUE and ans != FAIL
              then return GROW-LR(rule, position, m, NIL)
              else return ans
      else Pos <- m.pos
           // 2. a left recursive call found
           if m.ans is LR
              m.ans.detected <- TRUE
              return FAIL
           else return m.ans
```

### 解决间接左递归 ###

```
// data structures
LR: (seed: AST, rule: RULE, head: HEAD, next: LR)
HEAD: (rule: RULE, involvedSet, evalSet: set of RULEs)

// global vars
LRStack: a linked list of LR
HEADS: Position -> HEAD

APPLY-RULE(rule, position)
  let m = RECALL(rule, position)
  if  m = NIL
      then
          // create a new LR and push it onto the rule invocation stack
          let lr = new LR(FAIL, rule, NIL, LRStack)
          LRStack <- lr
          // memorize lr, then evaluate R
          m <- new MEMOENTRY(lr, position)
          MEMO(rule, position) <- m
          let ans = EVAL(rule.body)
          //pop lr off the rule invocation stack
          LRStack <- LRStack.next
          m.pos <- Pos
          if lr.head != NIL
             then lr.seed <- ans
                  return LR-ANSWER(rule, position, m)
             else m.ans <- ans
                  return ans
      else
          Pos <- m.pos
          if m.ans is LR
             then SETUP-LR(rule, m.ans)
                  return m.ans.seed
             else return m.ans
```

```
SETUP-LR(rule, lr)
  if lr.head = NIL then lr.head <- new HEAD(rule, {}, {})
  let s = LRStack
  while s.head.rule != lr.head.rule
        do s.head <- lr.head
           lr.head.involvedSet <- lr.head.involvedSet | {s.rule}
           s <- s.next
```

```
GROW-LR(rule, position, memoEntry, head)
  HEADS(position) <- head // line A
  while TRUE do
    Pos <- position
    head.evalSet <- COPY(head.involvedSet) // line B
    let ans = EVAL(rule.body)
    if(ans = FAIL or Pos <= memoEntry.pos) then break
    memoEntry.ans <- ans
    memoEntry.pos = Pos
  HEADS(P) <- NIL // line C
  Pos <- memoEntry.pos
  return memoEntry.ans
```

```
RECALL(rule, position)
  let m = MEMO(rule, position)
  let h = HEADS(position)
  // if not growing a seed parse, just return what is stored
  // in memo table.
  if h = NIL then return m
  // do not evaluate any rule that is not involved in the left recursion
  if m = NIL and rule NOT BELONG TO {h.rule} | h.involvedSet
     then return new MEMOENTRY(FAIL, position)
  if rule BELONG TO h.evalSet
     then h.evalSet <- h.evalSet - rule
          let ans = EVAL(rule.body)
          m.ans <- ans
          m.pos <- Pos
  return m
```

```
LR-ANSWER(rule, position, memo)
  let h = memo.ans.head
  if h.rule != rule
     then return memo.ans.seed
     else memo.ans <- m.ans.seed
          if m.ans = FAIL
             then return FAIL
             else return GROW-LR(rule, position, memo, h)
```

终于明白了大概，看吐了都。

下一步，结合 scala parsing combinator 中的实现再重新梳理一下。
