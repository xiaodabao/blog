---
title: "Load Balance RandomLoadBalance"
date: 2021-08-14T10:25:40+08:00
draft: false
---

# 负载均衡 随机算法

从实际出发，研究在开源软件中已实现的算法。本文具体分析负载均衡-随机算法，基于dubbo的RandomLoadBalance源码。

1. 准备多个Invoker, 每个Invoker设置具体的名字，权重，调用次数
2. 启动多个线程，每个线程分别多次调用selectInvoker方法
3. 每次选择完成后，更新调用次数+1
4. 全部调用完成后，输出每个Invoker的调用次数

**上代码**

```java
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
}

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ThreadLocalRandom;

/**
 * 负载均衡 权重随机算法
 */
public class RandomLoadBalance {

    public static Invoker doSelect(List<Invoker> invokers) {
        // Number of invokers
        int length = invokers.size();

        // 判断每个Invoker是否具有相同的权重值
        boolean sameWeight = true;
        // the maxWeight of every invokers, the minWeight = 0 or the maxWeight of the last invoker
        int[] weights = new int[length];
        // The sum of weights
        int totalWeight = 0;
        for (int i = 0; i < length; i++) {
            int weight = invokers.get(i).weight;
            // Sum
            totalWeight += weight;
            // weights[i]的值设置为前i个权重值之和, 这个值会在后续生成随机值后选择具体的Invoker时使用
            weights[i] = totalWeight;
            // totalWeight == weight * (i * 1) 是判断当前Invoker的权重是否和前n-1个Invoker相同
            // 记得之前版本的实现是通过和第一个Invoker的权重是否相同进行判断的, 需要额外维护一个firstWeight的变量
            if (sameWeight && totalWeight != weight * (i + 1)) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // 所有Invoker的权重值不完全一致, 则生成一个[0, totalWeight)之间的值
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // 这里之前设置的weights值就派上用场了, 从小到大遍历, 直到找到第一个大于offset的weights的索引i
            for (int i = 0; i < length; i++) {
                if (offset < weights[i]) {
                    return invokers.get(i);
                }
            }
        }
        // 如果权重都一样, 直接随机[0, length)即可
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }


    public static void main(String[] args) {
        // 初始化10个Invoker, 命名 Service i, Weight i;  i -> [1, 10]
        List<Invoker> invokers = new ArrayList<>(10);
        for (int i = 1; i <= 10; i++) {
            invokers.add(new Invoker("Service " + i, i));
        }
        CountDownLatch cdl = new CountDownLatch(10);

        // 启动10个线程, 每个线程调用10万次, 共调用100万次
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 100000; j++) {
                    Invoker selected = doSelect(invokers);
                    // 统计调用次数
                    selected.invokeCount.incrementAndGet();
                }
                cdl.countDown();
            }).start();
        }
        try {
            cdl.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 打印调用次数
        for (Invoker invoker : invokers) {
            System.out.println(invoker.name + ": invokeCount = " + invoker.invokeCount.get());
        }
    }
}
```

**输出结果**

```
Service 1: invokeCount = 18288

Service 2: invokeCount = 36459

Service 3: invokeCount = 54380

Service 4: invokeCount = 72509

Service 5: invokeCount = 90887

Service 6: invokeCount = 109046

Service 7: invokeCount = 126641

Service 8: invokeCount = 145982

Service 9: invokeCount = 163680

Service 10: invokeCount = 182128
```

**分析**

代码中weights[]的值
![invoker的权重值](https://xiaodabao.github.io/blog/images/randomloadbalance.jpg)

假设随机值 = 20
当 20 < weight[5] (21) 时, 则返回第5个Invoker

**附dubbo RandomBalance源码 version 3.0.2**

```java
package org.apache.dubbo.rpc.cluster.loadbalance;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.utils.StringUtils;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;

import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

import static org.apache.dubbo.common.constants.CommonConstants.TIMESTAMP_KEY;
import static org.apache.dubbo.common.constants.RegistryConstants.REGISTRY_KEY;
import static org.apache.dubbo.common.constants.RegistryConstants.REGISTRY_SERVICE_REFERENCE_PATH;
import static org.apache.dubbo.rpc.cluster.Constants.WEIGHT_KEY;

/**
 * This class select one provider from multiple providers randomly.
 * You can define weights for each provider:
 * If the weights are all the same then it will use random.nextInt(number of invokers).
 * If the weights are different then it will use random.nextInt(w1 + w2 + ... + wn)
 * Note that if the performance of the machine is better than others, you can set a larger weight.
 * If the performance is not so good, you can set a smaller weight.
 */
public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";

    /**
     * Select one invoker between a list using a random criteria
     *
     * @param invokers   List of possible invokers
     * @param url        URL
     * @param invocation Invocation
     * @param <T>
     * @return The selected invoker
     */
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        int length = invokers.size();

        if (!needWeightLoadBalance(invokers, invocation)) {
            return invokers.get(ThreadLocalRandom.current().nextInt(length));
        }

        // Every invoker has the same weight?
        boolean sameWeight = true;
        // the maxWeight of every invokers, the minWeight = 0 or the maxWeight of the last invoker
        int[] weights = new int[length];
        // The sum of weights
        int totalWeight = 0;
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // Sum
            totalWeight += weight;
            // save for later use
            weights[i] = totalWeight;
            if (sameWeight && totalWeight != weight * (i + 1)) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < length; i++) {
                if (offset < weights[i]) {
                    return invokers.get(i);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }

    private <T> boolean needWeightLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {

        Invoker invoker = invokers.get(0);
        URL invokerUrl = invoker.getUrl();
        // Multiple registry scenario, load balance among multiple registries.
        if (REGISTRY_SERVICE_REFERENCE_PATH.equals(invokerUrl.getServiceInterface())) {
            String weight = invokerUrl.getParameter(REGISTRY_KEY + "." + WEIGHT_KEY);
            if (StringUtils.isNotEmpty(weight)) {
                return true;
            }
        } else {
            String weight = invokerUrl.getMethodParameter(invocation.getMethodName(), WEIGHT_KEY);
            if (StringUtils.isNotEmpty(weight)) {
                return true;
            } else {
                String timeStamp = invoker.getUrl().getParameter(TIMESTAMP_KEY);
                if (StringUtils.isNotEmpty(timeStamp)) {
                    return true;
                }
            }
        }
        return false;
    }
}
```