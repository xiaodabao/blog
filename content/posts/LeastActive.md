---
title: "LeastActive"
date: 2021-08-19T00:08:12+08:00
draft: false
---


# 负载均衡 最少活跃数算法

从实际出发，研究在开源软件中已实现的算法。本文具体分析负载均衡-最少活跃数算法，基于dubbo的LeastActiveLoadBalance源码。

1. 准备多个Invoker, 每个Invoker设置具体的名字，权重，调用次数，随机生成活跃调用数
2. 多次调用selectInvoker方法，打印所有Invoker的活跃数，最终选择的Invoker名称
3. 每次选择完成后，更新调用次数+1

这种方式比较适合所有服务配置差不多的情形，大部分的情况下和设置的权重关系不大，在不同的服务器配置环境下，就不能有效的利用配置更好的服务器资源。

**上代码**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

public class LeastActiveLoadBalance {
    public static final String NAME = "leastactive";

    public static Invoker doSelect(List<Invoker> invokers) {
        // Number of invokers
        int length = invokers.size();
        // 最少的正在调用次数
        int leastActive = -1;
        // 多个Invoker可能具有相同的调用次数
        int leastCount = 0;
        // 记录选择的Invoker的索引, 他们具有相同的最少调用次数
        int[] leastIndexes = new int[length];
        // 记录选择的Invoker的权重
        int[] weights = new int[length];
        // totalWeight, 套路和随机算法差不多, 如果有多个最少调用次数的Invoker,
        // 再根据权重概率随机选择一个Invoker
        int totalWeight = 0;
        int firstWeight = 0;
        boolean sameWeight = true;

        // 存储随机生成的Invoker.active, 打印结果看看选择的invoker是否正确
        String[] activeCount = new String[length];
        for (int i = 0; i < length; i++) {
            Invoker invoker = invokers.get(i);
            // 在dubbo中是通过Filter来实时统计当前调用次数的, 测试时使用一个随机生成的数字
            int active = invoker.getRandomNumber();
            activeCount[i] = String.valueOf(active);
            int invokerWeight = invoker.weight;
            weights[i] = invokerWeight;

            /**
             * 1. 如果当前Invoker是第一次迭代 (此时leastActive == -1), 或当前Invoker的active小于最少leastActive
             * 则更新leastActive, 重置leastCount、leastIndexes, totalWeight, firstWeight、sameWeight
             * 2. 如果当前Invoker的active等于leastActive, 则表明存在多个最少leastActive相同的Invoker,
             * 将当前Invoker加入leastIndexes, 更新leastCount, totalWeight, sameWeight
             */
            if (leastActive == -1 || active < leastActive) {
                // 所有变量都重置
                leastActive = active;
                leastCount = 1;
                leastIndexes[0] = i;
                totalWeight = invokerWeight;
                firstWeight = invokerWeight;
                sameWeight = true;
            } else if (active == leastActive) {
                // 将当前Invoker的索引加入leastIndexes, 同时维护了leastCount的值
                leastIndexes[leastCount++] = i;
                totalWeight += invokerWeight;
                // 这里和随机算法中的判断就稍有不同, 维护了一个firstWeight的变量
                if (sameWeight && invokerWeight != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        System.out.println("activeCount: " + String.join(", ", activeCount));
        // 如果只有一个Invoker符合条件, 直接返回这个Invoker
        if (leastCount == 1) {
            System.out.println("selected invoker name: " + invokers.get(leastIndexes[0]).name);
            return invokers.get(leastIndexes[0]);
        }
        /**
         * 存在多个active相同的Invoker, 且权重不同, 这时就采用随机算法来决定选择哪个Invoker了
         */
        if (!sameWeight && totalWeight > 0) {
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                // 这里的算法通过减权重来实现, 一直减到小于0为止
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    System.out.println("selected invoker name: " + invokers.get(leastIndex).name);
                    return invokers.get(leastIndex);
                }
            }
        }
        // 权重相同或未设置权重, 随机选择一个, 注意这里是在leastIndexes[0, leastCount)中进行选择
        Invoker selectedInvoker = invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
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
activeCount: 17, 55, 29, 25, 94, 4, 95, 74, 53, 22
selected invoker name: Service 6
activeCount: 51, 31, 70, 48, 57, 8, 28, 17, 77, 95
selected invoker name: Service 6
activeCount: 12, 51, 10, 59, 48, 65, 81, 23, 52, 4
selected invoker name: Service 10
activeCount: 53, 42, 28, 55, 81, 69, 64, 21, 89, 37
selected invoker name: Service 8
activeCount: 45, 32, 46, 76, 87, 22, 10, 32, 67, 35
selected invoker name: Service 7
activeCount: 8, 94, 9, 76, 79, 41, 35, 93, 55, 97
selected invoker name: Service 1
activeCount: 40, 95, 42, 97, 49, 53, 67, 78, 12, 18
selected invoker name: Service 9
activeCount: 74, 28, 36, 95, 94, 20, 61, 79, 77, 84
selected invoker name: Service 6
activeCount: 49, 40, 26, 88, 38, 33, 99, 28, 8, 3
selected invoker name: Service 10
activeCount: 89, 94, 71, 57, 8, 31, 70, 42, 30, 6
selected invoker name: Service 10
```

**附dubbo LeastActiveLoadBalance源码 version 3.0.2**

```java
package org.apache.dubbo.rpc.cluster.loadbalance;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.RpcStatus;

import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

/**
 * LeastActiveLoadBalance
 * <p>
 * Filter the number of invokers with the least number of active calls and count the weights and quantities of these invokers.
 * If there is only one invoker, use the invoker directly;
 * if there are multiple invokers and the weights are not the same, then random according to the total weight;
 * if there are multiple invokers and the same weight, then randomly called.
 */
public class LeastActiveLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "leastactive";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        int length = invokers.size();
        // The least active value of all invokers
        int leastActive = -1;
        // The number of invokers having the same least active value (leastActive)
        int leastCount = 0;
        // The index of invokers having the same least active value (leastActive)
        int[] leastIndexes = new int[length];
        // the weight of every invokers
        int[] weights = new int[length];
        // The sum of the warmup weights of all the least active invokers
        int totalWeight = 0;
        // The weight of the first least active invoker
        int firstWeight = 0;
        // Every least active invoker has the same weight value?
        boolean sameWeight = true;


        // Filter out all the least active invokers
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            // Get the active number of the invoker
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
            // Get the weight of the invoker's configuration. The default value is 100.
            int afterWarmup = getWeight(invoker, invocation);
            // save for later use
            weights[i] = afterWarmup;
            // If it is the first invoker or the active number of the invoker is less than the current least active number
            if (leastActive == -1 || active < leastActive) {
                // Reset the active number of the current invoker to the least active number
                leastActive = active;
                // Reset the number of least active invokers
                leastCount = 1;
                // Put the first least active invoker first in leastIndexes
                leastIndexes[0] = i;
                // Reset totalWeight
                totalWeight = afterWarmup;
                // Record the weight the first least active invoker
                firstWeight = afterWarmup;
                // Each invoke has the same weight (only one invoker here)
                sameWeight = true;
                // If current invoker's active value equals with leaseActive, then accumulating.
            } else if (active == leastActive) {
                // Record the index of the least active invoker in leastIndexes order
                leastIndexes[leastCount++] = i;
                // Accumulate the total weight of the least active invoker
                totalWeight += afterWarmup;
                // If every invoker has the same weight?
                if (sameWeight && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // Choose an invoker from all the least active invokers
        if (leastCount == 1) {
            // If we got exactly one invoker having the least active value, return this invoker directly.
            return invokers.get(leastIndexes[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on 
            // totalWeight.
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
    }
}
```