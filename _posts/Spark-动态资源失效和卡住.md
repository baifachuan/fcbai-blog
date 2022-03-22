---
title: Spark 动态资源失效和卡住
tags: 编程基础
categories: 编程基础
keywords: Spark动态资源
description: Spark 动态资源失效和卡住，Spark 动态资源卡住
abbrlink: f879737d
date: 2022-02-14 12:00:31
---

### 问题描述
最近开启动态资源后，任务运行很慢，去Spark HistoryServer页面看了下，发现只剩下一个executor在运行，而运行中的Stage还有几千个task等待执行。executor数量并没有达到上限，但是spark却没有去申请更多的executor来运行task。

下面是该任务和动态资源有关的相关配置：
```
spark.dynamicAllocation.enabled true
spark.executor.instances 60
spark.dynamicAllocation.executorIdleTimeout 30s
spark.dynamicAllocation.minExecutors 1
spark.dynamicAllocation.maxExecutors 60
```

从配置可以看出，这个任务是开启了动态资源了，因此最多可以申请60个executor运行。观察日志发现一开始运行时也有60个executor在运行，但是随着任务的运行，executor最终只剩下一个（其他的executor都是因为空闲时间超过30s被kill）。
总结了下，一共有两个可疑的地方：
* 看日志发现，executor因为空闲被remove时，还有很多待运行的task。为什么executor会空闲长达30s不接受任务运行呢？
* executor只剩下一个的时候，待运行的task数量也还有很多。那为什么没按照动态资源的规则继续申请新的executor呢？

### 动态资源相关原理

#### 初始executor数量

无论是否开启动态资源，Spark Job在启动时都会去申请一定数量的executor。在开启动态资源时，这个数量由三个配置参数共同决定：
* spark.executor.instances
* spark.dynamicAllocation.minExecutors
* spark.dynamicAllocation.initialExecutors
如果这三个值都设置了，spark会取其中的最大值最为初始executor的数量。

#### executor数量的变动
在动态资源开启后，executor的数量会随着程序的运行进行增加。动态资源的管理主要由ExecutorAllocationManager类来负责，只有将spark.dynamicAllocation.enabled设置为true，ExecutorAllocationManager类才会初始化并启动。

下面是ExecutorAllocationManager类中一些和executor数量变化有关的参数：

```
//最少executor数量
int minNumExecutors
//最多executor数量
int maxNumExecutors
//目前期望的executor数量，这个的取值会不断变动，一般根据 (正在等待运行的task数量 + 运行中的task数量) 来决定
int numExecutorsTarget = initialNumExecutors
//要申请executor时递增的数量，初始值为1，后面每次申请都会让该值翻倍，也就是慢慢变成 1 2 4 8 16 32 ...
//当然，即使numExecutorsToAdd再大，申请的最终数量也不能超过maxNumExecutors
int numExecutorsToAdd = 1
```

ExecutorAllocationManager类在启动时，会开启一个定时任务线程，线程名为"spark-dynamic-executor-allocation"。该任务每隔100ms运行一次。该任务主要做两件事：
* 检查是否需要申请新的executor
* 检查是否有executor空闲太久，需要kill掉

#### Executor Add

是否需要申请新的executor和参数numExecutorsTarget密切相关。每次定时任务运行时，spark都会先获取需要executor的总数量，计算公式如下:
```
val numRunningOrPendingTasks = listener.totalPendingTasks + listener.totalRunningTasks
(numRunningOrPendingTasks + tasksPerExecutor - 1) / tasksPerExecutor
```

tasksPerExecutor就是每个executor可以运行的task数量。这个值由参数spark.executor.cores和spark.task.cpus相除获得。
获取到当前需要executor的总数量后，就拿这个值和numExecutorsTarget的值相比，如果需要的executor的总数量比numExecutorsTarget小，就说明此时executor数量太多，则主动减少executor数量，并更新numExecutorsTarget的值。
如果该值比numExecutorsTarget多，就可以尝试去申请新的executor。申请前还需要判断numExecutorsTarget是否已经大于等于maxNumExecutors，如果是则不能再申请新的executor。
申请的数量主要和参数numExecutorsToAdd有关，这个参数从1开始。每次申请完，这个参数都会翻倍，也就是1、2、4、8、16、…。另外，申请的时候也会判断申请的数量是否超过了maxNumExecutors来保证不超过maxNumExecutors。

下面列出整个流程的java伪代码：

```
//获取当前需要的executor数量
int maxNumExecutorsNeeded = getExecutorsNeeded()
if(maxNumExecutorsNeeded < numExecutorsTarget){
    //通知集群当前需要的executor数量
    requestTotalExecutors(maxNumExecutorsNeeded)
    numExecutorsTarget = maxNumExecutorsNeeded
}else{
    if (numExecutorsTarget >= maxNumExecutors){
        //不申请任何executor就退出
        return
    }else{
        int newTargetExecutor = mint(numExecutorsTarget + numExecutorsToAdd,maxNumExecutors)
        //通知集群当前需要的executor数量
        requestTotalExecutors(newTargetExecutor)
        numExecutorsTarget = maxNumExecutorsNeeded
        numExecutorsToAdd = numExecutorsToAdd * 2
    }
}
```

