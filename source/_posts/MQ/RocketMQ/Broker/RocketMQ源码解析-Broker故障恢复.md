---
title: RocketMQ源码解析-Broker故障恢复
categories:
  - MQ
  - RocketMQ
  - Broker
tags:
  - MQ
  - RocketMQ
  - Broker源码解析
  - Broker工作机制
  - Broker故障恢复
abbrlink: 136d3803
date: 2020-04-02 22:21:00
---

> 以下源码基于Rocket MQ 4.7.0

RocketMQ正常退出或者异常退出的时候，如果重新启动那么怎么恢复数据。接下来通过代码来分析这个过程。
### Broker启动
在broker第一次启动或者重新启动的时候会调用这样的一段代码：

```java
//BrokerControllerinitialize 中的方法
public boolean initialize() throws CloneNotSupportedException {
     result = result && this.messageStore.load();
}
```
从上面的代码可以知道，在broker进行初始的时候，会 **`MessageStore#load`** 方法，这个方法的默认实现为 **DefaultMessageStore** 。接下来看一下 load方法这里就是Broker恢复的入口：

```java
    public boolean load() {
        boolean result = true;

        try {
            //通过判断abort文件是否存在来判断是否正常退出
            boolean lastExitOK = !this.isTempFileExist();

            if (null != scheduleMessageService) {
                result = result && this.scheduleMessageService.load();
            }

            // 加载CommitLog
            result = result && this.commitLog.load();

            //加载ConsumeQueue
            result = result && this.loadConsumeQueue();

            if (result) {
                this.storeCheckpoint =
                    new StoreCheckpoint(StorePathConfigHelper.getStoreCheckpoint(this.messageStoreConfig.getStorePathRootDir()));

                this.indexService.load(lastExitOK);
                //恢复入口
                this.recover(lastExitOK);

                this.getMaxPhyOffset());
            }
        } catch (Exception e) {
            result = false;
        }

        if (!result) {
            this.allocateMappedFileService.shutdown();
        }

        return result;
    }
```
通过上面的代码可以知道恢复是通过 **`recover`** 方法来处理。

```java
    private void recover(final boolean lastExitOK) {
        //获取ConsumeQueue最大的物理偏移量--这个也是CommitLog中物理偏移量(后续会有测试的打印代码)
        long maxPhyOffsetOfConsumeQueue = this.recoverConsumeQueue();

        if (lastExitOK) {
            //正常退出处理
            this.commitLog.recoverNormally(maxPhyOffsetOfConsumeQueue);
        } else {
            //异常退出处理
            this.commitLog.recoverAbnormally(maxPhyOffsetOfConsumeQueue);
        }

        this.recoverTopicQueueTable();
    }
```
> 在RocketMQ的4.7.0版本中CommitLog#recoverAbnormally方法显示为过期，这里暂时就不去分析这个情况。等后续看这里如何处理。

下面来通过添加测试代码的方式说明一下 **`maxPhyOffsetOfConsumeQueue`** 到底是什么值。首先能在 **`recover`** 中添加如下代码然后打包源码：

![](https://github.com/mxsm/document/blob/master/image/MQ/RocketMQ/recover1.png?raw=true)

然后启动broker,我这里启动这个值为384

![](https://github.com/mxsm/document/blob/master/image/MQ/RocketMQ/recover2.png?raw=true)

然后通过客户端在产生一条消息到Broker

![](https://github.com/mxsm/document/blob/master/image/MQ/RocketMQ/recover3.png?raw=true)

通过监控broker日志(这个也是自己添加的)，存入CommitLog的大小为192字节。

![](https://github.com/mxsm/document/blob/master/image/MQ/RocketMQ/recover4.png?raw=true)

然后重启Broker发现这个 **`maxPhyOffsetOfConsumeQueue`**  变为了 **576** 。![](https://github.com/mxsm/document/blob/master/image/MQ/RocketMQ/recover5.png?raw=true)

通过这个日志的打印说明了 **`maxPhyOffsetOfConsumeQueue`**  为CommitLog日志中的物理偏移量。