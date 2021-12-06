## disruptor-3.3.2源码解析(5)-框架支持

作者：大飞

 

- **更方便的使用Disruptor：**

​    前面几篇看了Disruptor中的一些重要组件和组件的运行方式，也通过手动组合这些组件的方式给出了一些基本的用例。框架也提供了一个DSL-style API，来帮助我们更容易的使用框架，屏蔽掉一些细节(比如怎么构建RingBuffer、怎么关联追踪序列等)，相当于Builder模式。

​    在看Disruptor之前，先看一些辅助类，首先看下ConsumerRepository：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **class** ConsumerRepository<T> **implements** Iterable<ConsumerInfo>{ 
2.  
3.   **private** **final** Map<EventHandler<?>, EventProcessorInfo<T>> eventProcessorInfoByEventHandler = **new** IdentityHashMap<EventHandler<?>, EventProcessorInfo<T>>(); 
4.   **private** **final** Map<Sequence, ConsumerInfo> eventProcessorInfoBySequence = **new** IdentityHashMap<Sequence, ConsumerInfo>(); 
5.   **private** **final** Collection<ConsumerInfo> consumerInfos = **new** ArrayList<ConsumerInfo>(); 

​    可见ConsumerRepository内部存储着事件处理者(消费者)的信息，相当于事件处理者的仓库。

​    看一下里面的方法： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** add(**final** EventProcessor eventprocessor, 
2. ​        **final** EventHandler<? **super** T> handler, 
3. ​        **final** SequenceBarrier barrier){ 
4.   **final** EventProcessorInfo<T> consumerInfo = **new** EventProcessorInfo<T>(eventprocessor, handler, barrier); 
5.   eventProcessorInfoByEventHandler.put(handler, consumerInfo); 
6.   eventProcessorInfoBySequence.put(eventprocessor.getSequence(), consumerInfo); 
7.   consumerInfos.add(consumerInfo); 
8. } 

​    添加事件处理者(Event模式)、事件处理器和序列栅栏到仓库中。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** add(**final** EventProcessor processor){ 
2.   **final** EventProcessorInfo<T> consumerInfo = **new** EventProcessorInfo<T>(processor, **null**, **null**); 
3.   eventProcessorInfoBySequence.put(processor.getSequence(), consumerInfo); 
4.   consumerInfos.add(consumerInfo); 
5. } 

​    添加事件处理者(Event模式)到仓库中。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** add(**final** WorkerPool<T> workerPool, **final** SequenceBarrier sequenceBarrier){ 
2.   **final** WorkerPoolInfo<T> workerPoolInfo = **new** WorkerPoolInfo<T>(workerPool, sequenceBarrier); 
3.   consumerInfos.add(workerPoolInfo); 
4.   **for** (Sequence sequence : workerPool.getWorkerSequences()){ 
5. ​    eventProcessorInfoBySequence.put(sequence, workerPoolInfo); 
6.   } 
7. } 

​    添加事件处理者(Work模式)和序列栅栏到仓库中。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** Sequence[] getLastSequenceInChain(**boolean** includeStopped){ 
2.   List<Sequence> lastSequence = **new** ArrayList<Sequence>(); 
3.   **for** (ConsumerInfo consumerInfo : consumerInfos){ 
4. ​    **if** ((includeStopped || consumerInfo.isRunning()) && consumerInfo.isEndOfChain()){ 
5. ​      **final** Sequence[] sequences = consumerInfo.getSequences(); 
6. ​      Collections.addAll(lastSequence, sequences); 
7. ​    } 
8.   } 
9.   **return** lastSequence.toArray(**new** Sequence[lastSequence.size()]); 
10. } 

​    获取当前已经消费到RingBuffer上事件队列末尾的事件处理者的序列，可通过参数指定是否要包含已经停止的事件处理者。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** unMarkEventProcessorsAsEndOfChain(**final** Sequence... barrierEventProcessors){ 
2.   **for** (Sequence barrierEventProcessor : barrierEventProcessors){ 
3. ​    getEventProcessorInfo(barrierEventProcessor).markAsUsedInBarrier(); 
4.   } 
5. } 

​    重置已经处理到事件队列末尾的事件处理者的状态。

 

​    其他方法就不看了。

 

​    上面代码中出现的ConsumerInfo就相当于事件处理者信息和序列栅栏的包装类，ConsumerInfo本身是一个接口，针对Event模式和Work模式提供了两种实现：EventProcessorInfo和WorkerPoolInfo，代码都很容易理解，这里就不贴了。

 

**现在来看下Disruptor类，先看结构：**

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** Disruptor<T>{ 
2.  
3.   //事件队列。 
4.   **private** **final** RingBuffer<T> ringBuffer; 
5.   //用于执行事件处理的执行器。 
6.   **private** **final** Executor executor; 
7.   //事件处理信息仓库。 
8.   **private** **final** ConsumerRepository<T> consumerRepository = **new** ConsumerRepository<T>(); 
9.   //运行状态。 
10.   **private** **final** AtomicBoolean started = **new** AtomicBoolean(**false**); 
11.   //异常处理器。 
12.   **private** ExceptionHandler<? **super** T> exceptionHandler; 

​    可见，Disruptor内部包含了我们之前写用例使用到的所有组件。

 

**再看下构造方法：** 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** Disruptor(**final** EventFactory<T> eventFactory, **final** **int** ringBufferSize, **final** Executor executor){ 
2.   **this**(RingBuffer.createMultiProducer(eventFactory, ringBufferSize), executor); 
3. } 
4.  
5. **public** Disruptor(**final** EventFactory<T> eventFactory, 
6. ​         **final** **int** ringBufferSize, 
7. ​         **final** Executor executor, 
8. ​         **final** ProducerType producerType, 
9. ​         **final** WaitStrategy waitStrategy){ 
10.   **this**(RingBuffer.create(producerType, eventFactory, ringBufferSize, waitStrategy), 
11. ​     executor); 
12. } 
13.  
14. **private** Disruptor(**final** RingBuffer<T> ringBuffer, **final** Executor executor) 
15. { 
16.   **this**.ringBuffer = ringBuffer; 
17.   **this**.executor = executor; 
18. } 

​    可见，通过构造方法，可以对内部的RingBuffer和执行器进行初始化。

 

 **有了事件队列(RingBuffer)，接下来看看怎么构建事件处理者：**

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @SuppressWarnings("varargs") 
2. **public** EventHandlerGroup<T> handleEventsWith(**final** EventHandler<? **super** T>... handlers){ 
3.   **return** createEventProcessors(**new** Sequence[0], handlers); 
4. } 
5. EventHandlerGroup<T> createEventProcessors(**final** Sequence[] barrierSequences, 
6. ​                      **final** EventHandler<? **super** T>[] eventHandlers){ 
7.   checkNotStarted(); 
8.   **final** Sequence[] processorSequences = **new** Sequence[eventHandlers.length]; 
9.   **final** SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences); 
10.   **for** (**int** i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++){ 
11. ​    **final** EventHandler<? **super** T> eventHandler = eventHandlers[i]; 
12. ​    **final** BatchEventProcessor<T> batchEventProcessor = **new** BatchEventProcessor<T>(ringBuffer, barrier, eventHandler); 
13. ​    **if** (exceptionHandler != **null**){ 
14. ​      batchEventProcessor.setExceptionHandler(exceptionHandler); 
15. ​    } 
16. ​    consumerRepository.add(batchEventProcessor, eventHandler, barrier); 
17. ​    processorSequences[i] = batchEventProcessor.getSequence(); 
18.   } 
19.   **if** (processorSequences.length > 0){ 
20. ​    consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences); 
21.   } 
22.   **return** **new** EventHandlerGroup<T>(**this**, consumerRepository, processorSequences); 
23. } 

