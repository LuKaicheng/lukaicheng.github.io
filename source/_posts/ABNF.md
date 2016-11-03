---
title: ABNF应知应会
date: 2016-11-02 11:38:02
tags:
---
ABNF英文全称是Augmented Backus-Naur Form，这是一种基于BNF的元语言，在很多的Internet technical Specification中用于定义正式语法。

## 1 规则定义

### 1.1 规则形式

规则定义的形式如下所示

{% blockquote %}
 name = elements crlf
{% endblockquote %}

其中
- name 表示规则名称，对于它的解释可以参考[Rule Naming](#Rule Naming)，
- elements 可以是一个或者多个规则或者最终值的操作组合，关于最终值和操作可以参考[Terminal Values](#Terminal Values)
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

{% blockquote %}
CR = %d13
CR = %x0D
{% endblockquote %}

当碰到需要表示多个字符时，可以用**“.”**来进行串联

{% blockquote %}
false = %x66.61.6c.73.65
{% endblockquote%}

同时ABNF也允许使用双引号来直接说明文字文本

{% blockquote %}
command = "pwd"
{% endblockquote %}

但是需要注意的是，这里字符串是大小写不敏感，且使用的字符集是US-ASCII。因此上面的字符串会匹配"pwd","Pwd","pWd","pwD","PWd","pWD","PwD"和"PWD"。如果我们需要字符串具备大小写敏感特性，那么可以分别指定每个字符，有下面两种方式:

{% blockquote %}
command = %d112.119.100
command = %d112 %119 %109
{% endblockquote %}

## 2.操作
未完待续...

## 参考文档

[RFC 5234](https://tools.ietf.org/html/rfc5234)
[维基百科 扩充巴科斯范式](https://zh.wikipedia.org/wiki/%E6%89%A9%E5%85%85%E5%B7%B4%E7%A7%91%E6%96%AF%E8%8C%83%E5%BC%8F)