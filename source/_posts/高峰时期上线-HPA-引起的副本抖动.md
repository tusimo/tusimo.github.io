title: 高峰时期上线 HPA 引起的副本抖动
author: tusimo
tags:
  - kubernetes
  - hpa
  - pod
categories:
  - kubernetes
date: 2022-08-11 21:11:00
---
# 现象

我们在引入`kubernetes`的`hpa`功能后,高峰时期上线我们的`pod`副本数会缩小到`replicas`指定的数量后,然后`hpa`又会开始扩容,这个时候我们的`pod`无法支撑
请求数,会出现大量的超时.等到`hpa`扩容后会恢复,导致高峰时期不敢轻易上线.

# 原因

没有`hpa`之前,我们的副本数量由`replicas`参数控制,假如设置`replicas:2`,那么我们的`pod`副本数就一直都是 2 个.除非我们手动进行扩容.
引入`hpa`后,副本数会由`hpa`的`minReplicas`和`maxReplicas`控制.发布的时候会先平衡到`replicas`的值,然后`hpa`会根据配置动态的修改`replicas`值来控制`pod`的副本数.
假如`hpa`后我们的副本数为:`4`,大于初始值`replicas:2`的值了.那么发布的时候就会先缩小到`replicas:2`的值.再扩容到发布前的副本数:`4`,这个时候就会出现副本数的抖动.

# 演示图


![upload successful](/images/pasted-12.png)

# 解决办法

使用`hpa`后删除所有`deployments`里面设置的初始的`replicas`节点.不设置`replicas`,那么发布的时候,会根据当前的数量按照设置的更新策略进行.而不是先恢复到`replicas`的值.
那么这个时候初始的副本数则有`minReplicas`控制.

# 题外话

当我们使用滚动更新的时候,要保证副本数最小的可用副本数.不能把所有的副本都`Terminating`了.这样会引起服务异常.
尤其是我们在开发测试环境的时候.由于请求量不大,往往副本数只有一个.当我们滚动更新的时候,可能会导致副本数清零,再启动新副本.

```yaml

strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate

```

:::tip 通俗讲

- `maxSure` 控制加的行为,每次新增多少个副本.可以为整数和百分比
- `maxUnavailable` 控制减的行为, 表示最多不可用的数量

:::


`strategy`（更新策略）：
　　`.spec.strategy` 指定新的Pod替换旧的Pod的策略。 `.spec.strategy.type` 可以是`Recreate`或者是 `RollingUpdate`。`RollingUpdate`是默认值。

　　`Recreate`： 重建式更新，就是删一个建一个。类似于`ReplicaSet`的更新方式，即首先删除现有的Pod对象，然后由控制器基于新模板重新创建新版本资源对象。

　　`rollingUpdate`：滚动更新，简单定义 更新期间`pod`最多有几个等。可以指定`maxUnavailable` 和 `maxSurge` 来控制 `rolling update` 进程。

　　`maxSurge`：.spec.strategy.rollingUpdate.maxSurge 是可选配置项，用来指定可以超过期望的Pod数量的最大个数。该值可以是一个绝对值（例如5）或者是期望的Pod数量的百分比（例如10%）。当        `MaxUnavailable`为0时该值不可以为0。通过百分比计算的绝对值向上取整。默认值是1。

　　例如，该值设置成30%，启动rolling update后新的`ReplicatSet`将会立即扩容，新老Pod的总数不能超过期望的Pod数量的130%。旧的Pod被杀掉后，新的`ReplicaSet`将继续扩容，旧的`ReplicaSet`会进一步缩容，确保在升级的所有时刻所有的Pod数量和不会超过期望Pod数量的130%。

　　`maxUnavailable`：.spec.strategy.rollingUpdate.maxUnavailable 是可选配置项，用来指定在升级过程中不可用Pod的最大数量。该值可以是一个绝对值（例如5），也可以是期望Pod数量的百分比（例如10%）。通过计算百分比的绝对值向下取整。  如果.spec.strategy.rollingUpdate.maxSurge 为0时，这个值不可以为0。默认值是1。

　　例如，该值设置成30%，启动rolling update后旧的`ReplicatSet`将会立即缩容到期望的Pod数量的70%。新的Pod ready后，随着新的`ReplicaSet`的扩容，旧的`ReplicaSet`会进一步缩容确保在升级的所有时刻可以用的Pod数量至少是期望Pod数量的70%。

PS：`maxSurge`和`maxUnavailable`的属性值不可同时为0，否则Pod对象的副本数量在符合用户期望的数量后无法做出合理变动以进行更新操作。

　　在配置时，用户还可以使用`Deployment`控制器的`spec.minReadySeconds`属性来控制应用升级的速度。新旧更替过程中，新创建的`Pod`对象一旦成功响应就绪探测即被认为是可用状态，然后进行下一轮的替换。而`spec.minReadySeconds`能够定义在新的Pod对象创建后至少需要等待多长的时间才能会被认为其就绪，在该段时间内，更新操作会被阻塞。


:::tip 给个稳妥的配置,开发测试环境妥妥的

- `maxSure:1` 一个一个滚动更新,就是发布慢了点.
- `maxUnavailable:0` 至少有一个副本在运行

:::