​    可见，handleEventsWith方法内部会创建BatchEventProcessor。

​    当然，对于Event模式，还有一些玩法，其实之前几篇就看到过，我们可以设置两个EventHandler，然后事件会依次被这两个handler处理。Disruptor类中提供了更明确的定义(事实是结合了EventHandlerGroup的一些方法)，比如我想让事件先被处理器a处理，然后在被处理器b处理，就可以这么写：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. EventHandler<MyEvent> a = **new** EventHandler<MyEvent>() { ... }; 
2. EventHandler<MyEvent> b = **new** EventHandler<MyEvent>() { ... }; 
3. disruptor.handleEventsWith(a); //语句1 
4. disruptor.after(a).handleEventsWith(b); 

​    注意上面必须先写语句1，然后才能针对a调用after，否则after找不到处理器a，会报错。

​    上面的例子也可以这么写： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. EventHandler<MyEvent> a = **new** EventHandler<MyEvent>() { ... }; 
2. EventHandler<MyEvent> b = **new** EventHandler<MyEvent>() { ... }; 
3. disruptor.handleEventsWith(a).then(b); 

​    效果是一样的。

 

​    Disruptor还允许我们定制事件处理者： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1.   **public** EventHandlerGroup<T> handleEventsWith(**final** EventProcessorFactory<T>... eventProcessorFactories){ 
2. ​    **final** Sequence[] barrierSequences = **new** Sequence[0]; 
3. ​    **return** createEventProcessors(barrierSequences, eventProcessorFactories); 
4.   } 
5. **public** **interface** EventProcessorFactory<T>{ 
6.   EventProcessor createEventProcessor(RingBuffer<T> ringBuffer, Sequence[] barrierSequences); 
7. } 

