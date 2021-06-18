#### 消息存储


```
class ConsumeQueue{

    MappedFileQueue mappedFileQueue;
    String topic;
    int queueId;
    
    public ConsumeQueue{
        String queueDir = this.storePath
            + File.separator + topic
            + File.separator + queueId;

        //不同队列不同文件
        this.mappedFileQueue = new MappedFileQueue(queueDir, mappedFileSize, null);
    }
}

class MappedFileQueue{
    
    //文件路径
    String storePath;
    //文件大小
    int mappedFileSize;
    
    long flushedWhere = 0;
    long committedWhere = 0;
    
}

public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    ...
    MappedFile unlockMappedFile = null;
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

    //自旋锁或者可重入锁
    putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
    //写入消息实体
    result = mappedFile.appendMessage(msg, this.appendMessageCallback);
    switch (result.getStatus()) {
        case PUT_OK:
            break;
        case END_OF_FILE:
            //文件结束，新建文件重写
            unlockMappedFile = mappedFile;
            // Create a new file, re-write the message
            mappedFile = this.mappedFileQueue.getLastMappedFile(0);
            if (null == mappedFile) {
                // XXX: warn and notify me
                log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
            }
            result = mappedFile.appendMessage(msg, this.appendMessageCallback);
            break;
        case MESSAGE_SIZE_EXCEEDED:
        case PROPERTIES_SIZE_EXCEEDED:
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
        case UNKNOWN_ERROR:
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        default:
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
    }
    ...
    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

    // Statistics
    //消息统计
    storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
    storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());

    //刷盘处理
    handleDiskFlush(result, putMessageResult, msg);
    handleHA(result, putMessageResult, msg);

    return putMessageResult;
}

public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
            final MessageExtBrokerInner msgInner) {
    // Record ConsumeQueue information 
    keyBuilder.setLength(0);
    keyBuilder.append(msgInner.getTopic());
    keyBuilder.append('-');
    keyBuilder.append(msgInner.getQueueId());
    String key = keyBuilder.toString();
    //根据topic和queueId获取offset,不存在则为0
    Long queueOffset = CommitLog.this.topicQueueTable.get(key);
    if (null == queueOffset) {
        queueOffset = 0L;
        CommitLog.this.topicQueueTable.put(key, queueOffset);
    }       
    
    //事务消息处理
     // Transaction messages that require special handling
    final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
    switch (tranType) {
        //准备消息和回滚消息不会被消费，不会进去消息队列
        // Prepared and Rollback message is not consumed, will not enter the
        // consumer queuec
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
            queueOffset = 0L;
            break;
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
        default:
            break;
    }
    //序列化消息内容
    
    ...
    AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
        msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

    //事务消息处理，prepared和rollback的消息不进入consumeQueue
    switch (tranType) {
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
            break;
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
            // The next update ConsumeQueue information
            CommitLog.this.topicQueueTable.put(key, ++queueOffset);
            break;
        default:
            break;
    }
    return result;
}


//刷盘
 public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    // Synchronization flush 同步刷盘
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
        if (messageExt.isWaitStoreMsgOK()) {
            GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
            service.putRequest(request);
            CompletableFuture<PutMessageStatus> flushOkFuture = request.future();
            PutMessageStatus flushStatus = null;
            try {
                flushStatus = flushOkFuture.get(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout(),
                        TimeUnit.MILLISECONDS);
            } catch (InterruptedException | ExecutionException | TimeoutException e) {
                //flushOK=false;
            }
            if (flushStatus != PutMessageStatus.PUT_OK) {
                log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                    + " client address: " + messageExt.getBornHostString());
                putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
            }
        } else {
            service.wakeup();
        }
    }
    // Asynchronous flush 异步刷盘
    else {
        if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            flushCommitLogService.wakeup();
        } else {
            commitLogService.wakeup();
        }
    }
}


//高可用。即集群同步
public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
        HAService service = this.defaultMessageStore.getHaService();
        if (messageExt.isWaitStoreMsgOK()) {
            // Determine whether to wait
            if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                service.putRequest(request);
                service.getWaitNotifyObject().wakeupAll();
                PutMessageStatus replicaStatus = null;
                try {
                    replicaStatus = request.future().get(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout(),
                            TimeUnit.MILLISECONDS);
                } catch (InterruptedException | ExecutionException | TimeoutException e) {
                }
                if (replicaStatus != PutMessageStatus.PUT_OK) {
                    log.error("do sync transfer other node, wait return, but failed, topic: " + messageExt.getTopic() + " tags: "
                        + messageExt.getTags() + " client address: " + messageExt.getBornHostNameString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
                }
            }
            // Slave problem
            else {
                // Tell the producer, slave not available
                putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
            }
        }
    }

}
```