注意: requestTotalExecutors方法的入参不是说要再申请几个，而是告诉集群目标需要几个executor。如果这个值比集群目前运行的executor数量还少，集群也不会为了达到这个值强行kill掉一些executor。因此，调用这个方法并不会导致executor数量的减少。

#### Executor Remove

在开启动态资源后，对于那些空闲太久的executor，ExecutorAllocationManager类会主动kill掉这个executor来释放资源。

ExecutorAllocationManager类会维护每个executor的过期时间，这个过期时间由参数spark.dynamicAllocation.executorIdleTimeout或者spark.dynamicAllocation.cachedExecutorIdleTimeout决定，取哪个参数是根据executor有没有缓存来决定的(也就是是否有运行的rdd调用了cache或者persist)。空闲时间超过指定的时间就会被定时任务移除。

其中spark.dynamicAllocation.executorIdleTimeout的默认值是30s，spark.dynamicAllocation.cachedExecutorIdleTimeout的默认值却是无穷大，因此动态资源任务要额外注意spark.dynamicAllocation.cachedExecutorIdleTimeout的设置。

当一个task开始时，会将运行的executor标为忙碌，然后task结束后，将对应的executor标记为空闲。一旦executor被标记为空闲，就会开始计时，直到达到超时时间然后被移除。

```
在Spark-2.1.0版本中有个bug，当remove掉一个Executor时，由于numExecutorsTarget值没有更新，如果此时numExecutorsTarget的值刚好是maxNumExecutors，就会导致明明job需要新的executor，但是就是申请不到的问题（numExecutorsTarget >= maxNumExecutors条件始终成立）。
```

相关issue：https://issues.apache.org/jira/browse/SPARK-21834

### 问题定位

由于集群的spark版本刚好是2.1.0，根据issue https://issues.apache.org/jira/browse/SPARK-21834 ，确实会导致executor remove后无法申请到足够的executor数量的问题。（具体参考上面一节的Executor Remove小节）

上面的issue虽然解释了为什么最后只剩下一个executor在运行，又不申请新的executor的问题。但是还有一个问题还没解决：为什么在待运行的task还有很多的情况下，会有那么多的executor空闲时间超过30s？

仔细看了下spark调度task的相关源码，发现这个问题是由于 Task的本地化调度导致的。

#### Spark Task的本地化调度
先简单说下Task的调度。task调度的源头从CoarseGrainedSchedulerBackend类开始。CoarseGrainedSchedulerBackend类在启动时，会启动一个定时任务线程"driver-revive-thread"，每隔一定时间运行一次，间隔由参数spark.scheduler.revive.interval时间运行一次，默认是1s。

简单来讲，这个定时任务就是收集一下存活的executor，然后最终调用TaskSchedulerImpl#resourceOffers(executors)方法来处理。然后由TaskSchedulerImpl根据调度算法选择需要调度的TaskSetManager列表(排好序的，有先后顺序，这个和参数spark.scheduler.mode配置的调度算法有关)，之后逐个遍历这些TaskSetManager进行调度。另外，每个task运行结束后也会再次调用TaskSchedulerImpl#resourceOffers(executors)方法检查是否要继续调度下一个task。

TaskSetManager是由多个task组成的，代表一个stage的所有task。下面是TaskSetManager的一些关键参数:

```
//PROCESS_LOCAL 级别对应的task列表，key是executorId，value是task列表
private val pendingTasksForExecutor = new HashMap[String, ArrayBuffer[Int]]
//NODE_LOCAL 级别对应的task列表，key是host，value是task列表
private val pendingTasksForHost = new HashMap[String, ArrayBuffer[Int]]
//RACK_LOCAL 级别对应的task列表，key是rack，value是task列表
private val pendingTasksForRack = new HashMap[String, ArrayBuffer[Int]]
//不关心级别的task列表
var pendingTasksWithNoPrefs = new ArrayBuffer[Int]
//所有待执行的task
val allPendingTasks = new ArrayBuffer[Int]
//根据Task的情况获取当前TaskSet对应的几种级别。正常情况下依次是 PROCESS_LOCAL NODE_LOCAL NO_PREF RACK_LOCAL ANY
var myLocalityLevels = List[TaskLocality]
//对应级别的最长等待时间
var localityWaits
//最后一个task的启动时间
var lastLaunchTime
```

本地化级别从低到高依次是 PROCESS_LOCAL、NODE_LOCAL、NO_PREF、RACK_LOCAL、ANY。高级别的task列表都会包含低级别的task列表。也就是说pendingTasksForExecutor中有的task，在pendingTasksForHost中也一定会存在。对于pendingTasksForHost和pendingTasksForRack也是同理。

