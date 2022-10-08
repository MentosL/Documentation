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

#### RoundRobinLoadBalance （轮询算法）
  
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