#### 延迟消息的实现
RocketMQ支持固定级别的延迟消息，默认只是18个level的延迟级别
```
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

##### Broker处理延迟消息流程
1. commitLog
2. comsumeQueue->schdule-topic-xxx
3. SchduleMessageService
4. consumeQueue->目标topic

总共6个步骤
1. 修改消息Topic名称和队列信息
2. 转发消息到延迟主题的ComsumeQueue中
3. 延迟服务消费SCHEDULE_TOPIC_XXXX消息
4. 将信息重新存储到CommitLog中
5. 将消息投递到目标Topic中
6. 消费者消费目标topic中的数据

###### 1. 修改Topic名称和队列信息

```
if (msg.getDelayTimeLevel() > 0) {
    if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
        msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
    }

    //修改为Schdule_topic_xxx
    topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
    //根据延迟级别设置队列信息
    queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

    // Backup real topic, queueId 备份实际topic和队列信息
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
    msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

    //重新设置topic和队列信息
    msg.setTopic(topic);
    msg.setQueueId(queueId);
}
```

###### 2. 消息存储

```
long elapsedTimeInLock = 0;

MappedFile unlockMappedFile = null;
MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
try {
    long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
    this.beginTimeInLock = beginLockTimestamp;

    // Here settings are stored timestamp, in order to ensure an orderly
    // global
    msg.setStoreTimestamp(beginLockTimestamp);
    
    if (null == mappedFile || mappedFile.isFull()) {
        mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
    }
    if (null == mappedFile) {
        log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
        beginTimeInLock = 0;
        return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
    }
    
    result = mappedFile.appendMessage(msg, this.appendMessageCallback);
    switch (result.getStatus()) {
        case PUT_OK:
            break;
        case END_OF_FILE:
            //创建新mappedfile
            unlockMappedFile = mappedFile;
            // Create a new file, re-write the message
            mappedFile = this.mappedFileQueue.getLastMappedFile(0);
            if (null == mappedFile) {
                // XXX: warn and notify me
                log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
            }
            result = mappedFile.appendMessage(msg, this.appendMessageCallback);
            break;
        case MESSAGE_SIZE_EXCEEDED:
        case PROPERTIES_SIZE_EXCEEDED:
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
        case UNKNOWN_ERROR:
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        default:
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
}

elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
beginTimeInLock = 0;
```

###### 3. 消息投递

```
if (propertiesLength > 0) {
    byteBuffer.get(bytesContent, 0, propertiesLength);
    String properties = new String(bytesContent, 0, propertiesLength, MessageDecoder.CHARSET_UTF8);
    //将字符串存储的msg解析成属性map
    propertiesMap = MessageDecoder.string2messageProperties(properties);

    keys = propertiesMap.get(MessageConst.PROPERTY_KEYS);

    uniqKey = propertiesMap.get(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);

    String tags = propertiesMap.get(MessageConst.PROPERTY_TAGS);
    if (tags != null && tags.length() > 0) {
        tagsCode = MessageExtBrokerInner.tagsString2tagsCode(MessageExt.parseTopicFilterType(sysFlag), tags);
    }

    // 延迟消息处理
    // Timing message processing
    {
        //获取消息的延迟级别
        String t = propertiesMap.get(MessageConst.PROPERTY_DELAY_TIME_LEVEL);
        //消息topic为schdule-topic-XXXX 且延迟级别不为空
        if (TopicValidator.RMQ_SYS_SCHEDULE_TOPIC.equals(topic) && t != null) {
            int delayLevel = Integer.parseInt(t);

            if (delayLevel > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                delayLevel = this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel();
            }

            if (delayLevel > 0) {
                //计算投递时间 = 存储时间+延迟时间
                tagsCode = this.defaultMessageStore.getScheduleMessageService().computeDeliverTimestamp(delayLevel,
                    storeTimestamp);
            }
        }
    }
}

 public long computeDeliverTimestamp(final int delayLevel, final long storeTimestamp) {
        Long time = this.delayLevelTable.get(delayLevel);
        if (time != null) {
            return time + storeTimestamp;
        }

        return storeTimestamp + 1000;
    }
```

###### 4. broker定时任务处理

FIRST_DELAY_TIME 1000
DELAY_FOR_A_WHILE 100

```
 public void start() {
    if (started.compareAndSet(false, true)) {
        this.timer = new Timer("ScheduleMessageTimerThread", true);
        //对每一个延时级别创建一个TimeTask
        //delayLevelTable 存储的是延迟级别对应的延迟时间
        for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
            Integer level = entry.getKey();
            Long timeDelay = entry.getValue();
            //offsetTable 存储的是延迟级别对应的offset,offset为空则从头开始
            Long offset = this.offsetTable.get(level);
            if (null == offset) {
                offset = 0L;
            }

            if (timeDelay != null) {
                this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
            }
        }

        this.timer.scheduleAtFixedRate(new TimerTask() {

            @Override
            public void run() {
                try {
                    if (started.get()) ScheduleMessageService.this.persist();
                } catch (Throwable e) {
                    log.error("scheduleAtFixedRate flush exception", e);
                }
            }
        }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
    }
}


DeliverDelayedMessageTimerTask.run()
@Override
public void run() {
    try {
        if (isStarted()) {
            this.executeOnTimeup();
        }
    } catch (Exception e) {
        // XXX: warn and notify me
        log.error("ScheduleMessageService, executeOnTimeup exception", e);
        ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(
            this.delayLevel, this.offset), DELAY_FOR_A_PERIOD);
    }
}