由于TaskSchedulerImpl调度具体TaskSetManager的代码比较绕，为了方便理解，这里还是直接上伪代码：

```
//遍历当前TaskSetManager中存在的所有本地化级别，一般依次是 PROCESS_LOCAL NODE_LOCAL NO_PREF RACK_LOCAL ANY
//遍历的目的主要是为了优先调度低级别的task（如果TaskSetManager的本地化级别已经推进到高级别，此时就可以返回来调度低级别的task）
for(currentMaxLocality <- taskSet.myLocalityLevels){
    //遍历所有的executor
    for(executor <- executors){
        //调用TaskSetManager的方法获取本次要调度的task列表
        taskDescriptionList = taskSet.resourceOffer(executor.id,execute.host,currentMaxLocality){
            //spark的黑名单机制，如果executor的id或者host在黑名单里面，就过滤
            if(inblack(executor)){
                return null;
            }
             
            allowedLocality = maxLocality
 
            if(maxLocality != TaskLocality.NO_PREF){
                allowedLocality = getAllowedLocalityLevel(curTime){
                    while(currentLocalityIndex < myLocalityLevels.length - 1){
                        //获取当前TaskSetManager所处的级别
                        currentLocality = myLocalityLevels(currentLocalityIndex)
                        //判断该级别是否有任务需要运行
                        moreTasks = moreTasksToRunIn(currentLocality)
                        //如果当前级别没有任务需要运行了，就推进一个级别
                        if(!moreTasks){
                            currentLocalityIndex += 1
                        //判断当前级别的延迟时间是否超过了指定的值
                        }else if(curTime - lastLaunchTime >= localityWaits(currentLocalityIndex)){
                            lastLaunchTime += localityWaits(currentLocalityIndex)
                            currentLocalityIndex += 1
                        }else{
                            return myLocalityLevels(currentLocalityIndex)
                        }
                    }
                    return myLocalityLevels(currentLocalityIndex)
                }
                //尽量调度低级别的task
                allowedLocality = min(allowedLocality , maxLocality)
            }
            //根据allowedLocality级别获取对应的task
            return dequeueTask(execId, host, allowedLocality).map{...}
        }
        //taskDescriptionList 不为空则调度这些task（这里已经知道了哪个task要发给哪个executor调度了）
    }
}
```

从代码可以看出，spark的本地化调度是一个逐级推进的过程。从最低级别的PROCESS_LOCAL开始。推进到下一级有两个条件：

* 该级别的已经没有task需要运行
* 距离上一次该级别的最后一个task运行时间已经超过了spark.locality.wait的时间，这个值默认是3s

如果没有及时推进到下一级，TaskSetManager会不断尝试根据当前的本地性级别取task，如果有一些task已经无法满足该级别了，即使executor空闲再多也不会把这些task分配过去。因此才导致了大量的executor空闲了超过30s的时间。

#### 本地化调度级别没有及时推进导致的问题
结合本地化调度的推进过程又去看了下ApplicationMaster的日志，发现该任务task的运行时间很快，基本都是1秒内完成。并且在NODE_LOCAL这个级别，在许多executor已经没有task可以执行的情况下，有某几个executor还一直在执行。由于没有推进到下一级，那些没有task运行的executor就会空闲在那，最终空闲时间超过30s后被ExecutorAllocationManager给kill了。后面级别也推进到了RACK_LOCAL级别，但是由于https://issues.apache.org/jira/browse/SPARK-21834 这个issue，executor也没有恢复回来，一直保持在一个。

这个问题的形成主要有两个：

* task运行过快，低于spark.locality.wait的值
* 在NODE_LOCAL级别上，某几个executor长时间有task运行，其他大量的executor又没有task运行。可能是刚好如此分布，也可能是datanode数据倾斜造成的。

### 解决方案

#### 问题一
executor因为空闲被remove时，还有很多待运行的task。为什么executor会空闲长达30s不接受任务运行呢？

* 调低spark.locality.wait的值，让本地化级别可以较快的推进到下一级。但是这样也会减弱spark的数据本地化机制，导致任务运行变慢。
* 如果是datanode的数据倾斜导致的，可以对hdfs集群进行一次balance。
* 如果修复了问题二的话，其实这个问题只会导致任务运行时间稍微变慢一点，并不会对任务有太大的影响。

#### 问题二
executor只剩下一个的时候，待运行的task数量也还有很多。那为什么没按照动态资源的规则继续申请新的executor呢？

根据
https://issues.apache.org/jira/browse/SPARK-21834
的patch即可解决

### 总结
其实问题一严格来说也不能算bug，一定程度上只能说是策略的问题。
如果解决了问题二，问题一最多导致任务的运行效率小幅度降低。因此，如果解决了问题二，对于问题一我们可以采用参数调优的方式来优化任务的性能。

