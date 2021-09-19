---
title: "Load Balance RoundRobinLoadBalance"
date: 2021-08-15T22:10:59+08:00
draft: false
---


# 负载均衡 轮询算法

从实际出发，研究在开源软件中已实现的算法。本文具体分析负载均衡-轮询算法，基于dubbo的RoundRobinLoadBalance源码。

1. 准备多个Invoker, 每个Invoker设置具体的名字，权重，调用次数
2. 启动多个线程，每个线程分别多次调用selectInvoker方法
3. 每次选择完成后，更新调用次数+1
4. 全部调用完成后，输出每个Invoker的调用次数

**上代码**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicLong;

public class RoundRobinLoadBalance {

    /**
     * 轮询+权重
     */
    protected static class WeightedRoundRobin {
        /**
         * weight是权重值, 在setWeight方法中进行设置, 同时将current(可认为是优先级)设置为0
         */
        private int weight;
        private AtomicLong current = new AtomicLong(0);

        public void setWeight(int weight) {
            this.weight = weight;
            current.set(0);
        }

        /**
         * Invoker每经过一次选择, 都将其优先级提高weight, 优先级提高, 下次被选中的几率变大
         * @return
         */
        public long increaseCurrent() {
            return current.addAndGet(weight);
        }

        /**
         * 一旦当前Invoker被选中, 则其优先级降为最低
         * 方法就是current(当前优先级) - 全部Invoker的权重和
         * @param total
         */
        public void sel(int total) {
            current.addAndGet(-1 * total);
        }
    }

    /**
     * Invoker - WeightedRoundRobin的对应关系
     * 和dubbo的实现稍有不同, dubbo需要维护 Service - Method - WeightedRoundRobin 两级缓存
     * 本次是实验性质, 只需维护Invoker - WeightedRoundRobin一级缓存即可
     */
    private static ConcurrentMap<String, WeightedRoundRobin> methodWeightMap = new ConcurrentHashMap<>();

    public static Invoker doSelect(List<Invoker> invokers) {
        ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap;
        // totalWeight 计算所有Invoker的权重和
        int totalWeight = 0;
        long maxCurrent = Long.MIN_VALUE;
        // 记录最终选中的Invoker和WeightedRoundRobin
        Invoker selectedInvoker = null;
        WeightedRoundRobin selectedWRR = null;
        for (Invoker invoker : invokers) {
            // 因为我们测试时设置的Invoker.name都是唯一的, 所以用Invoker.name作为唯一标识符
            // 和dubbo调用中使用Url作为唯一标识符的作用是一样的, 用来区分每个Invoker
            String identifyString = invoker.name;
            int weight = invoker.weight;
            // 不存在则初始化, 在实际场景中, Invokers是通过服务的注册、关闭不断变化的
            WeightedRoundRobin weightedRoundRobin = map.computeIfAbsent(identifyString, k -> {
                WeightedRoundRobin wrr = new WeightedRoundRobin();
                wrr.setWeight(weight);
                return wrr;
            });
            // 每经过一次调用, 优先级都增加weight
            long cur = weightedRoundRobin.increaseCurrent();
            // 选择具有最高优先级的Invoker
            if (cur > maxCurrent) {
                maxCurrent = cur;
                selectedInvoker = invoker;
                selectedWRR = weightedRoundRobin;
            }
            totalWeight += weight;
        }
        // 选中Invoker之后, 将其对应的优先级降为最低, 通过减掉所有Invoker权重和的方式
        selectedWRR.sel(totalWeight);
        return selectedInvoker;
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


        // 额外测试, 单线程, 测试Invoker选择的顺序
        methodWeightMap.clear();

        for (int i = 0; i < 55; i++) {
            Invoker selected = doSelect(invokers);
            System.out.printf("第 %d 次选择: %s\n",  i + 1, selected.name);
        }
    }
}
```

**输出结果**

```
Service 1: invokeCount = 18182
Service 2: invokeCount = 36364
Service 3: invokeCount = 54545
Service 4: invokeCount = 72727
Service 5: invokeCount = 90909
Service 6: invokeCount = 109091
Service 7: invokeCount = 127273
Service 8: invokeCount = 145455
Service 9: invokeCount = 163636
Service 10: invokeCount = 181818
第 1 次选择: Service 10
第 2 次选择: Service 9
第 3 次选择: Service 8
第 4 次选择: Service 7
第 5 次选择: Service 6
第 6 次选择: Service 5
第 7 次选择: Service 4
第 8 次选择: Service 10
第 9 次选择: Service 3
第 10 次选择: Service 9
第 11 次选择: Service 8
第 12 次选择: Service 7
第 13 次选择: Service 2
第 14 次选择: Service 10
第 15 次选择: Service 6
第 16 次选择: Service 9
第 17 次选择: Service 5
第 18 次选择: Service 8
第 19 次选择: Service 10
第 20 次选择: Service 7
第 21 次选择: Service 4
第 22 次选择: Service 9
第 23 次选择: Service 6
第 24 次选择: Service 8
第 25 次选择: Service 10
第 26 次选择: Service 1
第 27 次选择: Service 3
第 28 次选择: Service 9
第 29 次选择: Service 7
第 30 次选择: Service 5
第 31 次选择: Service 10
第 32 次选择: Service 8
第 33 次选择: Service 6
第 34 次选择: Service 9
第 35 次选择: Service 4
第 36 次选择: Service 7
第 37 次选择: Service 10
第 38 次选择: Service 8
第 39 次选择: Service 5
第 40 次选择: Service 9
第 41 次选择: Service 2
第 42 次选择: Service 10
第 43 次选择: Service 6
第 44 次选择: Service 7
第 45 次选择: Service 8
第 46 次选择: Service 9
第 47 次选择: Service 3
第 48 次选择: Service 10
第 49 次选择: Service 4
第 50 次选择: Service 5
第 51 次选择: Service 6
第 52 次选择: Service 7
第 53 次选择: Service 8
第 54 次选择: Service 9
第 55 次选择: Service 10
```

**分析**

和随机算法不同的是 多次执行轮询doSelect方法，得到的结果都是一致的。在Invoker列表和权重值都不变的情况下，每次选择的结果是确定的。
下面图示说明前几次的选择逻辑
methodWeightedMap默认是无值、无序的，为了方便说明我们默认所有Invoker的初始值且其是有序的。

**第一次选择前：**
![第一次选择前的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance0.jpg)

**第一次选中WeightedRoundRobin[9], 选择后的结果为：**
![第一次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance1.jpg)

**第二次选中WeightedRoundRobin[8]，选择后的结果为：**
![第二次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance2.jpg)

**第三次选中WeightedRoundRobin[7]，选择后的结果为：**
![第三次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance3.jpg)

**第四次选中WeightedRoundRobin[6]，选择后的结果为：**
![第四次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance4.jpg)

**第五次选中WeightedRoundRobin[5]，选择后的结果为：**
![第五次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance5.jpg)

**第六次选中WeightedRoundRobin[4]，选择后的结果为：**
![第六次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance6.jpg)

**第七次选中WeightedRoundRobin[3]，选择后的结果为：**
![第七次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance7.jpg)

**第八次选中WeightedRoundRobin[9]，选择后的结果为：**
![第七次选择后的权重情况](https://xiaodabao.github.io/blog/images/RoundRobinLoadbalance8.jpg)

这里解释一下，选择之前，WeightedRoundRobin[3] = 24， WeightedRoundRobin[9] = 25， 所以本次选择了WeightedRoundRobin[9]
就分析到这里。

**附dubbo RoundRobinLoadBalance源码 version 3.0.2**

```java
import org.apache.dubbo.common.URL;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Round robin load balance.
 */