public void executeOnTimeup() {
    //根据延时级别获取对应queue信息
    ConsumeQueue cq =
        ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(TopicValidator.RMQ_SYS_SCHEDULE_TOPIC,
            delayLevel2QueueId(delayLevel));

    long failScheduleOffset = offset;

    if (cq != null) {
        SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
        if (bufferCQ != null) {
            try {
                long nextOffset = offset;
                int i = 0;
                ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
                for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                    //记录消息在commitlog中的位置
                    long offsetPy = bufferCQ.getByteBuffer().getLong();
                    //记录消息的大小
                    int sizePy = bufferCQ.getByteBuffer().getInt();
                    //记录消息时间
                    long tagsCode = bufferCQ.getByteBuffer().getLong();

                    long now = System.currentTimeMillis();
                    long deliverTimestamp = this.correctDeliverTimestamp(now, tagsCode);

                    nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);

                    long countdown = deliverTimestamp - now;
                    //当前时间大于等于消息消费时间
                    if (countdown <= 0) {
                        //从commitlog获取消息并进行解析
                        MessageExt msgExt = ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(
                                offsetPy, sizePy);

                        if (msgExt != null) {
                            try {
                                //重新设置消息信息，包括topic和队列信息
                                MessageExtBrokerInner msgInner = this.messageTimeup(msgExt);
                                if (TopicValidator.RMQ_SYS_TRANS_HALF_TOPIC.equals(msgInner.getTopic())) {
                                    log.error("[BUG] the real topic of schedule msg is {}, discard the msg. msg={}",
                                            msgInner.getTopic(), msgInner);
                                    continue;
                                }
                                //将消息重新投递到实际的topic中
                                PutMessageResult putMessageResult =
                                    ScheduleMessageService.this.writeMessageStore
                                        .putMessage(msgInner);
                            } catch (Exception e) {
                                /*
                                 * XXX: warn and notify me
                                 */
                                log.error(
                                    "ScheduleMessageService, messageTimeup execute error, drop it. msgExt="
                                        + msgExt + ", nextOffset=" + nextOffset + ",offsetPy="
                                        + offsetPy + ",sizePy=" + sizePy, e);
                            }
                        }
                    } else {
                        ScheduleMessageService.this.timer.schedule(
                            new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset),
                            countdown);
                        ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                        return;
                    }
                } // end of for

                nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(
                    this.delayLevel, nextOffset), DELAY_FOR_A_WHILE);
                ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                return;
            } finally {

                bufferCQ.release();
            }
        } // end of if (bufferCQ != null)
        else {

            long cqMinOffset = cq.getMinOffsetInQueue();
            if (offset < cqMinOffset) {
                failScheduleOffset = cqMinOffset;
                log.error("schedule CQ offset invalid. offset=" + offset + ", cqMinOffset="
                    + cqMinOffset + ", queueId=" + cq.getQueueId());
            }
        }
    } // end of if (cq != null)

    ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel,
        failScheduleOffset), DELAY_FOR_A_WHILE);
}

// 消息时间处理
private long correctDeliverTimestamp(final long now, final long deliverTimestamp) {

    long result = deliverTimestamp;

    long maxTimestamp = now + ScheduleMessageService.this.delayLevelTable.get(this.delayLevel);
    if (deliverTimestamp > maxTimestamp) {
        result = now;
    }

    return result;
}

```

#### 事务消息



```
生产者

