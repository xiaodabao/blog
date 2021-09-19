---
title: "ShortestResponse"
date: 2021-08-21T00:16:39+08:00
draft: false
---


# 负载均衡 最短响应时间算法

从实际出发，研究在开源软件中已实现的算法。本文具体分析负载均衡-最短响应时间算法，基于dubbo的ShortestResponseLoadBalance源码。本方法和上次介绍的最少活跃数算法的思路基本一样。

1.准备多个Invoker, 每个Invoker设置具体的名字，权重，调用次数，随机生成平均调用时间，当前调用活跃数
2.多次调用selectInvoker方法，打印所有Invoker的活跃数，最终选择的Invoker名称
3.每次选择完成后，更新调用次数+1

**上代码**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

public class ShortestResponseLoadBalance {
    public static final String NAME = "shortestresponse";

    public static Invoker doSelect(List<Invoker> invokers) {
        // Number of invokers
        int length = invokers.size();
        // 估算最短的响应时间  基于之前调用成功的平均耗时 * 当前正在调用的次数 active
        long shortestResponse = Long.MAX_VALUE;
        // 多个Invoker可能具有相同的最短响应时间
        int shortestCount = 0;
        // 记录选择的Invoker的索引, 他们具有相同的最短响应时间
        int[] shortestIndexes = new int[length];
        // 记录选择的Invoker的权重
        int[] weights = new int[length];
        // The sum of the warmup weights of all the shortest response  invokers
        int totalWeight = 0;
        // The weight of the first shortest response invokers
        int firstWeight = 0;
        // Every shortest response invoker has the same weight value?
        boolean sameWeight = true;

        // 存储随机生成的estimateResponse, 打印结果看看选择的invoker是否正确
        String[] estimateResponses = new String[length];
        for (int i = 0; i < length; i++) {
            Invoker invoker = invokers.get(i);
            // Calculate the estimated response time from the product of active connections and succeeded average elapsed time.
            // 随机生成一个Invoker成功响应的平均执行时间
            long succeededAverageElapsed = invoker.getRandomNumber();
            // 随机一个正在调用的请求数
            int active = invoker.getRandomNumber();
            // 预估的响应时间 = 平均执行时间 * 当前活跃的调用数
            long estimateResponse = succeededAverageElapsed * active;
            estimateResponses[i] = String.valueOf(estimateResponse);
            // 记录全部Invoker的权重
            int weight = invoker.weight;
            weights[i] = weight;
            /** 和上篇文章LeastActiveLoadBalance的选择逻辑一样
             * 1. 当前Invoker的estimateResponse小于最短响应时间shortestResponse
             * 则更新shortestResponse, 重置shortestCount、shortestIndexes, totalWeight, firstWeight、sameWeight
             * 2. 如果当前Invoker的estimateResponse等于shortestResponse, 则表明存在多个最短shortestResponse相同的Invoker,
             * 将当前Invoker加入shortestResponse, 更新shortestCount, totalWeight, sameWeight
             */
            if (estimateResponse < shortestResponse) {
                // 所有变量都重置
                shortestResponse = estimateResponse;
                shortestCount = 1;
                shortestIndexes[0] = i;
                totalWeight = weight;
                firstWeight = weight;
                sameWeight = true;
            } else if (estimateResponse == shortestResponse) {
                // 将当前Invoker的索引加入shortestIndexes, 同时维护了shortestCount的值
                shortestIndexes[shortestCount++] = i;
                totalWeight += weight;
                if (sameWeight && i > 0 && weight != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        System.out.println("estimateResponses: " + String.join(", ", estimateResponses));
        // 如果只有一个Invoker符合条件, 直接返回这个Invoker
        if (shortestCount == 1) {
            System.out.println("selected invoker name: " + invokers.get(shortestIndexes[0]).name);
            return invokers.get(shortestIndexes[0]);
        }
        /**
         * 存在多个shortestResponse相同的Invoker, 且权重不同, 这时就采用随机算法来决定选择哪个Invoker了
         */
        if (!sameWeight && totalWeight > 0) {
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            for (int i = 0; i < shortestCount; i++) {
                int shortestIndex = shortestIndexes[i];
                // 这里的算法通过减权重来实现, 一直减到小于0为止
                offsetWeight -= weights[shortestIndex];
                if (offsetWeight < 0) {
                    System.out.println("selected invoker name: " + invokers.get(shortestIndex).name);
                    return invokers.get(shortestIndex);
                }
            }
        }
        // 权重相同或未设置权重, 随机选择一个, 注意这里是在leastIndexes[0, leastCount)中进行选择
        Invoker selectedInvoker = invokers.get(shortestIndexes[ThreadLocalRandom.current().nextInt(shortestCount)]);
        System.out.println("selected invoker name: " + selectedInvoker.name);
        return selectedInvoker;
    }

    public static void main(String[] args) {
        // 初始化10个Invoker, 命名 Service i, Weight i;  i -> [1, 10]
        List<Invoker> invokers = new ArrayList<>(10);
        for (int i = 1; i <= 10; i++) {
            invokers.add(new Invoker("Service " + i, i));
        }

        // 只执行10次, 查看打印的结果
        for (int i = 0; i < 10; i++) {
            Invoker selected = doSelect(invokers);
            // 统计调用次数
            selected.invokeCount.incrementAndGet();
        }
    }
}
```
Invoker
```java
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.atomic.AtomicInteger;

public class Invoker {

    public String name;

    public int weight;

    public AtomicInteger invokeCount;

    public Invoker(String name, int weight) {
        this.name = name;
        this.weight = weight;
        invokeCount = new AtomicInteger(0);
    }

    /**
     * 随机生成100以内的数字
     * @return
     */
    public int getRandomNumber() {
        return ThreadLocalRandom.current().nextInt(100);
    }
}
```
**某一次调用的输出结果**

```
estimateResponses: 162, 434, 224, 3009, 2520, 3680, 5440, 0, 140, 4588
selected invoker name: Service 8
estimateResponses: 152, 1092, 3484, 252, 1428, 6084, 2001, 511, 550, 900
selected invoker name: Service 1
estimateResponses: 0, 0, 0, 3672, 570, 3510, 837, 2080, 66, 5688
selected invoker name: Service 3
estimateResponses: 0, 943, 4002, 4160, 242, 2448, 495, 1200, 180, 4212
selected invoker name: Service 1
estimateResponses: 2040, 1530, 1260, 5643, 6956, 5025, 6930, 675, 204, 5112
selected invoker name: Service 9
estimateResponses: 2842, 4690, 190, 2610, 1888, 459, 286, 1872, 747, 2322
selected invoker name: Service 3
estimateResponses: 372, 2128, 9215, 5929, 3344, 2795, 9212, 1242, 0, 3366
selected invoker name: Service 9
estimateResponses: 288, 192, 4104, 2262, 2596, 1032, 2880, 1106, 6887, 5160
selected invoker name: Service 2
estimateResponses: 504, 280, 2888, 885, 6708, 3328, 5859, 2254, 240, 2100
selected invoker name: Service 9
estimateResponses: 1537, 3738, 1105, 1452, 2560, 4176, 5760, 504, 310, 4160
selected invoker name: Service 9
```

**附dubbo ShortestResponseLoadBalance源码 version 3.0.2**

```java

package org.apache.dubbo.rpc.cluster.loadbalance;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.RpcStatus;

import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

/**
 * ShortestResponseLoadBalance
 * </p>
 * Filter the number of invokers with the shortest response time of success calls 
 * and count the weights and quantities of these invokers.
 * If there is only one invoker, use the invoker directly;
 * if there are multiple invokers and the weights are not the same, then random according to the total weight;
 * if there are multiple invokers and the same weight, then randomly called.
 */
public class ShortestResponseLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "shortestresponse";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        int length = invokers.size();
        // Estimated shortest response time of all invokers
        long shortestResponse = Long.MAX_VALUE;
        // The number of invokers having the same estimated shortest response time
        int shortestCount = 0;
        // The index of invokers having the same estimated shortest response time
        int[] shortestIndexes = new int[length];
        // the weight of every invokers
        int[] weights = new int[length];
        // The sum of the warmup weights of all the shortest response  invokers
        int totalWeight = 0;
        // The weight of the first shortest response invokers
        int firstWeight = 0;
        // Every shortest response invoker has the same weight value?
        boolean sameWeight = true;

        // Filter out all the shortest response invokers
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            RpcStatus rpcStatus = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName());
            // Calculate the estimated response time from the product of active connections and succeeded average elapsed time.
            long succeededAverageElapsed = rpcStatus.getSucceededAverageElapsed();
            int active = rpcStatus.getActive();
            long estimateResponse = succeededAverageElapsed * active;
            int afterWarmup = getWeight(invoker, invocation);
            weights[i] = afterWarmup;
            // Same as LeastActiveLoadBalance
            if (estimateResponse < shortestResponse) {
                shortestResponse = estimateResponse;
                shortestCount = 1;
                shortestIndexes[0] = i;
                totalWeight = afterWarmup;
                firstWeight = afterWarmup;
                sameWeight = true;
            } else if (estimateResponse == shortestResponse) {
                shortestIndexes[shortestCount++] = i;
                totalWeight += afterWarmup;
                if (sameWeight && i > 0
                        && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        if (shortestCount == 1) {
            return invokers.get(shortestIndexes[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            for (int i = 0; i < shortestCount; i++) {
                int shortestIndex = shortestIndexes[i];
                offsetWeight -= weights[shortestIndex];
                if (offsetWeight < 0) {
                    return invokers.get(shortestIndex);
                }
            }
        }
        return invokers.get(shortestIndexes[ThreadLocalRandom.current().nextInt(shortestCount)]);
    }
}
```