public class RoundRobinLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "roundrobin";

    private static final int RECYCLE_PERIOD = 60000;

    protected static class WeightedRoundRobin {
        private int weight;
        private AtomicLong current = new AtomicLong(0);
        private long lastUpdate;

        public int getWeight() {
            return weight;
        }

        public void setWeight(int weight) {
            this.weight = weight;
            current.set(0);
        }

        public long increaseCurrent() {
            return current.addAndGet(weight);
        }

        public void sel(int total) {
            current.addAndGet(-1 * total);
        }

        public long getLastUpdate() {
            return lastUpdate;
        }

        public void setLastUpdate(long lastUpdate) {
            this.lastUpdate = lastUpdate;
        }
    }

    private ConcurrentMap<String, ConcurrentMap<String, WeightedRoundRobin>> methodWeightMap = 
            new ConcurrentHashMap<String, ConcurrentMap<String, WeightedRoundRobin>>();

    /**
     * get invoker addr list cached for specified invocation
     * <p>
     * <b>for unit test only</b>
     *
     * @param invokers
     * @param invocation
     * @return
     */
    protected <T> Collection<String> getInvokerAddrList(List<Invoker<T>> invokers, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        Map<String, WeightedRoundRobin> map = methodWeightMap.get(key);
        if (map != null) {
            return map.keySet();
        }
        return null;
    }

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.computeIfAbsent(key, k -> new ConcurrentHashMap<>());
        int totalWeight = 0;
        long maxCurrent = Long.MIN_VALUE;
        long now = System.currentTimeMillis();
        Invoker<T> selectedInvoker = null;
        WeightedRoundRobin selectedWRR = null;
        for (Invoker<T> invoker : invokers) {
            String identifyString = invoker.getUrl().toIdentityString();
            int weight = getWeight(invoker, invocation);
            WeightedRoundRobin weightedRoundRobin = map.computeIfAbsent(identifyString, k -> {
                WeightedRoundRobin wrr = new WeightedRoundRobin();
                wrr.setWeight(weight);
                return wrr;
            });

            if (weight != weightedRoundRobin.getWeight()) {
                //weight changed
                weightedRoundRobin.setWeight(weight);
            }
            long cur = weightedRoundRobin.increaseCurrent();
            weightedRoundRobin.setLastUpdate(now);
            if (cur > maxCurrent) {
                maxCurrent = cur;
                selectedInvoker = invoker;
                selectedWRR = weightedRoundRobin;
            }
            totalWeight += weight;
        }
        if (invokers.size() != map.size()) {
            map.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
        }
        if (selectedInvoker != null) {
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }
        // should not happen here
        return invokers.get(0);
    }
}
```