TransactionMQProducer
@Override
public TransactionSendResult sendMessageInTransaction(final Message msg,
    final Object arg) throws MQClientException {
    if (null == this.transactionListener) {
        throw new MQClientException("TransactionListener is null", null);
    }

    msg.setTopic(NamespaceUtil.wrapNamespace(this.getNamespace(), msg.getTopic()));
    return this.defaultMQProducerImpl.sendMessageInTransaction(msg, null, arg);
}

defaultMQProducerImpl
public TransactionSendResult sendMessageInTransaction(final Message msg,
        final LocalTransactionExecuter localTransactionExecuter, final Object arg)
        throws MQClientException {
    TransactionListener transactionListener = getCheckListener();
    if (null == localTransactionExecuter && null == transactionListener) {
        throw new MQClientException("tranExecutor is null", null);
    }

    // ignore DelayTimeLevel parameter * 事务消息忽略设置的延时级别，即事务消费不可是延迟消息
    if (msg.getDelayTimeLevel() != 0) {
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_DELAY_TIME_LEVEL);
    }

    Validators.checkMessage(msg, this.defaultMQProducer);

    SendResult sendResult = null;
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
    try {
        sendResult = this.send(msg);
    } catch (Exception e) {
        throw new MQClientException("send message Exception", e);
    }

    LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
    Throwable localException = null;
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
                if (null != localTransactionExecuter) {
                    //prepare消息发送成功，执行本地消息
                    localTransactionState = localTransactionExecuter.executeLocalTransactionBranch(msg, arg);
                } else if (transactionListener != null) {
                    log.debug("Used new transaction API");
                    localTransactionState = transactionListener.executeLocalTransaction(msg, arg);
                }
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

    try {
        //提交或者回滚事务消息
        this.endTransaction(sendResult, localTransactionState, localException);
    } catch (Exception e) {
        log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);
    }

    ...
}


public void endTransaction(
        final SendResult sendResult,
        final LocalTransactionState localTransactionState,
        final Throwable localException) throws RemotingException, MQBrokerException, InterruptedException, UnknownHostException {
        final MessageId id;
        if (sendResult.getOffsetMsgId() != null) {
            id = MessageDecoder.decodeMessageId(sendResult.getOffsetMsgId());
        } else {
            id = MessageDecoder.decodeMessageId(sendResult.getMsgId());
        }
        String transactionId = sendResult.getTransactionId();
        final String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(sendResult.getMessageQueue().getBrokerName());
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
        String remark = localException != null ? ("executeLocalTransactionBranch exception: " + localException.toString()) : null;
        this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, requestHeader, remark,
            this.defaultMQProducer.getSendMsgTimeout());
    }

DefaultMQProducerImpl的checkTransactionState方法，是本地应用对回查消息的处理    
    
```

MQ Broker
1. 将消息的topic,queueId放进消息体的map里进行保存
2. 设置消息的topic设置为RMQ_SYS_TRANS_HALF_TOPIC，queueId 设置为0
3. 将消息写入磁盘持久化

```
public PutMessageResult putHalfMessage(MessageExtBrokerInner messageInner) {
    return store.putMessage(parseHalfMessageInner(messageInner));
}