​    handleEventsWith方法内部创建的Event模式的事件处理者，有没有Work模式的呢？

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @SuppressWarnings("varargs") 
2. **public** EventHandlerGroup<T> handleEventsWithWorkerPool(**final** WorkHandler<T>... workHandlers){ 
3.   **return** createWorkerPool(**new** Sequence[0], workHandlers); 
4. } 
5. EventHandlerGroup<T> createWorkerPool(**final** Sequence[] barrierSequences, **final** WorkHandler<? **super** T>[] workHandlers){ 
6.   **final** SequenceBarrier sequenceBarrier = ringBuffer.newBarrier(barrierSequences); 
7.   **final** WorkerPool<T> workerPool = **new** WorkerPool<T>(ringBuffer, sequenceBarrier, exceptionHandler, workHandlers); 
8.   consumerRepository.add(workerPool, sequenceBarrier); 
9.   **return** **new** EventHandlerGroup<T>(**this**, consumerRepository, workerPool.getWorkerSequences()); 
10. } 

​    handleEventsWithWorkerPool内部会创建WorkerPool。

 

**事件处理者也构建好了，接下来看看怎么启动它们：**

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** RingBuffer<T> start(){ 
2.   **final** Sequence[] gatingSequences = consumerRepository.getLastSequenceInChain(**true**); 
3.   ringBuffer.addGatingSequences(gatingSequences); 
4.   checkOnlyStartedOnce(); 
5.   **for** (**final** ConsumerInfo consumerInfo : consumerRepository){ 
6. ​    consumerInfo.start(executor); 
7.   } 
8.   **return** ringBuffer; 
9. } 

​    可见，启动过程中会将事件处理者的序列设置为RingBuffer的追踪序列。最后会启动事件处理者，并利用执行器来执行事件处理线程。

 

**事件处理者构建好了，也启动了，看看怎么发布事件：** 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** publishEvent(**final** EventTranslator<T> eventTranslator){ 
2.   ringBuffer.publishEvent(eventTranslator); 
3. } 

​    很简单，里面就是直接调用了RingBuffer来发布事件，之前几篇都分析过了。

 

 **再看看其他的方法：**

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** halt(){ 
2.   **for** (**final** ConsumerInfo consumerInfo : consumerRepository){ 
3. ​    consumerInfo.halt(); 
4.   } 
5. } 

