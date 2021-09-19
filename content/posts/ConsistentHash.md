---
title: "ConsistentHash"
date: 2021-08-17T23:07:44+08:00
draft: false
---

# 负载均衡 一致性Hash算法

从实际出发，研究在开源软件中已实现的算法。本文具体分析负载均衡-一致性Hash算法，基于dubbo的ConsistentHashLoadBalance源码。

一致性Hash算法需要ConsistentHashSelector的数据结构, 内部使用TreeMap来存储Invoker和其对应的HashCode值，为了保证请求的均匀调用，一般都采用一个Invoker映射为多个虚拟节点的方式，好处很多，自行搜索。当有请求到达时，根据请求的参数来计算本次请求的Hash值，可以自行定义选择N个参数来参与Hash值的计算，相同参数的请求可以定位至同一个Invoke上（前提是Invoker列表未发生变化）

10个Invoker在Hash环上的分布情况：
![十个Invoker在Hash环上的分布情形](https://xiaodabao.github.io/blog/images/ConsistentHash0.jpg)
分布可能十分不均匀，造成某些服务器压力过大。

增加多个虚拟Invoker后的分布情况：
仅画出了Invoker1的分布情况
![多个虚拟Invoker1在Hash环上的分布情形](https://xiaodabao.github.io/blog/images/ConsistentHash1.jpg)

测试代码：

1. 准备多个Invoker, 每个Invoker设置具体的名字，权重，调用次数
2. 启动多个线程，每个线程分别多次调用selectInvoker方法
3. 每次选择完成后，更新调用次数+1
4. 全部调用完成后，输出每个Invoker的调用次数
5. 设置同样的请求参数，打印每次选择的Invoker name

**上代码**

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.CountDownLatch;

public class ConsistentHashLoadBalance {

    public static ThreadLocal<MessageDigest> MD5 = new ThreadLocal<>();

    public static final String NAME = "consistenthash";

    /**
     * Hash nodes name
     */
    public static final String HASH_NODES = "hash.nodes";

    /**
     * Hash arguments name
     */
    public static final String HASH_ARGUMENTS = "hash.arguments";

    /**
     * 参与Hash计算的方法参数的索引值, 只有在其中的索引参数才会参与hash值计算
     * 本次测试假设第0个、第2个参数参与Hash值计算
     */
    public static final int[] HASH_PARAMETER_INDEX = new int[] {0, 2};

    private final ConcurrentMap<String, ConsistentHashSelector> selectors = new ConcurrentHashMap<>();


    protected Invoker doSelect(List<Invoker> invokers, String methodName, Object[] arguments) {
        String key = methodName;
        // using the hashcode of list to compute the hash only pay attention to the elements in the list
        int invokersHashCode = invokers.hashCode();
        ConsistentHashSelector selector = selectors.get(key);
        // 对应的ConsistentHashSelector不存在, 或invokers有变化, 则更新ConsistentHashSelector
        // 本次实验中invokers是固定的, 所以只有第一次调用会设置ConsistentHashSelector
        if (selector == null || selector.identityHashCode != invokersHashCode) {
            selectors.put(key, new ConsistentHashSelector(invokers, methodName, invokersHashCode));
            selector = selectors.get(key);
        }
        return selector.select(arguments);
    }

    /**
     * 获取MessageDigest, 注意此类不是线程安全的, 使用ThreadLocal来获取
     * @return
     */
    public static MessageDigest getMD5() {
        MessageDigest md5 = MD5.get();
        if (md5 == null) {
            try {
                md5 = MessageDigest.getInstance("MD5");
                MD5.set(md5);
            } catch (NoSuchAlgorithmException e) {
                e.printStackTrace();
            }
        }
        return md5;
    }

    private static final class ConsistentHashSelector {

        /**
         * 使用TreeMap来存储digest+Invoker, 主要是利用cellingEntry方法快速查询Invoker
         */
        private final TreeMap<Long, Invoker> virtualInvokers;

        /**
         * 副本数目, 在这表示虚拟Invoker的数量
         */
        private final int replicaNumber = 160;

        private final int identityHashCode;

        private final int[] argumentIndex;

        ConsistentHashSelector(List<Invoker> invokers, String methodName, int identityHashCode) {
            this.virtualInvokers = new TreeMap<>();
            this.identityHashCode = identityHashCode;
            argumentIndex = HASH_PARAMETER_INDEX;
            for (Invoker invoker : invokers) {
                // 使用Invoker唯一的名称来替代地址, 作为区分
                String address = invoker.name;
                // 创建 replicaNumber / 4 * 4 个虚拟节点, 本例中每个Invoker都创建160个虚拟节点
                for (int i = 0; i < replicaNumber / 4; i++) {
                    byte[] digest = getMD5().digest((address + i).getBytes(StandardCharsets.UTF_8));
                    for (int h = 0; h < 4; h++) {
                        long m = hash(digest, h);
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }

        /**
         * 因为我们只测试一个方法, 所以直接传递参数来选择Invoker
         * @param arguments
         * @return
         */
        public Invoker select(Object[] arguments) {
            String key = toKey(arguments);
            byte[] digest = getMD5().digest(key.getBytes(StandardCharsets.UTF_8));
            return selectForKey(hash(digest, 0));
        }

        /**
         * 得到HashKey
         * @param args
         * @return
         */
        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            // 在这里我们只选择argumentIndex中的参数作为key参与Hash值的计算
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    // 一般来说, 对应的参数都应该重写toString方法
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }

        /**
         * 根据Hash值选择Invoker
         * @param hash
         * @return
         */
        private Invoker selectForKey(long hash) {
            // 在虚拟节点对应的TreeMap中选择刚大于hash值的Invoker
            Map.Entry<Long, Invoker> entry = virtualInvokers.ceilingEntry(hash);
            if (entry == null) {
                // 如果不存在, hash值大于最大的虚拟节点的hash值, 则应该选择第一个Invoker
                entry = virtualInvokers.firstEntry();
            }
            return entry.getValue();
        }

        /**
         * 计算Hash值
         * @param digest
         * @param number
         * @return
         */
        private long hash(byte[] digest, int number) {
            /**
             * 在初始化虚拟Invoker的时候, number分别为 0,1,2,3
             * 也就是说分别以 0~3, 4~7, 8~11, 12~15 的byte来计算hash值
             * 保证4次计算结果比较分散
             */
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }
    }

    public static void main(String[] args) {
        ConsistentHashLoadBalance consistentHashLoadBalance = new ConsistentHashLoadBalance();
        // 初始化10个Invoker, 命名 Service i, Weight i;  i -> [1, 10]
        List<Invoker> invokers = new ArrayList<>(10);
        for (int i = 1; i <= 10; i++) {
            invokers.add(new Invoker("Service " + i, i));
        }
        CountDownLatch cdl = new CountDownLatch(10);

        // 启动10个线程, 每个线程调用10万次, 共调用100万次
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                Object[] arguments = new Object[4];
                for (int j = 0; j < 100000; j++) {
                    // UUID随机生成4个参数
                    arguments[0] = UUID.randomUUID().toString();
                    arguments[1] = UUID.randomUUID().toString();
                    arguments[2] = UUID.randomUUID().toString();
                    arguments[3] = UUID.randomUUID().toString();
                    Invoker selected = consistentHashLoadBalance.doSelect(invokers, "defaultMethod", arguments);
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

        // 测试Hash一致性, 固定第0个、第2个参数, 第1个、第3个参数随机生成, 看是否一直选择同一个Invoker
        String firstArg = UUID.randomUUID().toString();
        String thirdArg = UUID.randomUUID().toString();

        Object[] arguments = new Object[4];
        arguments[0] = firstArg;
        arguments[2] = thirdArg;
        for (int i = 0; i < 20; i++) {
            arguments[1] = UUID.randomUUID().toString();
            arguments[3] = UUID.randomUUID().toString();
            Invoker selected = consistentHashLoadBalance.doSelect(invokers, "defaultMethod", arguments);
            System.out.println("当前选择的Invoker = " + selected.name);
        }
    }
}
```

**输出结果**

```
Service 1: invokeCount = 107767
Service 2: invokeCount = 90664
Service 3: invokeCount = 111901
Service 4: invokeCount = 97779
Service 5: invokeCount = 96242
Service 6: invokeCount = 97307
Service 7: invokeCount = 109512
Service 8: invokeCount = 102675
Service 9: invokeCount = 82314
Service 10: invokeCount = 103839
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
当前选择的Invoker = Service 8
```

**分析**

由结果可知，请求大致分配均匀，当增大虚拟节点个数，会更加均匀。
参数0、2不变时，每次请求返回的Invoker都一样。

**附dubbo ConsistentHashLoadBalance源码 version 3.0.2**

```java
package org.apache.dubbo.rpc.cluster.loadbalance;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.io.Bytes;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.support.RpcUtils;

import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

import static org.apache.dubbo.common.constants.CommonConstants.COMMA_SPLIT_PATTERN;

/**
 * ConsistentHashLoadBalance
 */
public class ConsistentHashLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "consistenthash";

    /**
     * Hash nodes name
     */
    public static final String HASH_NODES = "hash.nodes";

    /**
     * Hash arguments name
     */
    public static final String HASH_ARGUMENTS = "hash.arguments";

    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = 
            new ConcurrentHashMap<String, ConsistentHashSelector<?>>();

    @SuppressWarnings("unchecked")
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String methodName = RpcUtils.getMethodName(invocation);
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
        // using the hashcode of list to compute the hash only pay attention to the elements in the list
        int invokersHashCode = invokers.hashCode();
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        if (selector == null || selector.identityHashCode != invokersHashCode) {
            selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, invokersHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation);
    }

    private static final class ConsistentHashSelector<T> {

        private final TreeMap<Long, Invoker<T>> virtualInvokers;

        private final int replicaNumber;

        private final int identityHashCode;

        private final int[] argumentIndex;

        ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
            this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
            this.identityHashCode = identityHashCode;
            URL url = invokers.get(0).getUrl();
            this.replicaNumber = url.getMethodParameter(methodName, HASH_NODES, 160);
            String[] index = COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, HASH_ARGUMENTS, "0"));
            argumentIndex = new int[index.length];
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
            for (Invoker<T> invoker : invokers) {
                String address = invoker.getUrl().getAddress();
                for (int i = 0; i < replicaNumber / 4; i++) {
                    byte[] digest = Bytes.getMD5(address + i);
                    for (int h = 0; h < 4; h++) {
                        long m = hash(digest, h);
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }

        public Invoker<T> select(Invocation invocation) {
            String key = toKey(invocation.getArguments());
            byte[] digest = Bytes.getMD5(key);
            return selectForKey(hash(digest, 0));
        }

        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }

        private Invoker<T> selectForKey(long hash) {
            Map.Entry<Long, Invoker<T>> entry = virtualInvokers.ceilingEntry(hash);
            if (entry == null) {
                entry = virtualInvokers.firstEntry();
            }
            return entry.getValue();
        }

        private long hash(byte[] digest, int number) {
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }
    }
}
```