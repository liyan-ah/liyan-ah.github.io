+++
title = "ratelimit服务流量限制"
author = "李岩"
date = 2021-02-07T17:30:00+08:00

[taxonomies]
tags = ["系统稳定性", "压力测试", "自我修养", "读代码"]
categories = ["code"]
+++
> 在日常的工作中，固定QPS或者固定并发数是常用的两个衡量系统容量时采用的流量控制手段。本文以[Go语言高级编程](https://chai2010.cn/advanced-go-programming-book/) 服务流量限制的内容为开端，对服务流量限制进行展开描述，同时对`Jmeter`及`golang ratelimit`中的流量限制方法进行描述。

1. 起因
2. 漏桶法
3. 令牌桶法
4. Jmeter中流量吞吐控制
5. golang ratelimit
<!-- more-->

# 起因
流量限制手段在系统流量控制以及系统质量评估上都有广泛的应用。对于有多个子模块/下游的系统，如果已知其中一个模块/下游是整个系统处理能力的瓶颈，从系统的入口添加流量限制并添加超量告警，不失为是保护系统的有效手段。从质量保证的手段来说，在衡量一个系统的稳定性时，需要有一个有效的手段来控制给予系统的压力并进行控制。  
固定并发数量的流量控制方式是相对容易实现的：对于系统而言，可以添加一个连接池；对于请求方而言，维护一个请求并发池即可。对于固定QPS的流量控制手段而言，则又复杂一些：由于基本指令的直接支持，所以固定QPS的流量控制手段多在基于并发的流量控制上进行二次的封装。封装的措施实际上又会影响控制的效果。笔者曾经在搜索系统上，尝试基于Jmeter，使用1000个线程来产生一个固定的100QPS的并发数。由于Jmeter固定吞吐量实现的特点，导致实际产生的效果中，100个请求多集中在1分钟的前几秒，甚至是最开始1s的前若干ms。使得服务承受的顺势并发非常大，服务出现异常也是可以预见的事情了。  
了解一些流量控制的手段还是有必要的。本文主要梳理一下[Go语言高级编程](https://chai2010.cn/advanced-go-programming-book/)提到的漏桶及令牌桶两种方法，并且进行简单的实现。

# 漏桶法
基于[Leaky_bucket](https://en.wikipedia.org/wiki/Leaky_bucket)的描述，目前广泛流行的漏桶法存在两种模式：度量法（the leaky bucket as a meter）及队列法（the leaky bucket as a queue）。  
度量法在处理时，单位时间内的请求如果超过了预设的数量，会将请求丢弃。比如，需要固定的流量为100QPS，我们以100ms作为一个衡量单元，即10 query/100ms。则，在单位的100ms内，如果请求数量超过了10，则将超过10的请求丢弃。对于队列法，则会将超过的请求均放在一个队列里，在下个时间单位内，按照先进先出的原则，处理队列内的请求。  
在请求数量较多且分布均匀的场景下，度量法更加适用。系统已经处于处理的极限，额外的请求存储似乎不太现实。对于流量分布不均的场景下，队列法能够抹平流量的不均匀。在队列长度可控的场景下，队列法能够兼顾请求方（尽量不丢请求）及服务方（控制流量）。至于超出的部分，应该考虑引入告警等方式来把控风险。

# 令牌桶法
对令牌桶法的详细介绍见[Token bucket](https://en.wikipedia.org/wiki/Token_bucket)。令牌桶法可以认为是更加一般的漏桶法。严格意义上的漏桶法要求每次仅有一个单位的请求被允许，令牌桶法则将其扩展为固定时间段内，产出多个令牌，被请求申请。当令牌桶法每次仅允许一个令牌时，显然就成了漏桶法。

# Jmeter中吞吐量的控制逻辑
笔者找到的Jmeter最新版本为[ConstantThroughputTimer](https://github.com/apache/jmeter/blob/master/src/components/src/main/java/org/apache/jmeter/timers/ConstantThroughputTimer.java)。在该实现中，主要分为单线程、多线程、共享线程等模式下的吞吐量（Jmeter中的吞吐量为Query Per Minutes)等模式。可以看出，Jmeter在不同的限流逻辑下，计算每个线程需要的delay时间实现jmeter的请求调度，体现了漏桶法的思路。
相关代码如下：
```
    // Calculate the delay based on the mode
    private long calculateDelay() {
        long delay;
        // N.B. we fetch the throughput each time, as it may vary during a test
        double msPerRequest = MILLISEC_PER_MIN / getThroughput();
        switch (mode) {
        case AllActiveThreads: // Total number of threads
            delay = Math.round(JMeterContextService.getNumberOfThreads() * msPerRequest);
            break;

        case AllActiveThreadsInCurrentThreadGroup: // Active threads in this group
            delay = Math.round(JMeterContextService.getContext().getThreadGroup().getNumberOfThreads() * msPerRequest);
            break;

        case AllActiveThreads_Shared: // All threads - alternate calculation
            delay = calculateSharedDelay(allThreadsInfo,Math.round(msPerRequest));
            break;

        case AllActiveThreadsInCurrentThreadGroup_Shared: //All threads in this group - alternate calculation
            final org.apache.jmeter.threads.AbstractThreadGroup group =
                JMeterContextService.getContext().getThreadGroup();
            ThroughputInfo groupInfo = threadGroupsInfoMap.get(group);
            if (groupInfo == null) {
                groupInfo = new ThroughputInfo();
                ThroughputInfo previous = threadGroupsInfoMap.putIfAbsent(group, groupInfo);
                if (previous != null) { // We did not replace the entry
                    groupInfo = previous; // so use the existing one
                }
            }
            delay = calculateSharedDelay(groupInfo,Math.round(msPerRequest));
            break;

        case ThisThreadOnly:
        default: // e.g. 0
            delay = Math.round(msPerRequest); // i.e. * 1
            break;
        return delay;
    }
```

# golang ratelimit介绍
golang中也有很多请求控制的方法。工程中经常使用的 `chan(bool)`+`WaitGroup`池化了请求限制，可以认为是令牌桶法的思路的一种简化；golang自带的`Ticker`则会在固定的时间间隔内产生一个就绪的状态，可以看出漏桶法的思想。更加工程化的选择，可以看下[golang ratelimit](https://github.com/uber-go/ratelimit)uber开源的这个golang版本的ratelimit实现。水平优先，就贴一个网上找来的源码分析文章[uber-go 漏桶限流器使用与原理分析](https://www.cyhone.com/articles/analysis-of-uber-go-ratelimit/)。
# 总结
本文对常用的两个限流方法`漏桶法`及`令牌桶法`进行了简单的描述。同时简单涉及了下`Jmeter`中的流量限制及`golang`中不同请求限制措施的思路。  