作者：唯有坚持不懈 | 出处：[https://blog.csdn.net/prestigeding/article/details/78888290](https://blog.csdn.net/prestigeding/article/details/78888290)


RocketMQ事务消息阅读目录指引：  
[RocketMQ源码分析之从官方示例窥探RocketMQ事务消息实现基本思想][RocketMQ_RocketMQ]  
[RocketMQ源码分析之RocketMQ事务消息实现原理上篇][RocketMQ_RocketMQ 1]  
[RocketMQ源码分析之RocketMQ事务消息实现原理中篇----事务消息状态回查][RocketMQ_RocketMQ_----]  
[RocketMQ源码分析之事务消息实现原理下篇-消息服务器Broker提交回滚事务实现原理][RocketMQ_-_Broker]  
[RocketMQ事务消息实战][RocketMQ]

   根据上节Demo示例，发送事务消息的入口为：`TransactionMQProducer\#sendMessageInTransaction`：

```
public TransactionSendResult sendMessageInTransaction(final Message msg, final Object arg) throws MQClientException {
        if (null == this.transactionListener) {    // @1
            throw new MQClientException("TransactionListener is null", null);
        }

        return this.defaultMQProducerImpl.sendMessageInTransaction(msg, transactionListener, arg);  // @2
    }
```

   代码@1：如果`transactionListener`为空，则直接抛出异常。  
   代码@2：调用d`efaultMQProducerImpl的sendMessageInTransaction`方法。  
  ` DefaultMQProducerImpl\#sendMessageInTransaction`

```
public TransactionSendResult sendMessageInTransaction(final Message msg,
           final TransactionListener tranExecuter, final Object arg)  throws MQClientException {
```

   Step1：首先先阐述一下参数含义。`final` `Message` `msg`：消息；`TransactionListener` `tranExecuter`：事务监听器；` Object arg`：其他附加参数，该参数会再 `TransactionListener` 回调函数中原值传入。  
   DefaultMQProducerImpl\#sendMessageInTransaction

```
SendResult sendResult = null;
MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
try {
       sendResult = this.send(msg);
} catch (Exception e) {
       throw new MQClientException("send message Exception", e);
}
```

   Step2：在消息属性中，添加两个属性：`TRAN\_MSG`，其值为 `true`，表示为事务消息；`PGROUP`：消息所属发送者组，然后以同步方式发送消息。  
   在消息发送之前，会先检查消息的属性`TRAN\_MSG`,如果存在并且值为 `true`，则通过设置消息系统标记的方式，设置消息为`MessageSysFlag.TRANSACTION\_PREPARED\_TYPE`。  
   `DefaultMQProducerImpl\#sendKernelImpl`

```
final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
       sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
}

SendMessageProcessor#sendMessage
String traFlag = oriProps.get(MessageConst.PROPERTY_TRANSACTION_PREPARED);
if (traFlag != null && Boolean.parseBoolean(traFlag)) {
        if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
             response.setCode(ResponseCode.NO_PERMISSION);
             response.setRemark(
                    "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
                        + "] sending transaction message is forbidden");
             return response;
       }
      putMessageResult = this.brokerController.getTransactionalMessageService().prepareMessage(msgInner);
} else {
      putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
}
```

   Step3：`Broker` 端首先客户发送消息请求后，判断消息类型，如果是事务消息，则调用`TransactionalMessageService\#prepareMessage`方法，否则走原先的逻辑，调用`MessageStore\#putMessage`方法。

```
org.apache.rocketmq.broker.transaction.queue.TransactionalMessageServiceImpl#prepareMessage
public PutMessageResult prepareMessage(MessageExtBrokerInner messageInner) {
        return transactionalMessageBridge.putHalfMessage(messageInner);
 }
```

   step4：事务消息，将调用`TransactionalMessageServiceImpl\#prepareMessage`方法，继而调用`TransactionalMessageBridge\#prepareMessag`e方法。

```
TransactionalMessageBridge#parseHalfMessageInner
public PutMessageResult putHalfMessage(MessageExtBrokerInner messageInner) {
        return store.putMessage(parseHalfMessageInner(messageInner));
    }

    private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {
        MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
        MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID,
            String.valueOf(msgInner.getQueueId()));
        msgInner.setSysFlag(
            MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));
        msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
        msgInner.setQueueId(0);
        msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
        return msgInner;
    }
```

   Step5：备份消息的原主题名称与原队列ID，然后取消是事务消息的消息标签，重新设置消息的主题为：`RMQ\_SYS\_TRANS\_HALF\_TOPIC`，队列ID固定为0。
   
   然后调用`MessageStore\#putMessage`方法将消息持久化，这里`TransactionalMessageBridge`桥接类，就是封装事务消息的相关流程，最终调用 `MessageStore` 完成消息的持久化。
   
   消息入库后，会继续回到`DefaultMQProducerImpl\#sendMessageInTransaction`，上文的Step2后面，也就是通过同步将消息发送到消息服务端。  `DefaultMQProducerImpl\#sendMessageInTransaction`

```
switch (sendResult.getSendStatus()) {
            case SEND_OK: {
                try {
                    if (sendResult.getTransactionId() != null) {
                        msg.putUserProperty("__transactionId__", sendResult.getTransactionId());
                    }
                    String transactionId = msg.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                    if (null != transactionId && !"".equals(transactionId)) {
                        msg.setTransactionId(transactionId);
                    }
                    localTransactionState = tranExecuter.executeLocalTransaction(msg, arg);
                    if (null == localTransactionState) {
                        localTransactionState = LocalTransactionState.UNKNOW;
                    }

                    if (localTransactionState != LocalTransactionState.COMMIT_MESSAGE) {
                        log.info("executeLocalTransactionBranch return {}", localTransactionState);
                        log.info(msg.toString());
                    }
                } catch (Throwable e) {
                    log.info("executeLocalTransactionBranch exception", e);
                    log.info(msg.toString());
                    localException = e;
                }
            }
            break;
            case FLUSH_DISK_TIMEOUT:
            case FLUSH_SLAVE_TIMEOUT:
            case SLAVE_NOT_AVAILABLE:
                localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
                break;
            default:
                break;
        }
```

   Step6：如果消息发送成功，会回调`TransactionListener\#executeLocalTransaction`方法，执行本地事务，并且返回本地事务状态为：`public enum LocalTransactionState` \`{COMMIT\_MESSAGE,ROLLBACK\_MESSAGE,  UNKNOW,\}` 之一
   
 >注意：`TransactionListener\#executeLocalTransaction`是在发送者成功发送`PREPARED`消息后，会执行本地事务方法，然后返回本地事务状态；如果`PREPARED`消息发送失败，则不会调用  
`TransactionListener\#executeLocalTransaction`，并且本地事务消息，设置为  `LocalTransactionState.ROLLBACK\_MESSAGE`，表示消息需要被回`DefaultMQProducerImpl\#sendMessageInTransaction`

```
try {
this.endTransaction(sendResult, localTransactionState, localException);
} catch (Exception e) {
log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);
}
```

   step7：调用 `endTransaction` 方法结束事务（提交或回滚）。  
   `DefaultMQProducerImpl\#endTransaction`

```
EndTransactionRequestHeader requestHeader = new EndTransactionRequestHeader();
requestHeader.setTransactionId(transactionId);
requestHeader.setCommitLogOffset(id.getOffset());
switch (localTransactionState) {
    case COMMIT_MESSAGE:
         requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
         break;
    case ROLLBACK_MESSAGE:
         requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
         break;
     case UNKNOW:
         requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
         break;
     default:
         break;
}
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
requestHeader.setTranStateTableOffset(sendResult.getQueueOffset());
requestHeader.setMsgId(sendResult.getMsgId());
```

   step8：组装结束事务请求，主要参数为：事务ID、事务操作（`commitOrRollback`)、消费组、消息队列偏移量、消息ID，`fromTransactionCheck`，从这里发出的请求，默认为 `false`。`Broker` 端的请求处理器为：`EndTransactionProcessor`。  
   step9：`EndTransactionProcessor` 根据事务提交类型：`TRANSACTION\_COMMIT\_TYPE`（提交事务）、`TRANSACTION\_ROLLBACK\_TYPE`（回滚事务）、`TRANSACTION\_NOT\_TYPE`、忽略该请求，会记录info级别的日志相关的代码将在下文详细分析，在这里，我们先大概梳理一条消息发送的路径TransactionMQProducer\#sendMessageInTransaction的调用链来总结一下事务消息的发送流程。 
   
