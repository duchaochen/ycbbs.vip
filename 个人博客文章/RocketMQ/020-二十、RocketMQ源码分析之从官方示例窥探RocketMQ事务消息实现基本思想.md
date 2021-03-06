>作者：唯有坚持不懈 
出处：[https://blog.csdn.net/prestigeding/article/details/78888290](https://blog.csdn.net/prestigeding/article/details/78888290)
------------

RocketMQ事务消息阅读目录指引：  
[RocketMQ源码分析之从官方示例窥探RocketMQ事务消息实现基本思想][RocketMQ_RocketMQ]  
[RocketMQ源码分析之RocketMQ事务消息实现原理上篇][RocketMQ_RocketMQ 1]  
[RocketMQ源码分析之RocketMQ事务消息实现原理中篇----事务消息状态回查][RocketMQ_RocketMQ_----]  
[RocketMQ源码分析之事务消息实现原理下篇-消息服务器Broker提交回滚事务实现原理][RocketMQ_-_Broker]  
[RocketMQ事务消息实战][RocketMQ]


  ` RocketMQ4.3.0`版本开始支持事务消息，本节开始将剖析事务消息的实现原理，首先将从官方给出的 `Demo` 实例入手，以此通往 `RocketMQ` 事务消息的世界中。  
   
   官方版本未发布之前，从 `apache` `rocketmq` 第一个版本上线后，代码中存在者与事务消息相关的代码，例如 `COMMIT`、`ROLLBACK`、`PREPARED`， 网上对于事务消息的“声音”基本上是使用类似二阶段提交，消息系统标志MessageSysFlag中定义的：`TRANSACTION\_PREPARED\_TYPE、TRANSACTION\_COMMIT\_TYPE、  `
`TRANSACTION\_ROLLBACK\_TYPE`，消息发送者首先发送TRANSACTION\_PREPARED\_TYPE类型的消息，然后事务介绍后，发送commit请求或rollback请求，如果 `commit`,`rollback` 消息丢失的话，`rocketmq` 会在一定超时时间后会查，应用程序需要告知该消息是提交还是回滚。让我们各自带着自己的理解和猜出，先重点看一下Demo程式，大概可以窥探一些大体的信息。

   Demo实例程序位于：/rocketmq-example/src/main/java/org/apache/rocketmq/example/transaction包中。从而先运行生产者，然后运行消费者，判断事务消息的预发放、提交、回滚等效果，二话不说，先运行一下，看下效果再说：  
   消息发送端运行结果：

```
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D5767EC0000, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=1], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D57680F0001, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=2], queueOffset=1]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D57681E0002, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=3], queueOffset=2]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D57682B0003, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=0], queueOffset=3]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D5768380004, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=1], queueOffset=4]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D5768490005, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=2], queueOffset=5]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D5768560006, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=3], queueOffset=6]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D5768640007, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=0], queueOffset=7]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D5768730008, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=1], queueOffset=8]
SendResult [sendStatus=SEND_OK, msgId=C0A8010518DC6D06D69C8D5768800009, offsetMsgId=null, messageQueue=MessageQueue [topic=transaction_topic_test, brokerName=broker-a, queueId=2], queueOffset=9]
```

   综上所述，服务端发送了10条消息，但我们从 `rocketmq-consonse` 上只能查看到3条消息，一个合理的解释就是只有3条消息提交，其他都回滚了，如图所示：  
![img\_0914\_01\_1.png][img_0914_01_1.png]  
   实例代码分析：

```
public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransactionListenerImpl();        // @1
        TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
        producer.setNamesrvAddr("127.0.0.1:9876");
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });      // @2
        producer.setExecutorService(executorService);                                // @3
        producer.setTransactionListener(transactionListener);                      // @4
        producer.start();
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {                                                                    // @5
            try {
                Message msg =
                    new Message("transaction_topic_test", tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }
        for (int i = 0; i < 100000; i++) {     //这里只是阻止生产者过早退出，导致事务消息的相关机制无法运行
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
```

   代码@1：创建 `TransactionListener` 实例，字面理解为事务消息事件监听器，下文详细对其进行展开。  
   代码@2：`ExecutorService` `executorService`，创建一个线程池，其线程的名称前缀”`client-transaction-msg-check-thread`“，从字面理解为客户端事务消息状态检测线程，我们可以大胆的猜测一下是不是这个线程池调用 `TransactionListener` 方法，完成对事务消息的检测呢？【这里只是作者的猜测，大家不能当真，在作者后续文章发布后，如果该观点错误，会加以修复，这里写出来，主要是想分享一下我读源码的方法】。  
   代码@3：为事务消息发送者设置线程池。  
   代码@4：为事务消息发送者设置事务监听器。  
   代码@5：发送10条消息。  
   接着，我们再来看一下 `TransactionListener` 的实现：  
   `TransactionListenerImpl`

```
public class TransactionListenerImpl implements TransactionListener {
    private AtomicInteger transactionIndex = new AtomicInteger(0);

    private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();

    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        int value = transactionIndex.getAndIncrement();
        int status = value % 3;
        localTrans.put(msg.getTransactionId(), status);
        return LocalTransactionState.UNKNOW;
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        Integer status = localTrans.get(msg.getTransactionId());
        if (null != status) {
            switch (status) {
                case 0:
                    return LocalTransactionState.UNKNOW;
                case 1:
                    return LocalTransactionState.COMMIT_MESSAGE;
                case 2:
                    return LocalTransactionState.ROLLBACK_MESSAGE;
            }
        }
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```

   `executeLocalTransaction`:方法，记录本地事务的事务状态，这里其实现就是循环设置事务消息的状态为`0,1,2`，`demo` 中是把消息的状态数据存放在一个 `Map` 中，实际应用，应该会持久化消息的事务状态，例如数据库或缓存。  
   
其关键是 `checkLocalTransaction`，会查本地事务表，判断事务的状态如为0：`UNKNOW`，1：`COMMIT\_MESSAGE；ROLLBACK\_MESSAGE`。这里就能解释，生产者连续发10条消息，因为只有3条消息的事务状态为`COMMIT\_MESSAG`E，故消息消费者只能消费3条。  

   到这里，基本还是可以得知事务消息的实现方式，基本与文章开头所示的“网上声音”实现类似，下一节将详细分析 `TransactionMQProducer` 事务消息发送的实现细节。  
在这里也可以提出一个疑问，如果事务消息的有 `PREPARED`、`COMMITED`、`ROLLBACK` 状态，`RocketMQ` 如何快速找到 `PREPARED` 的消息，并去回查消息。

   郑重声明：本文主要是展示事务消息的基本使用，本文所下的结论还仅仅是作者的猜测，下一篇文章，才会重点分析事务消息的实现细节，本文一个重要的目的，是向读者朋友们展示作者学习源码的一个方法，总结为：对事情先做大概全面了解（网上，官方文档）、然后加以自己的思考，从Demo实例入手学习，将学习任务分解之（最大的原因是时间的碎片利用），边写边看，最终形成一篇一篇的文章，最后多多复习自己缩写，望对大家有帮助。  
   这算不算文末有彩蛋呢？呵呵，下一篇见：详细分析 `RocketMQ` 事务消息的实现细节。



[RocketMQ_RocketMQ]: https://blog.csdn.net/prestigeding/article/details/81259646
[RocketMQ_RocketMQ 1]: https://blog.csdn.net/prestigeding/article/details/81263833
[RocketMQ_RocketMQ_----]: https://blog.csdn.net/prestigeding/article/details/81275892
[RocketMQ_-_Broker]: https://blog.csdn.net/prestigeding/article/details/81277067
[RocketMQ]: https://blog.csdn.net/prestigeding/article/details/81318980https://blog.csdn.net/prestigeding/article/details/81318980
[img_0914_01_1.png]: https://gitee.com/duchaochen/gongzhonghao/raw/master/个人博客文章/001-images/souyunku-web/2019/09/0914/01/20/img_0914_01_1.png



写完了如果写得有什么问题，希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")