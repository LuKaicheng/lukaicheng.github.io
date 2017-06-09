---
title: 基于标签的人群统计实现分析
date: 2017-06-09 20:15:41
tags: 
- Redis
- BitSet
---

## 问题背景

最近在做项目过程中，碰到了这样一个需求：当前台页面勾选偏好标签的时候，后端服务需要快速给出标签对应的人群数量。如果仅仅只是单个标签的话，那么问题是非常简单的，然而现实是残酷的，我必须考虑基于多个标签勾选的交并集统计情况。

在此，先做一个假设，每个标签后面对应的人群数量是一百万，当然有可能人群会有重叠，而后续的讨论都是基于此数量级展开。

<!--more-->

## 初始想法

考虑到在实际业务场景中用户标识是手机号码，这是一个数字，而且每个用户之间互不相同，为了尽可能满足快速响应以及节省空间的原则，我首先想到了利用**位图**的思想来实现这个需求。在Java中就提供了这样一个类来实现位图操作，它就是**BitSet**，下面是简单的示例：

```java
public class BitSetDemo {

    public static void main(String[] args) {
        BitSet bs1 = new BitSet();
        bs1.set(14);
        bs1.set(20);
        bs1.set(43);
        //输出bs1中设置为true的位数
        System.out.println(bs1.cardinality());
        BitSet bs2 = new BitSet();
        bs2.set(7);
        bs2.set(20);
        bs2.set(33);
        bs2.set(45);
        //输出bs2中设置为true的位数
        System.out.println(bs2.cardinality());
        //bs1和bs2进行交集操作
        bs1.and(bs2);
        //输出bs1和bs2进行交集操作之后设置为true的位数
        System.out.println(bs1.cardinality());
    }

}
```

BitSet可以通过设置具体索引位的值为true来代表存在此数字，而且它还提供了基于位的交、并、差等操作(需要值得注意的是**这些操作会对原有位图进行修改**)。

映射到实际的业务问题，如果把用户标识(将手机号码看成数字)对应的位设置成true来代表存在此用户，那么似乎可以通过操作不同标签对应的位图，从而实现所期望的功能。然而这里还遗漏了一个问题，那就是位图持久化。

## Redis bitmap

正当我苦恼持久化的问题时，无意中翻到Redis居然提供了类似位图操作的命令：**SETBIT**、**BITCOUNT**、**BITOP**， 而且它还提供了RDB和AOF两种持久化机制，另外由于Redis是内存数据库，在命令响应方面也是非常迅速，可以说非常符合我这个问题的场景。

不过，当真正用Redis去实现的时候，又碰到了之前忽视的一个细节：手机号是11号数字，超过了Redis位图所能表示的范围，这一点在**SETBIT**命令的文档中有如下说明：

> The *offset* argument is required to be greater than or equal to 0, and smaller than 2^32 (this limits bitmaps to 512MB)

为了解决这个细节问题，我想出了一个方案：将手机号前两位作为key的一部分，剩下的9位作为值插入到位图里面，由于手机号第一位是1，那么实际上一个标签最多需要10个位图来表示。为了避免用**KEYS**命令扫描获取某个标签所有的key，可以把这些key放到Redis Set中，每次可以先使用**SMEMEBERS**命令获取到所有key。

## Bitmap VS Set

在给他人讲述了我的方案之后，由于此设计在实现上确实有些繁琐复杂，有人提出是否考虑直接使用Set来存储所有的用户号码(他用两个百万随机用户号码的Set做交并操作，效率可以接受)。

于是，我开始对Set方案进行验证，除了处理速度之外，主要关注容量消耗，最终经过测试发现，一百万随机手机号码用Redis Set存储会消耗大概**90M**内存。回过头来，分析我的Bitmap方案，如果要表达9位数字，那么需要2^30，即128M(实际测试下来发现也大致符合这个值)，而一个标签最多需要分10个key，所以最大占据**1280M**。 

诚然Set方案会随着标签下用户群的数量增加容量会随之增加，Bitmap方案的容量较为恒定，然而在这个场景下(最开始假设了一百万的量级，真实场景也大致如此甚至更少)，确实是采用Set会优于Bitmap。

## Cluster & hash tags

尽管决定采用Set方案，在实际编码以及调试过程中，又碰到了Redis的一个限制：如果Redis是采用集群方式部署，假设Set对应的key不属于同个节点，那么就无法透明的支持Set的交并操作。

引用自Redis Cluster规范：

> Redis Cluster implements all the single key commands available in the non-distributed version of Redis. Commands performing complex multi-key operations like Set type unions or intersections are implemented as well as long as the keys all belong to the same node.

这样一来，就需要通过调用者来处理不同标签对应的key分配在不同节点的情况，每次进行多个标签交并操作时，首先需要从多个节点获取到对应标签的用户集合，然后在调用者的程序当中进行实际的集合交并，这样一来会大大增加网络传输量。

幸运的是，Redis提供了一种技巧，可以强制让多个key分配到相同的节点，这种技巧叫做**hash tags**。众所周知，Redis在执行key有关的命令前，先会计算key所对应的slot，不同的key由于计算出来的slot不同，往往就会位于不同的节点。而这个计算有一个短路的地方，假设碰到一个key包含**"{"**和**"}"**，那么只会使用花括号内的子字符串(*第一个花括号内的字符串*)进行slot的计算，这意味着{foo}bar1和{foo}bar2将会位于同一个节点。想要对这个概念有更详细的了解，可以查看[Keys hash tags](https://redis.io/topics/cluster-spec#keys-hash-tags)。 

考虑到可能出现的数据倾斜问题，我们可以将同一大类的标签都指派到相同节点，不同大类的标签指派到不同节点。由于交并操作符合交换律和结合律，那么可以优先计算出相同大类的交并集合，最终通过程序汇总，计算出不同大类的交并集合。这样一来，在考虑数据平衡的情况下，也相应减少网络传输。

## 写在最后

目前这是准备实施的最终方案，当然将来说不定还会想到更好的方式。不过对我自身来说，这次的方案设计、分析、改进，让我更加深刻意识到分析问题要综合考虑项目实际情况、业务细节以及技术限制，任何好的方案都是通过不断权衡和取舍而做出的选择，这个世界上没有银弹。

## 参考

[Redis SETBIT 命令](https://redis.io/commands/setbit)

[Redis BITCOUNT 命令](https://redis.io/commands/bitcount)

[Redis BITOP 命令](https://redis.io/commands/bitop)

[Redis hash tags](https://redis.io/topics/cluster-spec#keys-hash-tags)

