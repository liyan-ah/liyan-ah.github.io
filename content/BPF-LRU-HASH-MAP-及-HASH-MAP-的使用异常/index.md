+++
title = "BPF LRU_HASH_MAP 及 HASH_MAP 的使用异常"
author = "李岩"
date = 2024-03-12T11:02:00+08:00

[taxonomies]
tags = ["bpf", "bugs"]
categories = ["BPF"]
+++
> BPF 技术看起来还有很多不易察觉的缺陷。最近又踩了一个坑。记录下。

`LRU_HASH_MAP` 在实现的时候，出现了不符合预期的数据驱逐问题：设定一个 512 大小的`LRU_HASH_MAP`，很可能出现在40-50个`key`的时候，之前的`key`就被覆盖。在一段时间未更新时，重新更新也可能会出现异常。总结就是，执行了写入操作，很可能没有写入。这个问题在[Elements incorrectly evicted from eBPF LRU hash map](https://stackoverflow.com/questions/75882443/elements-incorrectly-evicted-from-ebpf-lru-hash-map)有较为详细的描述。  
<!--more-->
但是，笔者之所以使用`LRU_HASH_MAP`主要是期望保持整个程序的鲁棒性：期望可以一直写入`key`而不需对`bpf map`的状态进行维护。如果使用固定大小的`HASH_MAP`，当写入的`key`超过`map`的预设大小时，测试的`demo`会出现崩溃的现象。由于`LRU_HASH_MAP`的功能出现了不符合预期的情况，显然也就需要使用`HASH_MAP`来替代了。    
笔者遇到的需要使用`LRU_HASH_MAP`的场景有两种：1. 作为配置项。用户态的代码向`BPF MAP`里写入配置，而后在`BPF`里使用。2. 作为数据中转。比如涉及多个`hook`点配合时，就需要使用`BPF MAP`来存储一些中间数据。对于场景1，可以直接使用`HASH_MAP`来替代，用户态添加一些检查的措施，定期批量对`MAP`进行数据清理以及数据写入即可。但是对于场景2，可能会麻烦很多：数据是随时产出的，用户态没有办法控制其产出的频率、周期。目前能想到的是设置一个较大的`MAP`（这个异常的触发是否和`MAP`的大小有关？），仍然使用`LRU_HASH_MAP`；或者设置一个较大的`HASH_MAP`，然后定时在用户态进行数据的清理。  
以上。