​    停止事件处理者。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** shutdown(){ 
2.   **try**{ 
3. ​    shutdown(-1, TimeUnit.MILLISECONDS); 
4.   }**catch** (**final** TimeoutException e){ 
5. ​    exceptionHandler.handleOnShutdownException(e); 
6.   } 
7. } 
8. **public** **void** shutdown(**final** **long** timeout, **final** TimeUnit timeUnit) **throws** TimeoutException{ 
9.   **final** **long** timeOutAt = System.currentTimeMillis() + timeUnit.toMillis(timeout); 
10.   **while** (hasBacklog()){ 
11. ​    **if** (timeout >= 0 && System.currentTimeMillis() > timeOutAt){ 
12. ​      **throw** TimeoutException.INSTANCE; 
13. ​    } 
14. ​    // Busy spin 
15.   } 
16.   halt(); 
17. } 
18. **private** **boolean** hasBacklog(){ 
19.   **final** **long** cursor = ringBuffer.getCursor(); 
20.   **for** (**final** Sequence consumer : consumerRepository.getLastSequenceInChain(**false**)){ 
21. ​    **if** (cursor > consumer.get()){ 
22. ​      **return** **true**; 
23. ​    } 
24.   } 
25.   **return** **false**; 
26. } 

​    等待所有能处理的事件都处理完了，再定制事件处理者，有超时选项。

 

 **好了，其他方法都比较简单，不看了。最后来使用Disruptor写一个生产者消费者模式吧：**

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   //创建一个执行器(线程池)。 
3.   Executor executor = Executors.newFixedThreadPool(4); 
4.   //创建一个Disruptor。 
5.   Disruptor<MyDataEvent> disruptor =  
6. ​      **new** Disruptor<MyDataEvent>(**new** MyDataEventFactory(), 4, executor); 
7.   //创建两个事件处理器。 
8.   MyDataEventHandler handler1 = **new** MyDataEventHandler(); 
9.   KickAssEventHandler handler2 = **new** KickAssEventHandler(); 
10.   //同一个事件，先用handler1处理再用handler2处理。 
11.   disruptor.handleEventsWith(handler1).then(handler2); 
12.   //启动Disruptor。 
13.   disruptor.start(); 
14.   //发布10个事件。 
15.   **for**(**int** i=0;i<10;i++){ 
16. ​    disruptor.publishEvent(**new** MyDataEventTranslator()); 
17. ​    System.out.println("发布事件["+i+"]"); 
18. ​    **try** { 
19. ​      TimeUnit.SECONDS.sleep(3); 
20. ​    } **catch** (InterruptedException e) { 
21. ​      e.printStackTrace(); 
22. ​    } 
23.   } 
24.   **try** { 
25. ​    System.in.read(); 
26.   } **catch** (IOException e) { 
27. ​    e.printStackTrace(); 
28.   } 
29. } 

​    看下输出： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. 发布事件[0] 
2. handle event's data:MyData [id=0, value=holy shit!]isEndOfBatch:**true** 
3. kick your ass 0 times!!!! 
4. 发布事件[1] 
5. handle event's data:MyData [id=1, value=holy shit!]isEndOfBatch:**true** 
6. kick your ass 1 times!!!! 
7. 发布事件[2] 
8. handle event's data:MyData [id=2, value=holy shit!]isEndOfBatch:**true** 
9. kick your ass 2 times!!!! 
10. 发布事件[3] 
11. handle event's data:MyData [id=3, value=holy shit!]isEndOfBatch:**true** 
12. kick your ass 3 times!!!! 
13. 发布事件[4] 
14. handle event's data:MyData [id=4, value=holy shit!]isEndOfBatch:**true** 
15. kick your ass 4 times!!!! 
16. 发布事件[5] 
17. handle event's data:MyData [id=5, value=holy shit!]isEndOfBatch:**true** 
18. kick your ass 5 times!!!! 
19. 发布事件[6] 
20. handle event's data:MyData [id=6, value=holy shit!]isEndOfBatch:**true** 
21. kick your ass 6 times!!!! 
22. 发布事件[7] 
23. handle event's data:MyData [id=7, value=holy shit!]isEndOfBatch:**true** 
24. kick your ass 7 times!!!! 
25. 发布事件[8] 
26. handle event's data:MyData [id=8, value=holy shit!]isEndOfBatch:**true** 
27. kick your ass 8 times!!!! 
28. 发布事件[9] 
29. handle event's data:MyData [id=9, value=holy shit!]isEndOfBatch:**true** 
30. kick your ass 9 times!!!! 

  

- **最后总结：**

​    **1.使用时可以直接使用Disruptor这个类来更方便的完成代码编写，注意灵活使用。**

​    **2.最后别忘了单线程/多线程生产者、Event/Work处理模式等等。**