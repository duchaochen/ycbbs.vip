作者：唯有坚持不懈 | 出处：[https://blog.csdn.net/prestigeding/article/details/78888290](https://blog.csdn.net/prestigeding/article/details/78888290)

--------------------

RocketMQ事务消息阅读目录指引：  
[RocketMQ源码分析之从官方示例窥探RocketMQ事务消息实现基本思想][RocketMQ_RocketMQ]  
[RocketMQ源码分析之RocketMQ事务消息实现原理上篇][RocketMQ_RocketMQ 1]  
[RocketMQ源码分析之RocketMQ事务消息实现原理中篇----事务消息状态回查][RocketMQ_RocketMQ_----]  
[RocketMQ源码分析之事务消息实现原理下篇-消息服务器Broker提交回滚事务实现原理][RocketMQ_-_Broker]  
[RocketMQ事务消息实战][RocketMQ]

--------------------

   上节已经梳理了RocketMQ发送事务消息的流程（基于二阶段提交），本节将继续深入学习事务状态消息回查，我们知道，第一次提交到消息服务器，消息的主题被替换为`RMQ\_SYS\_TRANS\_HALF\_TOPIC`，当执行本地事务，如果返回本地事务状态为`UN\_KNOW`时，第二次提交到服务器时将不会做任何操作，也就是消息还存在与`RMQ\_SYS\_TRANS\_HALF\_TOPIC`主题中，并不能被消息消费者消费，那这些消息最终如何被提交或回滚呢？
   
   原来`RocketMQ`使用`TransactionalMessageCheckService`线程定时去检测RMQ\_SYS\_TRANS\_HALF\_TOPIC主题中的消息，回查消息的事务状态。`TransactionalMessageCheckService`的检测频率默认1分钟，可通过在broker.conf文件中设置transactionCheckInterval的值来改变默认值，单位为毫秒。

温馨提示：文末附有流程图。

   `TransactionalMessageCheckService\#onWaitEnd`

```
protected void onWaitEnd() {
        long timeout = brokerController.getBrokerConfig().getTransactionTimeOut();         // @1
        int checkMax = brokerController.getBrokerConfig().getTransactionCheckMax();    // @2
        long begin = System.currentTimeMillis();
        log.info("Begin to check prepare message, begin time:{}", begin);
        this.brokerController.getTransactionalMessageService().check(timeout, checkMax, this.brokerController.getTransactionalMessageCheckListener());       // @3
        log.info("End to check prepare message, consumed time:{}", System.currentTimeMillis() - begin);
    }
```

   代码@1：从 `broker` 配置文件中获取 `transactionTimeOut` 参数值，表示事务的过期时间，一个消息的存储时间 + 该值 大于系统当前时间，才对该消息执行事务状态会查。  
   
   代码@2：从 `broker` 配置文件中获取 `transactionCheckMax` 参数值，表示事务的最大检测次数，如果超过检测次数，消息会默认为丢弃，即rollback消息。  
   
   接下来重点分析 `TransactionalMessageService\#check` 的实现逻辑，其实现类：`org.apache.rocketmq.broker.transaction.queue.TransactionalMessageServiceImpl  `
   `TransactionalMessageServiceImpl\#check`

```
String topic = MixAll.RMQ_SYS_TRANS_HALF_TOPIC;
Set<MessageQueue> msgQueues = transactionalMessageBridge.fetchMessageQueues(topic);
if (msgQueues == null || msgQueues.size() == 0) {
      log.warn("The queue of topic is empty :" + topic);
      return;
}
```

   step1：根据主题名称，获取该主题下所有的消息队列。  `TransactionalMessageServiceImpl\#check`

```
for (MessageQueue messageQueue : msgQueues) {
    // ...
}
```

   Step2：循环遍历消息队列，从单个消息消费队列去获取消息。  ` TransactionalMessageServiceImpl\#check`

```
long startTime = System.currentTimeMillis();
MessageQueue opQueue = getOpQueue(messageQueue);
long halfOffset = transactionalMessageBridge.fetchConsumeOffset(messageQueue);
long opOffset = transactionalMessageBridge.fetchConsumeOffset(opQueue);
log.info("Before check, the queue={} msgOffset={} opOffset={}", messageQueue, halfOffset, opOffset);
if (halfOffset < 0 || opOffset < 0) {
     log.error("MessageQueue: {} illegal offset read: {}, op offset: {},skip this queue", messageQueue, halfOffset, opOffset);
     continue;
}
```

   Step3：获取对应的操作队列，其主题为：`RMQ\_SYS\_TRANS\_OP\_HALF\_TOPIC`，然后获取操作队列的消费进度、待操作的消费队列的消费进度，如果任意一小于0，忽略该消息队列，继续处理下一个队列。  
   `TransactionalMessageServiceImpl\#check`

```
List<Long> doneOpOffset = new ArrayList<>();
HashMap<Long, Long> removeMap = new HashMap<>();
PullResult pullResult = fillOpRemoveMap(removeMap, opQueue, opOffset, halfOffset, doneOpOffset);
if (null == pullResult) {
      log.error("The queue={} check msgOffset={} with opOffset={} failed, pullResult is null",
      messageQueue, halfOffset, opOffset);
      continue;
}
```

   Step4：调用 `fillOpRemoveMap` 主题填充`removeMap`、`doneOpOffset` 数据结构，这里主要的目的是避免重复调用事务回查接口，这里说一下`RMQ\_SYS\_TRANS\_HALF\_TOPIC`、`RMQ\_SYS\_TRANS\_OP\_HALF\_TOPI`C这两个主题的作用。  
   `RMQ\_SYS\_TRANS\_HALF\_TOPIC：prepare`消息的主题，事务消息首先先进入到该主题。  
   `RMQ\_SYS\_TRANS\_OP\_HALF\_TOPIC`：当消息服务器收到事务消息的提交或回滚请求后，会将消息存储在该主题下。  
   `TransactionalMessageServiceImpl\#check`

```
// single thread
int getMessageNullCount = 1;
long newOffset = halfOffset;
long i = halfOffset;                         // @1 
while (true) {                                   
if (System.currentTimeMillis() - startTime > MAX_PROCESS_TIME_LIMIT) {                                        // @2
      	log.info("Queue={} process time reach max={}", messageQueue, MAX_PROCESS_TIME_LIMIT);
        break;
      }
      if (removeMap.containsKey(i)) {         // @3
            log.info("Half offset {} has been committed/rolled back", i);
            removeMap.remove(i);
      } else {
            GetResult getResult = getHalfMsg(messageQueue, i);      // @4
            MessageExt msgExt = getResult.getMsg();   
            if (msgExt == null) {       // @5
            	if (getMessageNullCount++ > MAX_RETRY_COUNT_WHEN_HALF_NULL) {      
                	break;
                }
                if (getResult.getPullResult().getPullStatus() == PullStatus.NO_NEW_MSG) {
                       log.info("No new msg, the miss offset={} in={}, continue check={}, pull result={}", i,
                       messageQueue, getMessageNullCount, getResult.getPullResult());
                       break;
               } else {
                       log.info("Illegal offset, the miss offset={} in={}, continue check={}, pull result={}",
                                    i, messageQueue, getMessageNullCount, getResult.getPullResult());
                       i = getResult.getPullResult().getNextBeginOffset();
                       newOffset = i;
                       continue;
               }
       }

       if (needDiscard(msgExt, transactionCheckMax) || needSkip(msgExt)) {    // @6
                listener.resolveDiscardMsg(msgExt);
                newOffset = i + 1;
                i++;
                continue;
       }
       if (msgExt.getStoreTimestamp() >= startTime) {
               log.info("Fresh stored. the miss offset={}, check it later, store={}", i,
                                new Date(msgExt.getStoreTimestamp()));
               break;
       }

       long valueOfCurrentMinusBorn = System.currentTimeMillis() - msgExt.getBornTimestamp();     // @7
       long checkImmunityTime = transactionTimeout;                                                                           
       String checkImmunityTimeStr = msgExt.getUserProperty(MessageConst.PROPERTY_CHECK_IMMUNITY_TIME_IN_SECONDS);
       if (null != checkImmunityTimeStr) {  // @8
             checkImmunityTime = getImmunityTime(checkImmunityTimeStr, transactionTimeout);
             if (valueOfCurrentMinusBorn < checkImmunityTime) {
                   if (checkPrepareQueueOffset(removeMap, doneOpOffset, msgExt, checkImmunityTime)) {
                          newOffset = i + 1;
                          i++;
                         continue;
                    }
              }
        } else {   // @9
              if ((0 <= valueOfCurrentMinusBorn) && (valueOfCurrentMinusBorn < checkImmunityTime)) {
                    log.info("New arrived, the miss offset={}, check it later checkImmunity={}, born={}", i,
                    checkImmunityTime, new Date(msgExt.getBornTimestamp()));
                    break;
               }
       }
       List<MessageExt> opMsg = pullResult.getMsgFoundList();
       boolean isNeedCheck = (opMsg == null && valueOfCurrentMinusBorn > checkImmunityTime)
                 || (opMsg != null && (opMsg.get(opMsg.size() - 1).getBornTimestamp() - startTime > transactionTimeout))
                 || (valueOfCurrentMinusBorn <= -1);     // @10

        if (isNeedCheck) {
                if (!putBackHalfMsgQueue(msgExt, i)) {    // @11
                       continue;
                }
                listener.resolveHalfMsg(msgExt);
        } else {
                pullResult = fillOpRemoveMap(removeMap, opQueue, pullResult.getNextBeginOffset(), halfOffset, doneOpOffset);   // @12
                log.info("The miss offset:{} in messageQueue:{} need to get more opMsg, result is:{}", i,
                                messageQueue, pullResult);
                continue;
        }
   }
  newOffset = i + 1;
  i++;
}
if (newOffset != halfOffset) {    // @13
     transactionalMessageBridge.updateConsumeOffset(messageQueue, newOffset);
}
long newOpOffset = calculateOpOffset(doneOpOffset, opOffset);
if (newOpOffset != opOffset) {  // @14                       
     transactionalMessageBridge.updateConsumeOffset(opQueue, newOpOffset);
}
```

   本段代码比较长，却是事务状态回查的重点实现。  
   代码@1：先解释几个局部变量的含义。  
   
      `getMessageNullCount` ：获取空消息的次数。  
      `newOffset` ：当前处理RMQ\_SYS\_TRANS\_HALF\_TOPIC\#queueId的最新进度。 
	  
      i：当前处理消息的队列偏移量，其主题依然为RMQ\_SYS\_TRANS\_HALF\_TOPIC。  
	  
   代码@2：这段代码应该不陌生，这是 `RocketMQ` 处理任务的一个通用处理逻辑，就是一个任务处理，可以限制每次最多处理的时间，`RocketMQ`为待检测主题`RMQ\_SYS\_TRANS\_HALF\_TOPIC`的每个队列，做事务状态回查，一次最多不超过60S，目前该值不可配置。  
   
   代码@3：如果 `removeMap` 中包含当前处理的消息，则继续下一条，`removeMap` 中 的值是通过 `Step3` 中，填充的，具体实现逻辑是从`RMQ\_SYS\_TRANS\_OP\_HALF\_TOPIC`主题中拉取32条，如果拉取的消息队列偏移量大于等于`RMQ\_SYS\_TRANS\_HALF\_TOPIC\#queueId` 当前的处理进度时，会添加到 `removeMap` 中，表示已处理过。  
   
   代码@4：根据消息队列偏移量i从消费队列中获取消息。  
   
   代码@5：如果消息为空，则根据允许重复次数进行操作，默认重试一次，目前不可配置。其具体实现为：  
      1、如果超过重试次数，直接跳出，结束该消息队列的事务状态回查。  
      2、如果是由于没有新的消息而返回为空（拉取状态为：PullStatus.NO\_NEW\_MSG），则结束该消息队列的事务状态回查。  
      3、其他原因，则将偏移量i设置为： `getResult.getPullResult().getNextBeginOffset()`，重新拉取。  
   代码@6：判断该消息是否需要 `discard`(吞没，丢弃，不处理)、或`skip`(跳过)，其依据如下：  
   
      1、`needDiscard` 依据：如果该消息回查的次数超过允许的最大回查次数，则该消息将被丢弃，即事务消息提交失败，不能被消费者消费，其做法，主要是每回查一次，在消息属性`TRANSACTION\_CHECK\_TIMES`中增1，默认最大回查次数为5次。  
	  
      2、needSkip依据：如果事务消息超过文件的过期时间，默认72小时（具体请查看RocketMQ过期文件相关内容），则跳过该消息。  
   代码@7：处理事务超时相关概念，先解释几个局部变量：、  
      `valueOfCurrentMinusBorn` ：该消息已存储的时间,等于系统当前时间减去消息存储的时间戳。

      `checkImmunityTime` ：立即检测事务消息的时间，其设计的意义是，应用程序在发送事务消息后，事务不会马上提交，该时间就是假设事务消息发送成功后，应用程序事务提交的时间，在这段时间内，RocketMQ任务事务未提交，故不应该在这个时间段向应用程序发送回查请求。  

      `transactionTimeout`：事务消息的超时时间，这个时间是从OP拉取的消息的最后一条消息的存储时间与 `check` 方法开始的时间，如果时间差超过了 `transactionTimeout`，`就算时间小于checkImmunityTime` 时间，也发送事务回查指令。  
   代码@8：如果消息指定了事务消息过期时间属性（PROPERTY\_CHECK\_IMMUNITY\_TIME\_IN\_SECONDS），如果当前时间已超过该值。  
   代码@9：如果当前时间还未过（应用程序事务结束时间），则跳出本次回查处理的，等下一次再试。  
   
   代码@10：判断是否需要发送事务回查消息，具体逻辑：  
      1、如果从操作队列(`RMQ\_SYS\_TRANS\_OP\_HALF\_TOPIC`)中没有已处理消息并且已经超过（应用程序事务结束时间），参数 `transactionTimeOut` 值。  
      2、如果操作队列不为空，并且最后一天条消息的存储时间已经超过 `transactionTimeOut` 值。  
代码@11：如果需要发送事务状态回查消息，则先将消息再次发送到`RMQ\_SYS\_TRANS\_HALF\_TOPIC`主题中，发送成功则返回true，否则返回false，这里还有一个实现关键点：

```
if (putMessageResult != null
            && putMessageResult.getPutMessageStatus() == PutMessageStatus.PUT_OK) {
            msgExt.setQueueOffset(
                putMessageResult.getAppendMessageResult().getLogicsOffset());
            msgExt.setCommitLogOffset(
                putMessageResult.getAppendMessageResult().getWroteOffset());
            msgExt.setMsgId(putMessageResult.getAppendMessageResult().getMsgId());
}
```

   如果发送成功，会将该消息的 `queueOffset`、`commitLogOffset` 设置为重新存入的偏移量，为什么需要这样呢，答案在`listener.resolveHalfMsg(msgExt)`中。  
   `AbstractTransactionalMessageCheckListener\#resolveHalfMsg`

```
public void resolveHalfMsg(final MessageExt msgExt) {
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    sendCheckMessage(msgExt);
                } catch (Exception e) {
                    LOGGER.error("Send check message error!", e);
                }
            }
        });
    }
```

   发送具体的事务回查机制，这里用一个线程池来异步发送回查消息，为了回查进度保存的简化，这里只要发送了回查消息，当前回查进度会向前推动，如果回查失败，上一步骤新增的消息将可以再次发送回查消息，那如果回查消息发送成功，那会不会下一次又重复发送回查消息呢？
   
   这个可以根据OP队列中的消息来判断是否重复，如果回查消息发送成功并且消息服务器完成提交或回滚操作，这条消息会发送到 `OP` 队列中，然后首先会通过 `fillOpRemoveMap` 根据处理进度获取一批已处理的消息，来与消息判断是否重复，由于`fillopRemoveMap`一次只拉32条消息，那又如何保证一定能拉取到与当前消息的处理记录呢？其实就是通过代码@10，如果此批消息最后一条未超过事务延迟消息，则继续拉取更多消息进行判断（@12）和(@14),`op` 队列也会随着回查进度的推进而推进。  
   
   代码@12：如果无法判断是否发送回查消息，则加载更多的已处理消息进行刷选。  
   代码@13：保存(Prepare)消息队列的回查进度。  
   代码@14：保存处理队列（op）的进度。  
   上述讲解了 `TransactionalMessageCheckService` 回查定时线程的发送回查消息的整体流程与实现细节，接下来重点分析一下上述步骤@11,通过异步方式发送消息回查的实现过程。  
   `AbstractTransactionalMessageCheckListener\#sendCheckMessage`

```
public void sendCheckMessage(MessageExt msgExt) throws Exception {
        CheckTransactionStateRequestHeader checkTransactionStateRequestHeader = new CheckTransactionStateRequestHeader();   
        checkTransactionStateRequestHeader.setCommitLogOffset(msgExt.getCommitLogOffset());
        checkTransactionStateRequestHeader.setOffsetMsgId(msgExt.getMsgId());
        checkTransactionStateRequestHeader.setMsgId(msgExt.getUserProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX));
        checkTransactionStateRequestHeader.setTransactionId(checkTransactionStateRequestHeader.getMsgId());
        checkTransactionStateRequestHeader.setTranStateTableOffset(msgExt.getQueueOffset());    // @1
        msgExt.setTopic(msgExt.getUserProperty(MessageConst.PROPERTY_REAL_TOPIC));
        msgExt.setQueueId(Integer.parseInt(msgExt.getUserProperty(MessageConst.PROPERTY_REAL_QUEUE_ID)));
        msgExt.setStoreSize(0);      // @2
        String groupId = msgExt.getProperty(MessageConst.PROPERTY_PRODUCER_GROUP);     // @3
        Channel channel = brokerController.getProducerManager().getAvaliableChannel(groupId);
        if (channel != null) {
            brokerController.getBroker2Client().checkProducerTransactionState(groupId, channel, checkTransactionStateRequestHeader, msgExt);     // @4
        } else {
            LOGGER.warn("Check transaction failed, channel is null. groupId={}", groupId);
        }
    }
```

   代码@1：首先构建回查事务状态请求消息，请求核心参数包括：消息 `offsetId`、消息ID(索引)、消息事务ID、事务消息队列中的偏移量（`RMQ\_SYS\_TRANS\_HALF\_TOPIC`）。  
   代码@2：恢复原消息的主题、队列，并设置 `storeSize` 为0。  
   代码@3：获取生产者组名称。  
   代码@4：根据生产者组获取任意一个生产者，通过与其连接发送事务回查消息，回查消息的请求者为【`Broker` 服务器】，接收者为( `client`，具体为消息生产者)。  
   其处理类为：`org.apache.rocketmq.client.impl.ClientRemotingProcessor\#processRequest`，其详细逻辑实现方法为：  
   `ClientRemotingProcessor\#checkTransactionState`

```
public RemotingCommand checkTransactionState(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
        final CheckTransactionStateRequestHeader requestHeader =
            (CheckTransactionStateRequestHeader) request.decodeCommandCustomHeader(CheckTransactionStateRequestHeader.class);
        final ByteBuffer byteBuffer = ByteBuffer.wrap(request.getBody());
        final MessageExt messageExt = MessageDecoder.decode(byteBuffer);
        if (messageExt != null) {
            String transactionId = messageExt.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
            if (null != transactionId && !"".equals(transactionId)) {
                messageExt.setTransactionId(transactionId);
            }
            final String group = messageExt.getProperty(MessageConst.PROPERTY_PRODUCER_GROUP);
            if (group != null) {
                MQProducerInner producer = this.mqClientFactory.selectProducer(group);
                if (producer != null) {
                    final String addr = RemotingHelper.parseChannelRemoteAddr(ctx.channel());
                    producer.checkTransactionState(addr, messageExt, requestHeader);     // @1
                } else {
                    log.debug("checkTransactionState, pick producer by group[{}] failed", group);
                }
            } else {
                log.warn("checkTransactionState, pick producer group failed");
            }
        } else {
            log.warn("checkTransactionState, decode message failed");
        }

        return null;
    }
```

   代码@1：最终调用生产者的`checkTransactionState`方法。  
   `DefaultMQProducerImpl\#checkTransactionState`

```
public void checkTransactionState(final String addr, final MessageExt msg,
        final CheckTransactionStateRequestHeader header) {
        Runnable request = new Runnable() {        // @1
            private final String brokerAddr = addr;
            private final MessageExt message = msg;
            private final CheckTransactionStateRequestHeader checkRequestHeader = header;
            private final String group = DefaultMQProducerImpl.this.defaultMQProducer.getProducerGroup();

            @Override
            public void run() {
                TransactionListener transactionCheckListener = DefaultMQProducerImpl.this.checkListener();      // @1
                if (transactionCheckListener != null) {
                    LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
                    Throwable exception = null;
                    try {
                        localTransactionState = transactionCheckListener.checkLocalTransaction(message);            // @2
                    } catch (Throwable e) {
                        log.error("Broker call checkTransactionState, but checkLocalTransactionState exception", e);
                        exception = e;
                    }

                    this.processTransactionState(                                                                                                       // @3
                        localTransactionState,
                        group,
                        exception);
                } else {
                    log.warn("checkTransactionState, pick transactionCheckListener by group[{}] failed", group);
                }
            }

            private void processTransactionState(
                final LocalTransactionState localTransactionState,
                final String producerGroup,
                final Throwable exception) {
                final EndTransactionRequestHeader thisHeader = new EndTransactionRequestHeader();
                thisHeader.setCommitLogOffset(checkRequestHeader.getCommitLogOffset());
                thisHeader.setProducerGroup(producerGroup);
                thisHeader.setTranStateTableOffset(checkRequestHeader.getTranStateTableOffset());
                thisHeader.setFromTransactionCheck(true);

                String uniqueKey = message.getProperties().get(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                if (uniqueKey == null) {
                    uniqueKey = message.getMsgId();
                }
                thisHeader.setMsgId(uniqueKey);
                thisHeader.setTransactionId(checkRequestHeader.getTransactionId());
                switch (localTransactionState) {
                    case COMMIT_MESSAGE:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
                        break;
                    case ROLLBACK_MESSAGE:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
                        log.warn("when broker check, client rollback this transaction, {}", thisHeader);
                        break;
                    case UNKNOW:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
                        log.warn("when broker check, client does not know this transaction state, {}", thisHeader);
                        break;
                    default:
                        break;
                }

                String remark = null;
                if (exception != null) {
                    remark = "checkLocalTransactionState Exception: " + RemotingHelper.exceptionSimpleDesc(exception);
                }

                try {
                    DefaultMQProducerImpl.this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, thisHeader, remark,
                        3000);
                } catch (Exception e) {
                    log.error("endTransactionOneway exception", e);
                }
            }
        };

        this.checkExecutor.submit(request);    
    }
```

   上述代码虽多，其实实现思路非常清晰，先使用一个匿名类（ Runnable ）构建一个运行任务，然后提交到checkExecutor线程池中执行，这与我第一篇文章的猜测是吻合的，那重点分析一下该任务的允许逻辑，对应在run方法中。  
   
   代码@1：获取消息发送者的`TransactionListener`。  
   
   代码@2：执行`TransactionListener\#checkLocalTransaction`，检测本地事务状态，也就是应用程序需要实现TransactionListener\#checkLocalTransaction，告知RocketMQ该事务的事务状态，然后返回COMMIT\_MESSAGE、ROLLBACK\_MESSAGE、UNKNOW中的一个，然后向Broker发送END\_TRANSACTION命令即可，  
   
代码@3：发送END\_TRANSACTION到Broker，其具体实现，已经在[RocketMQ源码分析之RocketMQ事务消息实现原理上篇][RocketMQ_RocketMQ 1]中详细讲解过，再次不重复。  
   到这里，事务消息状态回查流程就讲解完毕，接下来以一张流程图介绍本篇的讲解。  
   
![img\_0914\_01\_1.png][img_0914_01_1.png]  
   下一篇，将重点分析Broker在收到事务状态为`COMMIT\_MESSAGE、ROLLBACK\_MESSAGE`时如果提交、回滚事务。

[RocketMQ_RocketMQ]: https://blog.csdn.net/prestigeding/article/details/81259646
[RocketMQ_RocketMQ 1]: https://blog.csdn.net/prestigeding/article/details/81263833
[RocketMQ_RocketMQ_----]: https://blog.csdn.net/prestigeding/article/details/81275892
[RocketMQ_-_Broker]: https://blog.csdn.net/prestigeding/article/details/81277067
[RocketMQ]: https://blog.csdn.net/prestigeding/article/details/81318980https://blog.csdn.net/prestigeding/article/details/81318980
[img_0914_01_1.png]: https://gitee.com/duchaochen/gongzhonghao/raw/master/个人博客文章/001-images/souyunku-web/2019/09/0914/01/22/img_0914_01_1.png
[img_0914_01_2.png]: https://gitee.com/duchaochen/gongzhonghao/raw/master/个人博客文章/001-images/souyunku-web/2019/09/0914/01/22/img_0914_01_2.png
[img_0914_01_3.png]: https://gitee.com/duchaochen/gongzhonghao/raw/master/个人博客文章/001-images/souyunku-web/2019/09/0914/01/22/img_0914_01_3.png

写完了如果写得有什么问题，希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")