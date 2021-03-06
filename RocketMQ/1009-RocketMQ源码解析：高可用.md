-------

![](http://www.yunai.me/images/common/wechat_mp.jpeg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。

-------


- [1. 概述](#)
- [2. Namesrv 高可用](#)
	- [2.1 Broker 注册到 Namesrv](#)
	- [2.2 Producer、Consumer 访问 Namesrv](#)
- [3. Broker 高可用](#)
	- [3.2 Broker 主从](#)
		- [3.1.1 配置](#)
		- [3.1.2 组件](#)
		- [3.1.3 通信协议](#)
		- [3.1.4 Slave](#)
		- [3.1.5 Master](#)
		- [3.1.6 Master_SYNC](#)
	- [3.2 Producer 发送消息](#)
	- [3.3 Consumer 消费消息](#)
- [4. 总结](#)

# 1. 概述

本文主要解析 `Namesrv`、`Broker` 如何实现高可用，`Producer`、`Consumer` 怎么与它们通信保证高可用。

# 2. Namesrv 高可用

**启动多个 `Namesrv` 实现高可用。**  
相较于 `Zookeeper`、`Consul`、`Etcd` 等，`Namesrv` 是一个**超轻量级**的注册中心，提供**命名服务**。

## 2.1 Broker 注册到 Namesrv

* 📌 **多个 `Namesrv` 之间，没有任何关系（不存在类似 `Zookeeper` 的 `Leader`/`Follower` 等角色），不进行通信与数据同步。通过 `Broker` 循环注册多个 `Namesrv`。**

```Java
  1: // ⬇️⬇️⬇️【BrokerOuterAPI.java】
  2: public RegisterBrokerResult registerBrokerAll(
  3:     final String clusterName,
  4:     final String brokerAddr,
  5:     final String brokerName,
  6:     final long brokerId,
  7:     final String haServerAddr,
  8:     final TopicConfigSerializeWrapper topicConfigWrapper,
  9:     final List<String> filterServerList,
 10:     final boolean oneway,
 11:     final int timeoutMills) {
 12:     RegisterBrokerResult registerBrokerResult = null;
 13: 
 14:     List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
 15:     if (nameServerAddressList != null) {
 16:         for (String namesrvAddr : nameServerAddressList) { // 循环多个 Namesrv
 17:             try {
 18:                 RegisterBrokerResult result = this.registerBroker(namesrvAddr, clusterName, brokerAddr, brokerName, brokerId,
 19:                     haServerAddr, topicConfigWrapper, filterServerList, oneway, timeoutMills);
 20:                 if (result != null) {
 21:                     registerBrokerResult = result;
 22:                 }
 23: 
 24:                 log.info("register broker to name server {} OK", namesrvAddr);
 25:             } catch (Exception e) {
 26:                 log.warn("registerBroker Exception, {}", namesrvAddr, e);
 27:             }
 28:         }
 29:     }
 30: 
 31:     return registerBrokerResult;
 32: }
```

## 2.2 Producer、Consumer 访问 Namesrv

* 📌 **`Producer`、`Consumer` 从 `Namesrv`列表选择一个可连接的进行通信。**

```Java
  1: // ⬇️⬇️⬇️【NettyRemotingClient.java】
  2: private Channel getAndCreateNameserverChannel() throws InterruptedException {
  3:     // 返回已选择、可连接Namesrv
  4:     String addr = this.namesrvAddrChoosed.get();
  5:     if (addr != null) {
  6:         ChannelWrapper cw = this.channelTables.get(addr);
  7:         if (cw != null && cw.isOK()) {
  8:             return cw.getChannel();
  9:         }
 10:     }
 11:     //
 12:     final List<String> addrList = this.namesrvAddrList.get();
 13:     if (this.lockNamesrvChannel.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
 14:         try {
 15:             // 返回已选择、可连接的Namesrv
 16:             addr = this.namesrvAddrChoosed.get();
 17:             if (addr != null) {
 18:                 ChannelWrapper cw = this.channelTables.get(addr);
 19:                 if (cw != null && cw.isOK()) {
 20:                     return cw.getChannel();
 21:                 }
 22:             }
 23:             // 从【Namesrv列表】中选择一个连接的返回
 24:             if (addrList != null && !addrList.isEmpty()) {
 25:                 for (int i = 0; i < addrList.size(); i++) {
 26:                     int index = this.namesrvIndex.incrementAndGet();
 27:                     index = Math.abs(index);
 28:                     index = index % addrList.size();
 29:                     String newAddr = addrList.get(index);
 30: 
 31:                     this.namesrvAddrChoosed.set(newAddr);
 32:                     Channel channelNew = this.createChannel(newAddr);
 33:                     if (channelNew != null)
 34:                         return channelNew;
 35:                 }
 36:             }
 37:         } catch (Exception e) {
 38:             log.error("getAndCreateNameserverChannel: create name server channel exception", e);
 39:         } finally {
 40:             this.lockNamesrvChannel.unlock();
 41:         }
 42:     } else {
 43:         log.warn("getAndCreateNameserverChannel: try to lock name server, but timeout, {}ms", LOCK_TIMEOUT_MILLIS);
 44:     }
 45: 
 46:     return null;
 47: }
```

# 3. Broker 高可用

**启动多个 `Broker分组` 形成 `集群` 实现高可用。**  
**`Broker分组` = `Master节点`x1 + `Slave节点`xN。**  
类似 `MySQL`，`Master节点` 提供**读写**服务，`Slave节点` 只提供**读**服务。  

## 3.2 Broker 主从

* **每个分组，`Master`节点 不断发送新的 `CommitLog` 给 `Slave`节点。 `Slave`节点 不断上报本地的 `CommitLog` 已经同步到的位置给 `Master`节点。**
* **`Broker分组` 与 `Broker分组` 之间没有任何关系，不进行通信与数据同步。**
* **消费进度 目前不支持 `Master`/`Slave` 同步。**

集群内，`Master`节点 有**两种**类型：`Master_SYNC`、`Master_ASYNC`：前者在 `Producer` 发送消息时，等待 `Slave`节点 存储完毕后再返回发送结果，而后者不需要等待。


### 3.1.1 配置

目前官方提供三套配置：

* **2m-2s-async**

| brokerClusterName| brokerName | brokerRole | brokerId |
| --- | --- | --- | --- |
| DefaultCluster | broker-a | ASYNC_MASTER | 0 |
| DefaultCluster | broker-a | SLAVE | 1 |
| DefaultCluster | broker-b | ASYNC_MASTER | 0 |
| DefaultCluster | broker-b | SLAVE | 1 |

* **2m-2s-sync**

| brokerClusterName| brokerName | brokerRole | brokerId |
| ---| --- | --- | --- |
| DefaultCluster | broker-a | SYNC_MASTER | 0 |
| DefaultCluster | broker-a | SLAVE | 1 |
| DefaultCluster | broker-b | SYNC_MASTER | 0 |
| DefaultCluster | broker-b | SLAVE | 1 |

* **2m-noslave**

| brokerClusterName| brokerName | brokerRole | brokerId |
| ---| --- | --- | --- |
| DefaultCluster | broker-a | ASYNC_MASTER | 0 |
| DefaultCluster | broker-b | ASYNC_MASTER | 0 |

### 3.1.2 组件

再看具体实现代码之前，我们来看看 `Master`/`Slave`节点 包含的组件：  
![HA组件图.png](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1009/HA组件图.png)

* `Master`节点
    * `AcceptSocketService` ：接收 `Slave`节点 连接。
    * `HAConnection`
        * `ReadSocketService` ：**读**来自 `Slave`节点 的数据。 
        * `WriteSocketService` ：**写**到往 `Slave`节点 的数据。
* `Slave`节点
    * `HAClient` ：对 `Master`节点 连接、读写数据。

### 3.1.3 通信协议

`Master`节点 与 `Slave`节点 **通信协议**很简单，只有如下两条。

| 对象 | 用途 | 第几位 | 字段 | 数据类型 | 字节数 | 说明
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Slave=>Master | 上报CommitLog**已经**同步到的**物理**位置 |  |  |  |  |  |
|  | | 0 | maxPhyOffset  |  Long | 8 | CommitLog最大物理位置 |
| Master=>Slave | 传输新的 `CommitLog` 数据 |  |  |  |  |  |
| | | 0 | fromPhyOffset | Long | 8 | CommitLog开始物理位置 | 
| | | 1 | size | Int | 4 | 传输CommitLog数据长度 | 
| | | 2 | body | Bytes | size | 传输CommitLog数据 | 

### 3.1.4 Slave

![HAClient顺序图](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1009/HAClient顺序图.png)

-------

* **`Slave` 主循环，实现了**不断不断不断**从 `Master` 传输 `CommitLog` 数据，上传 `Master` 自己本地的 `CommitLog` 已经同步物理位置。**

```Java
  1: // ⬇️⬇️⬇️【HAClient.java】
  2: public void run() {
  3:     log.info(this.getServiceName() + " service started");
  4: 
  5:     while (!this.isStopped()) {
  6:         try {
  7:             if (this.connectMaster()) {
  8:                 // 若到满足上报间隔，上报到Master进度
  9:                 if (this.isTimeToReportOffset()) {
 10:                     boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
 11:                     if (!result) {
 12:                         this.closeMaster();
 13:                     }
 14:                 }
 15: 
 16:                 this.selector.select(1000);
 17: 
 18:                 // 处理读取事件
 19:                 boolean ok = this.processReadEvent();
 20:                 if (!ok) {
 21:                     this.closeMaster();
 22:                 }
 23: 
 24:                 // 若进度有变化，上报到Master进度
 25:                 if (!reportSlaveMaxOffsetPlus()) {
 26:                     continue;
 27:                 }
 28: 
 29:                 // Master过久未返回数据，关闭连接
 30:                 long interval = HAService.this.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;
 31:                 if (interval > HAService.this.getDefaultMessageStore().getMessageStoreConfig()
 32:                     .getHaHousekeepingInterval()) {
 33:                     log.warn("HAClient, housekeeping, found this connection[" + this.masterAddress
 34:                         + "] expired, " + interval);
 35:                     this.closeMaster();
 36:                     log.warn("HAClient, master not response some time, so close connection");
 37:                 }
 38:             } else {
 39:                 this.waitForRunning(1000 * 5);
 40:             }
 41:         } catch (Exception e) {
 42:             log.warn(this.getServiceName() + " service has exception. ", e);
 43:             this.waitForRunning(1000 * 5);
 44:         }
 45:     }
 46: 
 47:     log.info(this.getServiceName() + " service end");
 48: }
```

* 第 8 至 14 行 ：**固定间隔（默认5s）**向 `Master` 上报 `Slave` 本地 `CommitLog` 已经同步到的物理位置。该操作还有**心跳**的作用。
* 第 16 至 22 行 ：处理 `Master` 传输 `Slave` 的 `CommitLog` 数据。

-------

* **我们来看看 `#dispatchReadRequest(...)` 与 `#reportSlaveMaxOffset(...)` 是怎么实现的。**

```Java
  1: // 【HAClient.java】
  2: /**
  3:  * 读取Master传输的CommitLog数据，并返回是异常
  4:  * 如果读取到数据，写入CommitLog
  5:  * 异常原因：
  6:  *   1. Master传输来的数据offset 不等于 Slave的CommitLog数据最大offset
  7:  *   2. 上报到Master进度失败
  8:  *
  9:  * @return 是否异常
 10:  */
 11: private boolean dispatchReadRequest() {
 12:     final int msgHeaderSize = 8 + 4; // phyoffset + size
 13:     int readSocketPos = this.byteBufferRead.position();
 14: 
 15:     while (true) {
 16:         // 读取到请求
 17:         int diff = this.byteBufferRead.position() - this.dispatchPostion;
 18:         if (diff >= msgHeaderSize) {
 19:             // 读取masterPhyOffset、bodySize。使用dispatchPostion的原因是：处理数据“粘包”导致数据读取不完整。
 20:             long masterPhyOffset = this.byteBufferRead.getLong(this.dispatchPostion);
 21:             int bodySize = this.byteBufferRead.getInt(this.dispatchPostion + 8);
 22:             // 校验 Master传输来的数据offset 是否和 Slave的CommitLog数据最大offset 是否相同。
 23:             long slavePhyOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();
 24:             if (slavePhyOffset != 0) {
 25:                 if (slavePhyOffset != masterPhyOffset) {
 26:                     log.error("master pushed offset not equal the max phy offset in slave, SLAVE: "
 27:                         + slavePhyOffset + " MASTER: " + masterPhyOffset);
 28:                     return false;
 29:                 }
 30:             }
 31:             // 读取到消息
 32:             if (diff >= (msgHeaderSize + bodySize)) {
 33:                 // 写入CommitLog
 34:                 byte[] bodyData = new byte[bodySize];
 35:                 this.byteBufferRead.position(this.dispatchPostion + msgHeaderSize);
 36:                 this.byteBufferRead.get(bodyData);
 37:                 HAService.this.defaultMessageStore.appendToCommitLog(masterPhyOffset, bodyData);
 38:                 // 设置处理到的位置
 39:                 this.byteBufferRead.position(readSocketPos);
 40:                 this.dispatchPostion += msgHeaderSize + bodySize;
 41:                 // 上报到Master进度
 42:                 if (!reportSlaveMaxOffsetPlus()) {
 43:                     return false;
 44:                 }
 45:                 // 继续循环
 46:                 continue;
 47:             }
 48:         }
 49: 
 50:         // 空间写满，重新分配空间
 51:         if (!this.byteBufferRead.hasRemaining()) {
 52:             this.reallocateByteBuffer();
 53:         }
 54: 
 55:         break;
 56:     }
 57: 
 58:     return true;
 59: }
 60: 
 61: /**
 62:  * 上报进度
 63:  *
 64:  * @param maxOffset 进度
 65:  * @return 是否上报成功
 66:  */
 67: private boolean reportSlaveMaxOffset(final long maxOffset) {
 68:     this.reportOffset.position(0);
 69:     this.reportOffset.limit(8);
 70:     this.reportOffset.putLong(maxOffset);
 71:     this.reportOffset.position(0);
 72:     this.reportOffset.limit(8);
 73: 
 74:     for (int i = 0; i < 3 && this.reportOffset.hasRemaining(); i++) {
 75:         try {
 76:             this.socketChannel.write(this.reportOffset);
 77:         } catch (IOException e) {
 78:             log.error(this.getServiceName()
 79:                 + "reportSlaveMaxOffset this.socketChannel.write exception", e);
 80:             return false;
 81:         }
 82:     }
 83: 
 84:     return !this.reportOffset.hasRemaining();
 85: }
```

### 3.1.5 Master

* **`ReadSocketService` 逻辑同 `HAClient#processReadEvent(...)` 基本相同，我们直接看代码。**

```Java
  1: // ⬇️⬇️⬇️【ReadSocketService.java】
  2: private boolean processReadEvent() {
  3:     int readSizeZeroTimes = 0;
  4: 
  5:     // 清空byteBufferRead
  6:     if (!this.byteBufferRead.hasRemaining()) {
  7:         this.byteBufferRead.flip();
  8:         this.processPostion = 0;
  9:     }
 10: 
 11:     while (this.byteBufferRead.hasRemaining()) {
 12:         try {
 13:             int readSize = this.socketChannel.read(this.byteBufferRead);
 14:             if (readSize > 0) {
 15:                 readSizeZeroTimes = 0;
 16: 
 17:                 // 设置最后读取时间
 18:                 this.lastReadTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
 19: 
 20:                 if ((this.byteBufferRead.position() - this.processPostion) >= 8) {
 21:                     // 读取Slave 请求来的CommitLog的最大位置
 22:                     int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % 8);
 23:                     long readOffset = this.byteBufferRead.getLong(pos - 8);
 24:                     this.processPostion = pos;
 25: 
 26:                     // 设置Slave CommitLog的最大位置
 27:                     HAConnection.this.slaveAckOffset = readOffset;
 28: 
 29:                     // 设置Slave 第一次请求的位置
 30:                     if (HAConnection.this.slaveRequestOffset < 0) {
 31:                         HAConnection.this.slaveRequestOffset = readOffset;
 32:                         log.info("slave[" + HAConnection.this.clientAddr + "] request offset " + readOffset);
 33:                     }
 34: 
 35:                     // 通知目前Slave进度。主要用于Master节点为同步类型的。
 36:                     HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
 37:                 }
 38:             } else if (readSize == 0) {
 39:                 if (++readSizeZeroTimes >= 3) {
 40:                     break;
 41:                 }
 42:             } else {
 43:                 log.error("read socket[" + HAConnection.this.clientAddr + "] < 0");
 44:                 return false;
 45:             }
 46:         } catch (IOException e) {
 47:             log.error("processReadEvent exception", e);
 48:             return false;
 49:         }
 50:     }
 51: 
 52:     return true;
 53: }
```

-------

* **`WriteSocketService` 计算 `Slave`开始同步的位置后，不断向 `Slave` 传输新的 `CommitLog`数据。**

![HA.WriteSocketService流程图](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1009/HA.WriteSocketService流程图.png)

```Java
  1: // ⬇️⬇️⬇️【WriteSocketService.java】
  2: @Override
  3: public void run() {
  4:     HAConnection.log.info(this.getServiceName() + " service started");
  5: 
  6:     while (!this.isStopped()) {
  7:         try {
  8:             this.selector.select(1000);
  9: 
 10:             // 未获得Slave读取进度请求，sleep等待。
 11:             if (-1 == HAConnection.this.slaveRequestOffset) {
 12:                 Thread.sleep(10);
 13:                 continue;
 14:             }
 15: 
 16:             // 计算初始化nextTransferFromWhere
 17:             if (-1 == this.nextTransferFromWhere) {
 18:                 if (0 == HAConnection.this.slaveRequestOffset) {
 19:                     long masterOffset = HAConnection.this.haService.getDefaultMessageStore().getCommitLog().getMaxOffset();
 20:                     masterOffset = masterOffset - (masterOffset % HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getMapedFileSizeCommitLog());
 21:                     if (masterOffset < 0) {
 22:                         masterOffset = 0;
 23:                     }
 24: 
 25:                     this.nextTransferFromWhere = masterOffset;
 26:                 } else {
 27:                     this.nextTransferFromWhere = HAConnection.this.slaveRequestOffset;
 28:                 }
 29: 
 30:                 log.info("master transfer data from " + this.nextTransferFromWhere + " to slave[" + HAConnection.this.clientAddr
 31:                     + "], and slave request " + HAConnection.this.slaveRequestOffset);
 32:             }
 33: 
 34:             if (this.lastWriteOver) {
 35:                 long interval = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;
 36:                 if (interval > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaSendHeartbeatInterval()) { // 心跳
 37: 
 38:                     // Build Header
 39:                     this.byteBufferHeader.position(0);
 40:                     this.byteBufferHeader.limit(headerSize);
 41:                     this.byteBufferHeader.putLong(this.nextTransferFromWhere);
 42:                     this.byteBufferHeader.putInt(0);
 43:                     this.byteBufferHeader.flip();
 44: 
 45:                     this.lastWriteOver = this.transferData();
 46:                     if (!this.lastWriteOver)
 47:                         continue;
 48:                 }
 49:             } else { // 未传输完成，继续传输
 50:                 this.lastWriteOver = this.transferData();
 51:                 if (!this.lastWriteOver)
 52:                     continue;
 53:             }
 54: 
 55:             // 选择新的CommitLog数据进行传输
 56:             SelectMappedBufferResult selectResult =
 57:                 HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
 58:             if (selectResult != null) {
 59:                 int size = selectResult.getSize();
 60:                 if (size > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize()) {
 61:                     size = HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize();
 62:                 }
 63: 
 64:                 long thisOffset = this.nextTransferFromWhere;
 65:                 this.nextTransferFromWhere += size;
 66: 
 67:                 selectResult.getByteBuffer().limit(size);
 68:                 this.selectMappedBufferResult = selectResult;
 69: 
 70:                 // Build Header
 71:                 this.byteBufferHeader.position(0);
 72:                 this.byteBufferHeader.limit(headerSize);
 73:                 this.byteBufferHeader.putLong(thisOffset);
 74:                 this.byteBufferHeader.putInt(size);
 75:                 this.byteBufferHeader.flip();
 76: 
 77:                 this.lastWriteOver = this.transferData();
 78:             } else { // 没新的消息，挂起等待
 79:                 HAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
 80:             }
 81:         } catch (Exception e) {
 82: 
 83:             HAConnection.log.error(this.getServiceName() + " service has exception.", e);
 84:             break;
 85:         }
 86:     }
 87: 
 88:     // 断开连接 & 暂停写线程 & 暂停读线程 & 释放CommitLog
 89:     if (this.selectMappedBufferResult != null) {
 90:         this.selectMappedBufferResult.release();
 91:     }
 92: 
 93:     this.makeStop();
 94: 
 95:     readSocketService.makeStop();
 96: 
 97:     haService.removeConnection(HAConnection.this);
 98: 
 99:     SelectionKey sk = this.socketChannel.keyFor(this.selector);
100:     if (sk != null) {
101:         sk.cancel();
102:     }
103: 
104:     try {
105:         this.selector.close();
106:         this.socketChannel.close();
107:     } catch (IOException e) {
108:         HAConnection.log.error("", e);
109:     }
110: 
111:     HAConnection.log.info(this.getServiceName() + " service end");
112: }
113: 
114: /**
115:  * 传输数据
116:  */
117: private boolean transferData() throws Exception {
118:     int writeSizeZeroTimes = 0;
119:     // Write Header
120:     while (this.byteBufferHeader.hasRemaining()) {
121:         int writeSize = this.socketChannel.write(this.byteBufferHeader);
122:         if (writeSize > 0) {
123:             writeSizeZeroTimes = 0;
124:             this.lastWriteTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
125:         } else if (writeSize == 0) {
126:             if (++writeSizeZeroTimes >= 3) {
127:                 break;
128:             }
129:         } else {
130:             throw new Exception("ha master write header error < 0");
131:         }
132:     }
133: 
134:     if (null == this.selectMappedBufferResult) {
135:         return !this.byteBufferHeader.hasRemaining();
136:     }
137: 
138:     writeSizeZeroTimes = 0;
139: 
140:     // Write Body
141:     if (!this.byteBufferHeader.hasRemaining()) {
142:         while (this.selectMappedBufferResult.getByteBuffer().hasRemaining()) {
143:             int writeSize = this.socketChannel.write(this.selectMappedBufferResult.getByteBuffer());
144:             if (writeSize > 0) {
145:                 writeSizeZeroTimes = 0;
146:                 this.lastWriteTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
147:             } else if (writeSize == 0) {
148:                 if (++writeSizeZeroTimes >= 3) {
149:                     break;
150:                 }
151:             } else {
152:                 throw new Exception("ha master write body error < 0");
153:             }
154:         }
155:     }
156: 
157:     boolean result = !this.byteBufferHeader.hasRemaining() && !this.selectMappedBufferResult.getByteBuffer().hasRemaining();
158: 
159:     if (!this.selectMappedBufferResult.getByteBuffer().hasRemaining()) {
160:         this.selectMappedBufferResult.release();
161:         this.selectMappedBufferResult = null;
162:     }
163: 
164:     return result;
165: }
```

### 3.1.6 Master_SYNC

* **`Producer` 发送消息时，`Master_SYNC`节点 会等待 `Slave`节点 存储完毕后再返回发送结果。**

核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【CommitLog.java】
  2: public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
  3:     // ....省略处理发送代码 
  4:     // Synchronous write double 如果是同步Master，同步到从节点
  5:     if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
  6:         HAService service = this.defaultMessageStore.getHaService();
  7:         if (msg.isWaitStoreMsgOK()) {
  8:             // Determine whether to wait
  9:             if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
 10:                 if (null == request) {
 11:                     request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
 12:                 }
 13:                 service.putRequest(request);
 14: 
 15:                 // 唤醒WriteSocketService
 16:                 service.getWaitNotifyObject().wakeupAll();
 17: 
 18:                 boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
 19:                 if (!flushOK) {
 20:                     log.error("do sync transfer other node, wait return, but failed, topic: " + msg.getTopic() + " tags: "
 21:                         + msg.getTags() + " client address: " + msg.getBornHostString());
 22:                     putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
 23:                 }
 24:             }
 25:             // Slave problem
 26:             else {
 27:                 // Tell the producer, slave not available
 28:                 putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
 29:             }
 30:         }
 31:     }
 32: 
 33:     return putMessageResult;
 34: }
```

* 第 16 行 ：唤醒 `WriteSocketService`。
    * 唤醒后，`WriteSocketService` 挂起等待新消息结束，`Master` 传输 `Slave` 新的 `CommitLog` 数据。
    * `Slave` 收到数据后，**立即**上报最新的 `CommitLog` 同步进度到 `Master`。`ReadSocketService` 唤醒**第 18 行**：`request#waitForFlush(...)`。

我们来看下 `GroupTransferService` 的核心逻辑代码：

```Java
  1: // ⬇️⬇️⬇️【GroupTransferService.java】
  2: private void doWaitTransfer() {
  3:     synchronized (this.requestsRead) {
  4:         if (!this.requestsRead.isEmpty()) {
  5:             for (CommitLog.GroupCommitRequest req : this.requestsRead) {
  6:                 // 等待Slave上传进度
  7:                 boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
  8:                 for (int i = 0; !transferOK && i < 5; i++) {
  9:                     this.notifyTransferObject.waitForRunning(1000); // 唤醒
 10:                     transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
 11:                 }
 12: 
 13:                 if (!transferOK) {
 14:                     log.warn("transfer messsage to slave timeout, " + req.getNextOffset());
 15:                 }
 16: 
 17:                 // 唤醒请求，并设置是否Slave同步成功
 18:                 req.wakeupCustomer(transferOK);
 19:             }
 20: 
 21:             this.requestsRead.clear();
 22:         }
 23:     }
 24: }
```

## 3.2 Producer 发送消息

* **`Producer` 发送消息时，会对 `Broker`集群 的所有队列进行选择。**

核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【DefaultMQProducerImpl.java】
  2: private SendResult sendDefaultImpl(//
  3:     Message msg, //
  4:     final CommunicationMode communicationMode, //
  5:     final SendCallback sendCallback, //
  6:     final long timeout//
  7: ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  8:     // .... 省略：处理【校验逻辑】
  9:     // 获取 Topic路由信息
 10:     TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
 11:     if (topicPublishInfo != null && topicPublishInfo.ok()) {
 12:         MessageQueue mq = null; // 最后选择消息要发送到的队列
 13:         Exception exception = null;
 14:         SendResult sendResult = null; // 最后一次发送结果
 15:         int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1; // 同步多次调用
 16:         int times = 0; // 第几次发送
 17:         String[] brokersSent = new String[timesTotal]; // 存储每次发送消息选择的broker名
 18:         // 循环调用发送消息，直到成功
 19:         for (; times < timesTotal; times++) {
 20:             String lastBrokerName = null == mq ? null : mq.getBrokerName();
 21:             MessageQueue tmpmq = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName); // 选择消息要发送到的队列
 22:             if (tmpmq != null) {
 23:                 mq = tmpmq;
 24:                 brokersSent[times] = mq.getBrokerName();
 25:                 try {
 26:                     beginTimestampPrev = System.currentTimeMillis();
 27:                     // 调用发送消息核心方法
 28:                     sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
 29:                     endTimestamp = System.currentTimeMillis();
 30:                     // 更新Broker可用性信息
 31:                     this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
 32:                     // .... 省略：处理【发送返回结果】
 33:                     }
 34:                 } catch (e) { // .... 省略：处理【异常】
 35:                     
 36:                 }
 37:             } else {
 38:                 break;
 39:             }
 40:         }
 41:         // .... 省略：处理【发送返回结果】
 42:     }
 43:     // .... 省略：处理【找不到消息路由】
 44: }
```

如下是调试 `#sendDefaultImpl(...)` 时 `TopicPublishInfo` 的结果，`Producer` 获得到了 `broker-a`,`broker-b` 两个 `Broker`分组 的消息队列：
![Producer.TopicPublishInfo.调试.png](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1009/Producer.TopicPublishInfo.调试.png)

## 3.3 Consumer 消费消息

* **`Consumer` 消费消息时，会对 `Broker`集群 的所有队列进行选择。**

# 4. 总结

![HA总结.jpeg](https://raw.githubusercontent.com/YunaiV/Blog/master/RocketMQ/images/1009/HA总结.jpeg)


