# Dubbo 不仅仅是 RPC 框架之服务治理篇

## 服务治理简介

> 服务治理通过改变运行时服务的行为和选址逻辑、达到限流、权重配置等目的进而保障保障服务在运行时期的稳定性。

## 负载均衡

### 概念介绍

> 将对应的工作任务进行平衡、分摊到多个操作单元上进行执行,从调用方式而言也可分为客户端负载以及服务端负载俩种方式。下面介绍dubbo中使用的几种负载均衡算法。

### Dubbo中的负载均衡

| 名称  | 说明  |
| --- | --- |
| RoundRobinLoadBalance | 加权轮询算法，根据权重设置轮询比例 |
| RandomLoadBalance | 随机算法，根据权重设置随机的概率 |
| ConsistentHashLoadBalance | Hash 一致性算法，相同请求参数分配到相同提供者 |

#### RoundRobinLoadBalance （加权轮询算法）
> `Dubbo`中使用了加权轮询算法进行负载均衡的实现，通过该算法可以巧妙的实现动态加权轮询。

主要由以下步骤去完成加权轮询算法的实现：
1. 初始化本地权重表，根据情况动态调整
2. 每次动态的更新本地权重表，更新算法为当前invoker的权重+本地权重表的old值
3. 选取本地权重最大的invoker，并将其本地权重表的权重-本轮所有invoker的权重和，并返回当前的invoker

代码部分： `org.apache.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance`

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // key = service + method 
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.computeIfAbsent(key, k -> new ConcurrentHashMap<>());
        int totalWeight = 0;
        long maxCurrent = Long.MIN_VALUE;
        long now = System.currentTimeMillis();
        Invoker<T> selectedInvoker = null;
        WeightedRoundRobin selectedWRR = null;
        for (Invoker<T> invoker : invokers) {
            String identifyString = invoker.getUrl().toIdentityString();
            // 获取当前invoker的权重
            int weight = getWeight(invoker, invocation);
            WeightedRoundRobin weightedRoundRobin = map.computeIfAbsent(identifyString, k -> {
                WeightedRoundRobin wrr = new WeightedRoundRobin();
                wrr.setWeight(weight);
                return wrr;
            });
            // 权重发生变化 更新权重
            if (weight != weightedRoundRobin.getWeight()) {
                //weight changed
                weightedRoundRobin.setWeight(weight);
            }
            // current+=weight
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
            // 移除长时间未更新的节点
            map.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
        }
        if (selectedInvoker != null) {
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }
        // should not happen here
        return invokers.get(0);
    }
```

举例： 三个提供方：a，b，c的权重分别为3，6，1
|   |   |a  |b  |c  |
| --- | --- |--- |--- |--- |
|  | 原始权重 | 3 | 6|1|
|所有权重 + 原始权重，本轮权重总和 = 20||6|12|2|



#### RandomLoadBalance（随机算法）
  
#### ConsistentHashLoadBalance（一致性哈希算法）
  
#### Demo
  

## 集群容错

### 概念介绍

### Dubbo中的集群容错

| 名称  | 说明  |
| --- | --- |
| Failover | 加权轮询算法，根据权重设置轮询比例 |
| Failfast | 随机算法，根据权重设置随机的概率 |
| Failsafe | Hash 一致性算法，相同请求参数分配到相同提供者 |
| Broadcast | Hash 一致性算法，相同请求参数分配到相同提供者 |

#### Failover  
#### Failfast  
#### Failsafe  
#### Broadcast  
#### Demo

## 路由选址

### 概念介绍

| 名称  | 说明  |
| --- | --- |
| Tag | 加权轮询算法，根据权重设置轮询比例 |
| Condition | 随机算法，根据权重设置随机的概率 |
| Mesh | Hash 一致性算法，相同请求参数分配到相同提供者 |

#### Tag  
#### Condition  
#### Mesh