private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {
    //保存真实topic和queueId
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

public static String TransactionalMessageUtil.buildHalfTopic()() {
    //"RMQ_SYS_TRANS_HALF_TOPIC" half topic
    return TopicValidator.RMQ_SYS_TRANS_HALF_TOPIC;
}
```

遍历未提交或者未回滚的半消息并且向生产者请求获取事务的状态   
超过72小时的事务消息，就算过期
TransactionalMessageService.check()
putBackHalfMsgQueue() 将半消息从磁盘读取到内存后会对其属性进行修改，如TRANSACTION_CHECK_TIMES （事务回查次数）,所以在发送回查消息前对半消息进行再次刷盘，保证顺序写入的方式，避免随机修改，保持高性能。
```
 /**
 * Traverse uncommitted/unroll back half message and send check back request to producer to obtain transaction
 * status
 **/
@Override
public void check(long transactionTimeout, int transactionCheckMax,
    AbstractTransactionalMessageCheckListener listener) {
    try {
        String topic = TopicValidator.RMQ_SYS_TRANS_HALF_TOPIC;
        //获取半消息set，broker相同queueId相同topic相同的消息相同
        Set<MessageQueue> msgQueues = transactionalMessageBridge.fetchMessageQueues(topic);
        if (msgQueues == null || msgQueues.size() == 0) {
            log.warn("The queue of topic is empty :" + topic);
            return;
        }
        log.debug("Check topic={}, queues={}", topic, msgQueues);
        for (MessageQueue messageQueue : msgQueues) {
            long startTime = System.currentTimeMillis();
            MessageQueue opQueue = getOpQueue(messageQueue);
            long halfOffset = transactionalMessageBridge.fetchConsumeOffset(messageQueue);
            long opOffset = transactionalMessageBridge.fetchConsumeOffset(opQueue);
            log.info("Before check, the queue={} msgOffset={} opOffset={}", messageQueue, halfOffset, opOffset);
            if (halfOffset < 0 || opOffset < 0) {
                log.error("MessageQueue: {} illegal offset read: {}, op offset: {},skip this queue", messageQueue,
                    halfOffset, opOffset);
                continue;
            }

            List<Long> doneOpOffset = new ArrayList<>();
            HashMap<Long, Long> removeMap = new HashMap<>();
            PullResult pullResult = fillOpRemoveMap(removeMap, opQueue, opOffset, halfOffset, doneOpOffset);
            if (null == pullResult) {
                log.error("The queue={} check msgOffset={} with opOffset={} failed, pullResult is null",
                    messageQueue, halfOffset, opOffset);
                continue;
            }
            // single thread
            int getMessageNullCount = 1;
            long newOffset = halfOffset;
            long i = halfOffset;
            while (true) {
                if (System.currentTimeMillis() - startTime > MAX_PROCESS_TIME_LIMIT) {
                    log.info("Queue={} process time reach max={}", messageQueue, MAX_PROCESS_TIME_LIMIT);
                    break;
                }
                if (removeMap.containsKey(i)) {
                    log.info("Half offset {} has been committed/rolled back", i);
                    Long removedOpOffset = removeMap.remove(i);
                    doneOpOffset.add(removedOpOffset);
                } else {
                    GetResult getResult = getHalfMsg(messageQueue, i);
                    MessageExt msgExt = getResult.getMsg();
                    ...
                    
                    //是否丢弃该半消息，超出最大回查次数或者超过72小时
                    if (needDiscard(msgExt, transactionCheckMax) || needSkip(msgExt)) {
                        listener.resolveDiscardMsg(msgExt);
                        newOffset = i + 1;
                        i++;
                        continue;
                    }
                    

                    //CHECK_IMMUNITY_TIME_IN_SECONDS时间内不进行回查
                    long valueOfCurrentMinusBorn = System.currentTimeMillis() - msgExt.getBornTimestamp();
                    long checkImmunityTime = transactionTimeout;
                    String checkImmunityTimeStr = msgExt.getUserProperty(MessageConst.PROPERTY_CHECK_IMMUNITY_TIME_IN_SECONDS);
                    if (null != checkImmunityTimeStr) {
                        checkImmunityTime = getImmunityTime(checkImmunityTimeStr, transactionTimeout);
                        if (valueOfCurrentMinusBorn < checkImmunityTime) {
                            if (checkPrepareQueueOffset(removeMap, doneOpOffset, msgExt)) {
                                newOffset = i + 1;
                                i++;
                                continue;
                            }
                        }
                    } else {
                        if ((0 <= valueOfCurrentMinusBorn) && (valueOfCurrentMinusBorn < checkImmunityTime)) {
                            log.debug("New arrived, the miss offset={}, check it later checkImmunity={}, born={}", i,
                                checkImmunityTime, new Date(msgExt.getBornTimestamp()));
                            break;
                        }
                    }
                    List<MessageExt> opMsg = pullResult.getMsgFoundList();
                    boolean isNeedCheck = (opMsg == null && valueOfCurrentMinusBorn > checkImmunityTime)
                        || (opMsg != null && (opMsg.get(opMsg.size() - 1).getBornTimestamp() - startTime > transactionTimeout))
                        || (valueOfCurrentMinusBorn <= -1);

                    if (isNeedCheck) {
                        //修改了半消息属性后重写半消息
                        if (!putBackHalfMsgQueue(msgExt, i)) {
                            continue;
                        }
                        listener.resolveHalfMsg(msgExt);
                    } else {
                        pullResult = fillOpRemoveMap(removeMap, opQueue, pullResult.getNextBeginOffset(), halfOffset, doneOpOffset);
                        log.debug("The miss offset:{} in messageQueue:{} need to get more opMsg, result is:{}", i,
                            messageQueue, pullResult);
                        continue;
                    }
                }
                newOffset = i + 1;
                i++;
            }
            if (newOffset != halfOffset) {
                transactionalMessageBridge.updateConsumeOffset(messageQueue, newOffset);
            }
            long newOpOffset = calculateOpOffset(doneOpOffset, opOffset);
            if (newOpOffset != opOffset) {
                transactionalMessageBridge.updateConsumeOffset(opQueue, newOpOffset);
            }
        }
    } catch (Throwable e) {
        log.error("Check error", e);
    }

}

//实际回查半消息
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


 



