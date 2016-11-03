---
title: ABNF应知应会
date: 2016-11-02 11:38:02
tags:
---
ABNF英文全称是Augmented Backus-Naur Form，这是一种基于BNF的元语言，在很多的Internet technical Specification中用于定义正式语法。

## 1 规则定义

### 1.1 规则形式

规则定义的形式如下所示

 > name = elements crlf

其中
- name 表示规则名称，对于它的解释可以参考[Rule Naming](#Rule Naming)，
- elements 可以是一个或者多个规则或者最终值的操作组合，关于最终值可以参考[Terminal Values](#Terminal Values)
- crlf 行结束标志(回车换行)

为了视觉效果，规则定义需要左对齐，如果碰上一个规则需要多行的情况，那么接下来的行需要缩进，而它们对齐和缩进的基准是ABNF规则的第一行。

### <span id="Rule Naming">1.2 规则名称</span>
规则名称由字母开头，后续可以包含字母、数字和连字符(减号)组合的序列,其中需要注意的是规则名称**不区分大小写**。而且不像BNF，用尖括号(**<**，**>**)包围规则名称并不是必需的。

### <span id="Terminal Values">1.3 最终值</span> 
所有规则最终都会由最终值来解释，而所谓的最终值是由一个指定的基数再结合一个或者多个数值字符来指定。当前已经定义的基数有三种：

- b： 二进制 binary
- d： 十进制 decimal
- x： 十六进制 hexadecimal

以回车CR为例，下面的规则分别采用十进制和十六进制为基数

> CR = %d13
> 
> CR = %x0D

当碰到需要表示多个字符时，可以用**“.”**来进行串联

> false = %x66.61.6c.73.65

同时ABNF也允许使用双引号来直接说明文字文本

> command = "pwd"

但是需要注意的是，这里字符串是大小写不敏感，且使用的字符集是US-ASCII。因此上面的字符串会匹配"pwd","Pwd","pWd","pwD","PWd","pWD","PwD"和"PWD"。如果我们需要字符串具备大小写敏感特性，那么可以分别指定每个字符，有下面两种方式:

> command = %d112.119.100
> 
> command = %d112 %d119 %d109

## 2.操作

### 2.1 级联 R1 R2

可以将已经定义的规则和最终值按顺序列出来，元素之间用空白字符来进行区分。
> foo = %x61
> 
> bar = %x62
> 
> mumble = foo bar foo

在上面的示例里规则foo匹配a，bar匹配b，mumble将匹配aba。

### 2.2 选择 R1 / R2
可以通过在规则之间插入/，让规则变成可选。
> ws = %x20 / %x09 / %x0A / %x0D

在上面的示例里，规则ws会匹配空格、制表符、换行、回车。

### 2.3 增量选择 R1 =/ R2
有时候，我们可能希望有一种增量的方式，可以在旧规则里添加新的功能可选项，这个时候增量选择就比较适用，它通过**=/**来将新规则变成旧规则的可选项之一。
> ruleset = alt1 / alt2
> 
> ruleset =/ alt3
> 
> ruleset =/ alt4 / alt5

最终ruleset等价于下面所示：
> ruleset = alt1 / alt2 / alt3 / alt4 / alt5

### 2.4 值范围
  
通过使用连字符(-),ABNF还可以实现指定一个范围的值。

> DIGIT = %x30-39

上面的示例规则，实际上等价于下面的规则

> DIGIT = "0" / "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9"

### 2.5 序列组合 (R1 R2)

用括号将连个规则包围起来，这样的组合会被当成单个元素，对于一些优先级比较容易混淆的情况尤其推荐使用它。

> group = elem (foo / bar) blat

上面的示例会匹配 elem foo blat 或者 elem bar blat，但是如果我们不使用括号的话

> group = elem foo / bar blat

由于操作符优先级的关系，其实group会匹配elem foo或bar blat。

### 2.6 不定量重复 \*Rule

我们可以在规则的前面添加\*，来表示重复这个规则，完整的形式是m\*nRule。其中m和n都是可选的，m表示至少重复多少次，n表示最多重复多少次。两者默认的值分别是0和无穷大，所以\*Rule表示允许任意次数的重复包括零次。1\*Rule表示规则至少重复一次，1\*2Rule表示规则重复一次或两次,3\*3Rule表示规则必须且仅允许重复3次。

### 2.7 定量重复 nRule

除了不定量重复之外，ABNF也允许指定次数的重复，完整形式是nRule，其实等价于n*nRule。运用这个方式，2DIGIT就表示2位数字，3ALPHA表示长度为3的字符串。

### 2.8 可选序列 [Rule]
可以使用方括号来圈定一个可选序列

> rule = [foo bar]

等价于

> rule = *1(foo bar)

### 2.9 注释

对于规则的说明，也提供了注释方式，以分号**;**开始，并到此行的结束。

> false = %x66.61.6c.73.65   ; false

未完待续...


## 参考文档

[RFC 5234](https://tools.ietf.org/html/rfc5234)
[维基百科 扩充巴科斯范式](https://zh.wikipedia.org/wiki/%E6%89%A9%E5%85%85%E5%B7%B4%E7%A7%91%E6%96%AF%E8%8C%83%E5%BC%8F)