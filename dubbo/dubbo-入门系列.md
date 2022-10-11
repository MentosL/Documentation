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

举例： 三个提供方：a，b，c的权重分别为2，7，1
|   |   |a  |b  |c  |
| --- | --- |--- |--- |--- |
|  | 原始权重 | 2 | 7|1|
|所有权重 + 原始权重，本轮权重总和 = 20| |4|14|2|
|本轮最大权重b - 本轮权重之和| 第1轮|4|14-20=-6|2|
|所有权重 + 原始权重，本轮权重总和 = 10| |6|1|3|
|本轮最大权重a - 本轮权重之和| 第2轮|6-10=-4|1|3|
|所有权重 + 原始权重，本轮权重总和 = 10| |-2|8|4|
|本轮最大权重b - 本轮权重之和| 第3轮|-2|8-10=-2|4|
|所有权重 + 原始权重，本轮权重总和 = 10| |0|5|5|
|本轮最大权重b - 本轮权重之和| 第4轮|0|5-10=-5|5|
|所有权重 + 原始权重，本轮权重总和 = 10| |2|2|6|
|本轮最大权重c - 本轮权重之和| 第5轮|2|2|6-10=-4|
|所有权重 + 原始权重，本轮权重总和 = 10| |4|9|-3|
|本轮最大权重b - 本轮权重之和| 第6轮|4|9-10=-1|-3|
|所有权重 + 原始权重，本轮权重总和 = 10| |6|6|2|
|本轮最大权重a - 本轮权重之和| 第7轮|6-10=-4|6|-2|
|所有权重 + 原始权重，本轮权重总和 = 10| |-2|13|-1|
|本轮最大权重b - 本轮权重之和| 第8轮|-2|13-10=3|-1|
|所有权重 + 原始权重，本轮权重总和 = 10| |0|10|0|
|本轮最大权重b - 本轮权重之和| 第9轮|0|10-10=0|0|
|所有权重 + 原始权重，本轮权重总和 = 10| |2|7|1|
|本轮最大权重b - 本轮权重之和| 第10轮|2|7-10=-3|1|


#### RandomLoadBalance（随机权重算法）
> RandomLoadBalance根据每个服务调用的权值次数来进行随机数，这样权值越大，动态调整越均衡。

主要由以下步骤去完成随机权重算法的实现：
1. 进行invoker权重的累加。
2. 每次动态在总权重范围内取随机值，同时进行循环遍历满足小于权重范围的invoker位置。
3. 如果当前invoker权重值都相等，则随机选择一个进行返回。

代码部分： `org.apache.dubbo.rpc.cluster.loadbalance.RandomLoadBalance`

```java
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
```

  
#### ConsistentHashLoadBalance（一致性哈希算法）
> ConsistentHashLoadBalance一致性哈希算法，相同参数的请求总是发到同一提供者。

代码部分： `org.apache.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance`

```java
   .......
   
   @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String methodName = RpcUtils.getMethodName(invocation);
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
        // using the hashcode of list to compute the hash only pay attention to the elements in the list
        int invokersHashCode = invokers.hashCode();
        // 获取选择器
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
            // 获取配置节点数  默认160
            this.replicaNumber = url.getMethodParameter(methodName, HASH_NODES, 160);
            // 获取需要进行hash的参数数组索引，默认对第一个参数进行hash
            String[] index = COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, HASH_ARGUMENTS, "0"));
            argumentIndex = new int[index.length];
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
            // 创建虚拟结点 映射 invoker
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
        // 选择 Invoker
        public Invoker<T> select(Invocation invocation) {
            //选择 参数作为key
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
            // 如未获取到 默认选择第一个Invoker返回
            if (entry == null) {
                entry = virtualInvokers.firstEntry();
            }
            return entry.getValue();
        }
        
     .......

    }
```

#### Demo
  

## 集群容错

### 概念介绍
> 服务通常都是以集群的形式部署来保证服务的高可用性。在dubbo远程调用过程中，被调用的远程服务并不是每时刻都保持良好的状态，当某个服务调用出现异常时候（比如网络抖动、服务短暂不可用），都需要进行容错。

### Dubbo中的集群容错

| 策略名称|	优点	|缺点	|主要应用场景 |
| --- | --- |--- |--- |
| Failover | 对调用者屏蔽调用失败的信息 |  增加RT，额外资源开销，资源浪费|  对调用rt不敏感的场景| 
| Failfast | 业务快速感知失败状态进行自主决策 |产生较多报错的信息 |  非幂等性操作，需要快速感知失败的场景|
| Failsafe | 即使失败了也不会影响核心流程 |对于失败的信息不敏感，需要额外的监控||旁路系统，失败不影响核心流程正确性的场景|
| Broadcast | 支持对所有的服务提供者进行操作 |资源消耗很大 | 通知所有提供者更新缓存或日志等本地资源信息|

#### Failover  
> 调用失败情况下，通过重试策略来帮助本次的请求的正常进行。

代码部分：`org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker`
```java
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyInvokers = invokers;
        checkInvokers(copyInvokers, invocation);
        String methodName = RpcUtils.getMethodName(invocation);
	// 获取重试次数
        int len = calculateInvokeTimes(methodName);
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            //Reselect before retry to avoid a change of candidate `invokers`.
            //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
            if (i > 0) {
                checkWhetherDestroyed();
                copyInvokers = list(invocation);
                // check again
                checkInvokers(copyInvokers, invocation);
            }
	    // 选择 invoker 排除不可用以及已经选择过的
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);
            RpcContext.getServiceContext().setInvokers((List) invoked);
            boolean success = false;
            try {
	    	// 进行远程调用
                Result result = invokeWithContext(invoker, invocation);
                if (le != null && logger.isWarnEnabled()) {
                    ...warn log...
                }
                success = true;
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                if (!success) {
                    providers.add(invoker.getUrl().getAddress());
                }
            }
        }
        throw new RpcException(....);
    }
```

#### Failfast
> 调用失败则异常返回

代码部分： `org.apache.dubbo.rpc.cluster.support.FailfastClusterInvoker`
```java
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        try {
	    // 远程调用
            return invokeWithContext(invoker, invocation);
        } catch (Throwable e) {
            if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
                throw (RpcException) e;
            }
            throw new RpcException(msg);
        }
    }
```

#### Failsafe  
> 调用失败则空数据返回，不会异常中断

代码部分：`org.apache.dubbo.rpc.cluster.support.FailsafeClusterInvoker`
```java
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invokeWithContext(invoker, invocation);
        } catch (Throwable e) {
            logger.error(CLUSTER_ERROR_RESPONSE,"Failsafe for provider exception","","Failsafe ignore exception: " + e.getMessage(),e);
            return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
        }
    }
```


#### Broadcast  
> 

代码部分：
java


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