![img\_0914\_01\_1.png][img_0914_01_1.png]  
   本文到这里，初步展示了事务消息的发送流程，总的说来，`RocketMQ` 的事务消息发送使用二阶段提交思路，首先，在消息发送时，先发送消息类型为 `Prepread` 类型的消息，然后在将该消息成功存入到消息服务器后，会回调   `TransactionListener\#executeLocalTransaction`，执行本地事务状态回调函数，然后根据该方法的返回值，结束事务：  
   1、`COMMIT\_MESSAGE` ：提交事务。  
   2、`ROLLBACK\_MESSAGE`：回滚事务。  
   3、UNKNOW：未知事务状态，此时消息服务器(`Broker`)收到`EndTransaction`命令时，将不对这种消息做处理，消息还处于`Prepared`类型，存储在主题为：`RMQ\_SYS\_TRANS\_HALF\_TOPIC`的队列中，然后消息发送流程将结束，那这些消息如何提交或回滚呢？为了实现避免客户端需要再次发送提交、回滚命令，`RocketMQ` 会采取定时任务将`RMQ\_SYS\_TRANS\_HALF\_TOPIC`中的消息取出，然后回到客户端，判断该消息是否需要提交或回滚，来完成事务消息的声明周期，该部分内容将在下节重点探讨。

   事务消息后续文章预告：  
   1、事务消息状态会查机制实现。  
   2、消息服务端在收到客户端的回滚、提交命令时，如果高效处理事务消息的提交、回滚动作。  
   3、事务消息实战。


[RocketMQ_RocketMQ]: https://blog.csdn.net/prestigeding/article/details/81259646
[RocketMQ_RocketMQ 1]: https://blog.csdn.net/prestigeding/article/details/81263833
[RocketMQ_RocketMQ_----]: https://blog.csdn.net/prestigeding/article/details/81275892
[RocketMQ_-_Broker]: https://blog.csdn.net/prestigeding/article/details/81277067
[RocketMQ]: https://blog.csdn.net/prestigeding/article/details/81318980https://blog.csdn.net/prestigeding/article/details/81318980
[img_0914_01_1.png]: https://gitee.com/duchaochen/gongzhonghao/raw/master/个人博客文章/001-images/souyunku-web/2019/09/0914/01/21/img_0914_01_1.png
[img_0914_01_2.png]: https://gitee.com/duchaochen/gongzhonghao/raw/master/个人博客文章/001-images/souyunku-web/2019/09/0914/01/21/img_0914_01_2.png
[img_0914_01_3.png]: https://gitee.com/duchaochen/gongzhonghao/raw/master/个人博客文章/001-images/souyunku-web/2019/09/0914/01/21/img_0914_01_3.png


写完了如果写得有什么问题，希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")