## disruptor-3.3.2源码解析(4)-处理事件

作者：大飞

 

- **Disruptor中如何处理事件：**

​    disruptor中提供了专门的事件处理器接口，先看下接口定义：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. /** 
2.  \* 事件处理器会等待RingBuffer中的事件变为可用(可处理)，然后处理可用的事件。 
3.  \* 一个事件处理器通常会关联一个线程。 
4.  */ 
5. **public** **interface** EventProcessor **extends** Runnable{ 
6.   /** 
7.    \* 获取一个事件处理器使用的序列引用。 
8.    */ 
9.   Sequence getSequence(); 
10.  
11.   **void** halt(); 
12.   **boolean** isRunning(); 
13. } 

​    接下来看一个实现，BatchEventProcessor。先看下内部结构：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** BatchEventProcessor<T> 
2.   **implements** EventProcessor{ 
3.  
4.   //表示当前事件处理器的运行状态 
5.   **private** **final** AtomicBoolean running = **new** AtomicBoolean(**false**); 
6.   //异常处理器。 
7.   **private** ExceptionHandler<? **super** T> exceptionHandler = **new** FatalExceptionHandler(); 
8.   //数据提供者。(RingBuffer) 
9.   **private** **final** DataProvider<T> dataProvider; 
10.   //序列栅栏 
11.   **private** **final** SequenceBarrier sequenceBarrier; 
12.   //正真处理事件的回调接口。 
13.   **private** **final** EventHandler<? **super** T> eventHandler; 
14.   //事件处理器使用的序列。 
15.   **private** **final** Sequence sequence = **new** Sequence(Sequencer.INITIAL_CURSOR_VALUE); 
16.   //超时处理器。 
17.   **private** **final** TimeoutHandler timeoutHandler; 
18.  
19.   **public** BatchEventProcessor(**final** DataProvider<T> dataProvider, 
20. ​                **final** SequenceBarrier sequenceBarrier, 
21. ​                **final** EventHandler<? **super** T> eventHandler){ 
22. ​    **this**.dataProvider = dataProvider; 
23. ​    **this**.sequenceBarrier = sequenceBarrier; 
24. ​    **this**.eventHandler = eventHandler; 
25. ​    **if** (eventHandler **instanceof** SequenceReportingEventHandler){ 
26. ​      ((SequenceReportingEventHandler<?>)eventHandler).setSequenceCallback(sequence); 
27. ​    } 
28. ​    timeoutHandler = (eventHandler **instanceof** TimeoutHandler) ? (TimeoutHandler) eventHandler : **null**; 
29.   } 

​    大体对内部结构有个印象，然后看一下功能方法的实现：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** Sequence getSequence(){ 
3.   **return** sequence; 
4. } 
5. @Override 
6. **public** **void** halt(){ 
7.   //设置运行状态为false。 
8.   running.set(**false**); 
9.   //通知序列栅栏。 
10.   sequenceBarrier.alert(); 
11. } 
12. @Override 
13. **public** **boolean** isRunning(){ 
14.   **return** running.get(); 
15. } 

​    这几个方法都比较容易理解，重点看下run方法： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **void** run(){ 
3.   //状态设置与检测。 
4.   **if** (!running.compareAndSet(**false**, **true**)){ 
5. ​    **throw** **new** IllegalStateException("Thread is already running"); 
6.   } 
7.   //先清除序列栅栏的通知状态。 
8.   sequenceBarrier.clearAlert(); 
9.   //如果eventHandler实现了LifecycleAware，这里会对其进行一个启动通知。 
10.   notifyStart(); 
11.   T event = **null**; 
12.   //获取要申请的序列值。 
13.   **long** nextSequence = sequence.get() + 1L; 
14.   **try**{ 
15. ​    **while** (**true**){ 
16. ​      **try**{ 
17. ​        //通过序列栅栏来等待可用的序列值。 
18. ​        **final** **long** availableSequence = sequenceBarrier.waitFor(nextSequence); 
19. ​        //得到可用的序列值后，批量处理nextSequence到availableSequence之间的事件。 
20. ​        **while** (nextSequence <= availableSequence){ 
21. ​          //获取事件。 
22. ​          event = dataProvider.get(nextSequence); 
23. ​          //将事件交给eventHandler处理。 
24. ​          eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence); 
25. ​          nextSequence++; 
26. ​        } 
27. ​        //处理完毕后，设置当前处理完成的最后序列值。 
28. ​        sequence.set(availableSequence); 
29. ​        //继续循环。 
30. ​      } 
31. ​      **catch** (**final** TimeoutException e){ 
32. ​        //如果发生超时，通知一下超时处理器(如果eventHandler同时实现了timeoutHandler，会将其设置为当前的超时处理器) 
33. ​        notifyTimeout(sequence.get()); 
34. ​      }**catch** (**final** AlertException ex){ 
35. ​        //如果捕获了序列栅栏变更通知，并且当前事件处理器停止了，那么退出主循环。 
36. ​        **if** (!running.get()){ 
37. ​          **break**; 
38. ​        } 
39. ​      }**catch** (**final** Throwable ex){ 
40. ​        //其他的异常都交给异常处理器进行处理。 
41. ​        exceptionHandler.handleEventException(ex, nextSequence, event); 
42. ​        //处理异常后仍然会设置当前处理的最后的序列值，然后继续处理其他事件。 
43. ​        sequence.set(nextSequence); 
44. ​        nextSequence++; 
45. ​      } 
46. ​    } 
47.   } 
48.   **finally**{ 
49. ​    //主循环退出后，如果eventHandler实现了LifecycleAware，这里会对其进行一个停止通知。 
50. ​    notifyShutdown(); 
51. ​    //设置事件处理器运行状态为停止。 
52. ​    running.set(**false**); 
53.   } 
54. } 

​    贴上其他方法：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** setExceptionHandler(**final** ExceptionHandler<? **super** T> exceptionHandler){ 
2.   **if** (**null** == exceptionHandler){ 
3. ​    **throw** **new** NullPointerException(); 
4.   } 
5.   **this**.exceptionHandler = exceptionHandler; 
6. } 
7.  
8. **private** **void** notifyTimeout(**final** **long** availableSequence){ 
9.   **try**{ 
10. ​    **if** (timeoutHandler != **null**){ 
11. ​      timeoutHandler.onTimeout(availableSequence); 
12. ​    } 
13.   } 
14.   **catch** (Throwable e){ 
15. ​    exceptionHandler.handleEventException(e, availableSequence, **null**); 
16.   } 
17. } 
18.  
19. **private** **void** notifyStart(){ 
20.   **if** (eventHandler **instanceof** LifecycleAware){ 
21. ​    **try**{ 
22. ​      ((LifecycleAware)eventHandler).onStart(); 
23. ​    }**catch** (**final** Throwable ex){ 
24. ​      exceptionHandler.handleOnStartException(ex); 
25. ​    } 
26.   } 
27. } 
28.  
29. **private** **void** notifyShutdown(){ 
30.   **if** (eventHandler **instanceof** LifecycleAware){ 
31. ​    **try**{ 
32. ​      ((LifecycleAware)eventHandler).onShutdown(); 
33. ​    }**catch** (**final** Throwable ex){ 
34. ​      exceptionHandler.handleOnShutdownException(ex); 
35. ​    } 
36.   } 
37. } 

​    总结一下BatchEventProcessor：

​       \1. BatchEventProcessor内部会记录自己的序列、运行状态。

​       \2. BatchEventProcessor需要外部提供数据提供者(其实就是队列-RingBuffer)、序列栅栏、异常处理器。

​       \3. BatchEventProcessor其实是将事件委托给内部的EventHandler来处理的。

 

​    知道了这些，再结合前几篇的内容，来写个生产者/消费者模式的代码爽一下吧！

​    首先，要定义具体处理事件的EventHandler，先来看下这个接口：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** EventHandler<T>{ 
2.   /** 
3.    \* Called when a publisher has published an event to the {@link RingBuffer} 
4.    \* 
5.    \* @param event published to the {@link RingBuffer} 
6.    \* @param sequence of the event being processed 
7.    \* @param endOfBatch flag to indicate if this is the last event in a batch from the {@link RingBuffer} 
8.    \* @throws Exception if the EventHandler would like the exception handled further up the chain. 
9.    */ 
10.   **void** onEvent(T event, **long** sequence, **boolean** endOfBatch) **throws** Exception; 
11. } 

​    接口很明确，没什么要解释的，然后定义我们的具体处理方式： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyDataEventHandler **implements** EventHandler<MyDataEvent>{ 
2.   @Override 
3.   **public** **void** onEvent(MyDataEvent event, **long** sequence, **boolean** endOfBatch) 
4. ​      **throws** Exception { 
5. ​    //注意这里小睡眠了一下!! 
6. ​    TimeUnit.SECONDS.sleep(3); 
7. ​    System.out.println("handle event's data:" + event.getData() +"isEndOfBatch:"+endOfBatch); 
8.   } 
9. } 

​    然后是主程序：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   //创建一个RingBuffer，注意容量是4。 
3.   RingBuffer<MyDataEvent> ringBuffer =  
4. ​      RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 4); 
5.   //创建一个事件处理器。 
6.   BatchEventProcessor<MyDataEvent> batchEventProcessor = 
7. ​      /* 
8. ​       \* 注意参数：数据提供者就是RingBuffer、序列栅栏也来自RingBuffer 
9. ​       \* EventHandler使用自定义的。 
10. ​       */ 
11. ​      **new** BatchEventProcessor<MyDataEvent>(ringBuffer,  
12. ​          ringBuffer.newBarrier(), **new** MyDataEventHandler()); 
13.   //将事件处理器本身的序列设置为ringBuffer的追踪序列。 
14.   ringBuffer.addGatingSequences(batchEventProcessor.getSequence()); 
15. ​    //启动事件处理器。 
16.   **new** Thread(batchEventProcessor).start(); 
17.   //往RingBuffer上发布事件 
18.   **for**(**int** i=0;i<10;i++){ 
19. ​    ringBuffer.publishEvent(**new** MyDataEventTranslatorWithIdAndValue(), i, i+"s"); 
20. ​    System.out.println("发布事件["+i+"]"); 
21.   } 
22.    
23.   **try** { 
24. ​    System.in.read(); 
25.   } **catch** (IOException e) { 
26. ​    e.printStackTrace(); 
27.   } 
28. } 

​    注意到当前RingBuffer只有4个空间，然后发了10个事件，而且消费者内部会sleep一下，所以会不会出现事件覆盖的情况呢(生产者在RingBuffer上绕了一圈从后面追上了消费者)。

​    看下输出： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. 发布事件[0] 
2. 发布事件[1] 
3. 发布事件[2] 
4. 发布事件[3] 
5. handle event's data:MyData [id=0, value=0s]isEndOfBatch:**true** 
6. 发布事件[4] 
7. handle event's data:MyData [id=1, value=1s]isEndOfBatch:**false** 
8. handle event's data:MyData [id=2, value=2s]isEndOfBatch:**false** 
9. handle event's data:MyData [id=3, value=3s]isEndOfBatch:**true** 
10. 发布事件[5] 
11. 发布事件[6] 
12. 发布事件[7] 
13. handle event's data:MyData [id=4, value=4s]isEndOfBatch:**true** 
14. 发布事件[8] 
15. handle event's data:MyData [id=5, value=5s]isEndOfBatch:**false** 
16. handle event's data:MyData [id=6, value=6s]isEndOfBatch:**false** 
17. handle event's data:MyData [id=7, value=7s]isEndOfBatch:**true** 
18. 发布事件[9] 
19. handle event's data:MyData [id=8, value=8s]isEndOfBatch:**true** 
20. handle event's data:MyData [id=9, value=9s]isEndOfBatch:**true** 

​    发现其实并没有覆盖事件的情况，而是发布程序会等待事件处理程序处理完毕，有可发布的空间，再进行发布。还记得前面文章分析的序列的next()方法，里面会通过观察追踪序列的情况来决定是否等待，也就是说，RingBuffer需要追踪事件处理器的序列，这个关系怎么确立的呢？就是上面代码中的这句话： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. //将事件处理器本身的序列设置为ringBuffer的追踪序列。 
2. ringBuffer.addGatingSequences(batchEventProcessor.getSequence()); 

​    接下来再改一下程序，将MyDataEventHandler中的睡眠代码去掉，然后主程序中发布事件后加上睡眠代码：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyDataEventHandler **implements** EventHandler<MyDataEvent>{ 
2.   @Override 
3.   **public** **void** onEvent(MyDataEvent event, **long** sequence, **boolean** endOfBatch) 
4. ​      **throws** Exception { 
5. ​    //TimeUnit.SECONDS.sleep(3); 
6. ​    System.out.println("handle event's data:" + event.getData() +"isEndOfBatch:"+endOfBatch); 
7.   } 
8. } 
9.   //主程序 
10.   **public** **static** **void** main(String[] args) { 
11. ​    //创建一个RingBuffer，注意容量是4。 
12. ​    RingBuffer<MyDataEvent> ringBuffer =  
13. ​        RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 4); 
14. ​    //创建一个事件处理器。 
15. ​    BatchEventProcessor<MyDataEvent> batchEventProcessor = 
16. ​        /* 
17. ​         \* 注意参数：数据提供者就是RingBuffer、序列栅栏也来自RingBuffer 
18. ​         \* EventHandler使用自定义的。 
19. ​         */ 
20. ​        **new** BatchEventProcessor<MyDataEvent>(ringBuffer,  
21. ​            ringBuffer.newBarrier(), **new** MyDataEventHandler()); 
22. ​    //将事件处理器本身的序列设置为ringBuffer的追踪序列。 
23. ​    ringBuffer.addGatingSequences(batchEventProcessor.getSequence()); 
24. ​    //启动事件处理器。 
25. ​    **new** Thread(batchEventProcessor).start(); 
26. ​    //往RingBuffer上发布事件 
27. ​    **for**(**int** i=0;i<10;i++){ 
28. ​      ringBuffer.publishEvent(**new** MyDataEventTranslatorWithIdAndValue(), i, i+"s"); 
29. ​      System.out.println("发布事件["+i+"]"); 
30. ​      **try** { 
31. ​        TimeUnit.SECONDS.sleep(3);//睡眠！！！ 
32. ​      } **catch** (InterruptedException e) { 
33. ​        e.printStackTrace(); 
34. ​      } 
35. ​    } 
36. ​     
37. ​    **try** { 
38. ​      System.in.read(); 
39. ​    } **catch** (IOException e) { 
40. ​      e.printStackTrace(); 
41. ​    } 
42.   } 

​    这种情况下，事件处理很快，但是发布事件很慢，事件处理器会等待事件的发布么？

​    看下输出： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. 发布事件[0] 
2. handle event's data:MyData [id=0, value=0s]isEndOfBatch:**true** 
3. 发布事件[1] 
4. handle event's data:MyData [id=1, value=1s]isEndOfBatch:**true** 
5. 发布事件[2] 
6. handle event's data:MyData [id=2, value=2s]isEndOfBatch:**true** 
7. 发布事件[3] 
8. handle event's data:MyData [id=3, value=3s]isEndOfBatch:**true** 
9. 发布事件[4] 
10. handle event's data:MyData [id=4, value=4s]isEndOfBatch:**true** 
11. 发布事件[5] 
12. handle event's data:MyData [id=5, value=5s]isEndOfBatch:**true** 
13. 发布事件[6] 
14. handle event's data:MyData [id=6, value=6s]isEndOfBatch:**true** 
15. 发布事件[7] 
16. handle event's data:MyData [id=7, value=7s]isEndOfBatch:**true** 
17. 发布事件[8] 
18. handle event's data:MyData [id=8, value=8s]isEndOfBatch:**true** 
19. 发布事件[9] 
20. handle event's data:MyData [id=9, value=9s]isEndOfBatch:**true** 

​    可见是会等待的，关键就是，事件处理器用的是RingBuffer的序列栅栏，会在栅栏上waitFor事件发布者： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **new** BatchEventProcessor<MyDataEvent>(ringBuffer,/*注意这里*/ringBuffer.newBarrier(), **new** MyDataEventHandler()); 

​    接下来，我们在改变一下姿势，再加一个事件处理器，看看会发生什么：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** KickAssEventHandler **implements** EventHandler<MyDataEvent>{ 
2.   @Override 
3.   **public** **void** onEvent(MyDataEvent event, **long** sequence, **boolean** endOfBatch) 
4. ​      **throws** Exception { 
5. ​    System.out.println("kick your ass "+sequence+" times!!!!"); 
6.   } 
7. } 

​    主程序如下：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   //创建一个RingBuffer，注意容量是4。 
3.   RingBuffer<MyDataEvent> ringBuffer =  
4. ​      RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 4); 
5.   //创建一个事件处理器。 
6.   BatchEventProcessor<MyDataEvent> batchEventProcessor = 
7. ​      /* 
8. ​       \* 注意参数：数据提供者就是RingBuffer、序列栅栏也来自RingBuffer 
9. ​       \* EventHandler使用自定义的。 
10. ​       */ 
11. ​      **new** BatchEventProcessor<MyDataEvent>(ringBuffer,  
12. ​          ringBuffer.newBarrier(), **new** MyDataEventHandler()); 
13.   //创建一个事件处理器。 
14.   BatchEventProcessor<MyDataEvent> batchEventProcessor2 = 
15. ​      /* 
16. ​       \* 注意参数：数据提供者就是RingBuffer、序列栅栏也来自RingBuffer 
17. ​       \* EventHandler使用自定义的。 
18. ​       */ 
19. ​      **new** BatchEventProcessor<MyDataEvent>(ringBuffer,  
20. ​          ringBuffer.newBarrier(), **new** KickAssEventHandler()); 
21.   //将事件处理器本身的序列设置为ringBuffer的追踪序列。 
22.   ringBuffer.addGatingSequences(batchEventProcessor.getSequence()); 
23.   ringBuffer.addGatingSequences(batchEventProcessor2.getSequence()); 
24.   //启动事件处理器。 
25.   **new** Thread(batchEventProcessor).start(); 
26.   **new** Thread(batchEventProcessor2).start(); 
27.   //往RingBuffer上发布事件 
28.   **for**(**int** i=0;i<10;i++){ 
29. ​    ringBuffer.publishEvent(**new** MyDataEventTranslatorWithIdAndValue(), i, i+"s"); 
30. ​    System.out.println("发布事件["+i+"]"); 
31. ​    **try** { 
32. ​      TimeUnit.SECONDS.sleep(3); 
33. ​    } **catch** (InterruptedException e) { 
34. ​      e.printStackTrace(); 
35. ​    } 
36.   } 
37.   **try** { 
38. ​    System.in.read(); 
39.   } **catch** (IOException e) { 
40. ​    e.printStackTrace(); 
41.   } 
42. } 

​    看下输出：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. 发布事件[0] 
2. kick your ass 0 times!!!! 
3. handle event's data:MyData [id=0, value=0s]isEndOfBatch:**true** 
4. 发布事件[1] 
5. kick your ass 1 times!!!! 
6. handle event's data:MyData [id=1, value=1s]isEndOfBatch:**true** 
7. 发布事件[2] 
8. handle event's data:MyData [id=2, value=2s]isEndOfBatch:**true** 
9. kick your ass 2 times!!!! 
10. 发布事件[3] 
11. handle event's data:MyData [id=3, value=3s]isEndOfBatch:**true** 
12. kick your ass 3 times!!!! 
13. 发布事件[4] 
14. kick your ass 4 times!!!! 
15. handle event's data:MyData [id=4, value=4s]isEndOfBatch:**true** 
16. 发布事件[5] 
17. kick your ass 5 times!!!! 
18. handle event's data:MyData [id=5, value=5s]isEndOfBatch:**true** 
19. 发布事件[6] 
20. handle event's data:MyData [id=6, value=6s]isEndOfBatch:**true** 
21. kick your ass 6 times!!!! 
22. 发布事件[7] 
23. kick your ass 7 times!!!! 
24. handle event's data:MyData [id=7, value=7s]isEndOfBatch:**true** 
25. 发布事件[8] 
26. kick your ass 8 times!!!! 
27. handle event's data:MyData [id=8, value=8s]isEndOfBatch:**true** 
28. 发布事件[9] 
29. kick your ass 9 times!!!! 
30. handle event's data:MyData [id=9, value=9s]isEndOfBatch:**true** 

​    相当于有两个消费者，消费相同的数据，又有点像发布/订阅模式了，呵呵。

 

​    当然，按照上面源码的分析，我们还可以让我们的Eventhandler同时实现TimeoutHandler和LifecycleAware来做更多的事；还可以定制一个ExceptionHandler来处理异常情况(框架中也提供了IgnoreExceptionHandler和FatalExceptionHandler两种ExceptionHandler实现，它们在异常处理时会记录不用级别的日志，后者还会抛出运行时异常)。

 

​    Disruptor还提供了一个队列接口-EventPoller。通过这个接口也可以用RingBuffer中获取事件并处理，而且这个获取方式是无等待的，如果当前没有可处理的事件，会返回相应的状态-PollState。但注释说明这还是个实验性质的类，这里就不分析了。

 

 

- **Work消费模式：**

​    看完上面的内容，可能会有这样的疑惑：实际用的时候可能不会只用单线程来消费吧，能不能使用多个线程来消费同一批事件，每个线程消费这批事件中的一部分呢？

​    disruptor对这种情况也进行了支持，如果上面的叫Event模式，那么这个就叫Work模式吧(不要在意这些名词，知道是怎么回事儿就行了...)

 

​    首先要看下WorkProcessor，还是先看结构： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** WorkProcessor<T> 
2.   **implements** EventProcessor{ 
3.  
4.   **private** **final** AtomicBoolean running = **new** AtomicBoolean(**false**); 
5.   **private** **final** Sequence sequence = **new** Sequence(Sequencer.INITIAL_CURSOR_VALUE); 
6.   **private** **final** RingBuffer<T> ringBuffer; 
7.   **private** **final** SequenceBarrier sequenceBarrier; 
8.   **private** **final** WorkHandler<? **super** T> workHandler; 
9.   **private** **final** ExceptionHandler<? **super** T> exceptionHandler; 
10.   **private** **final** Sequence workSequence; 
11.   **private** **final** EventReleaser eventReleaser = **new** EventReleaser(){ 
12. ​    @Override 
13. ​    **public** **void** release(){ 
14. ​      sequence.set(Long.MAX_VALUE); 
15. ​    } 
16.   }; 
17.  
18.   **public** WorkProcessor(**final** RingBuffer<T> ringBuffer, 
19. ​             **final** SequenceBarrier sequenceBarrier, 
20. ​             **final** WorkHandler<? **super** T> workHandler, 
21. ​             **final** ExceptionHandler<? **super** T> exceptionHandler, 
22. ​             **final** Sequence workSequence){ 
23. ​    **this**.ringBuffer = ringBuffer; 
24. ​    **this**.sequenceBarrier = sequenceBarrier; 
25. ​    **this**.workHandler = workHandler; 
26. ​    **this**.exceptionHandler = exceptionHandler; 
27. ​    **this**.workSequence = workSequence; 
28. ​    **if** (**this**.workHandler **instanceof** EventReleaseAware){ 
29. ​      ((EventReleaseAware)**this**.workHandler).setEventReleaser(eventReleaser); 
30. ​    } 
31.   } 

​    看起来结构和BatchEventProcessor类似，区别是EventHandler变成了WorkHandler，还有了额外的workSequence和一个eventReleaser。

​    

​    看下功能方法实现： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** Sequence getSequence(){ 
3.   **return** sequence; 
4. } 
5. @Override 
6. **public** **void** halt(){ 
7.   running.set(**false**); 
8.   sequenceBarrier.alert(); 
9. } 
10. @Override 
11. **public** **boolean** isRunning(){ 
12.   **return** running.get(); 
13. } 

​    这些方法和BatchEventProcessor的一样，不说了。看下主逻辑：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **void** run(){ 
3.   //状态设置与检测。 
4.   **if** (!running.compareAndSet(**false**, **true**)){ 
5. ​    **throw** **new** IllegalStateException("Thread is already running"); 
6.   } 
7.   //先清除序列栅栏的通知状态。 
8.   sequenceBarrier.clearAlert(); 
9.   //如果workHandler实现了LifecycleAware，这里会对其进行一个启动通知。 
10.   notifyStart(); 
11.   **boolean** processedSequence = **true**; 
12.   **long** cachedAvailableSequence = Long.MIN_VALUE; 
13.   **long** nextSequence = sequence.get(); 
14.   T event = **null**; 
15.   **while** (**true**){ 
16. ​    **try**{ 
17. ​      //判断上一个事件是否已经处理完毕。 
18. ​      **if** (processedSequence){ 
19. ​        //如果处理完毕，重置标识。 
20. ​        processedSequence = **false**; 
21. ​        //原子的获取下一要处理事件的序列值。 
22. ​        **do**{ 
23. ​          nextSequence = workSequence.get() + 1L; 
24. ​          sequence.set(nextSequence - 1L); 
25. ​        }**while** (!workSequence.compareAndSet(nextSequence - 1L, nextSequence)); 
26. ​      } 
27. ​      //检查序列值是否需要申请。这一步是为了防止和事件生产者冲突。 
28. ​      **if** (cachedAvailableSequence >= nextSequence){ 
29. ​        //从RingBuffer上获取事件。 
30. ​        event = ringBuffer.get(nextSequence); 
31. ​        //委托给workHandler处理事件。 
32. ​        workHandler.onEvent(event); 
33. ​        //设置事件处理完成标识。 
34. ​        processedSequence = **true**; 
35. ​      }**else**{ 
36. ​        //如果需要申请，通过序列栅栏来申请可用的序列。 
37. ​        cachedAvailableSequence = sequenceBarrier.waitFor(nextSequence); 
38. ​      } 
39. ​    }**catch** (**final** AlertException ex){ 
40. ​      //处理通知。 
41. ​      **if** (!running.get()){ 
42. ​        //如果当前处理器被停止，那么退出主循环。 
43. ​        **break**; 
44. ​      } 
45. ​    }**catch** (**final** Throwable ex){ 
46. ​      //处理异常 
47. ​      exceptionHandler.handleEventException(ex, nextSequence, event); 
48. ​      //如果异常处理器不抛出异常的话，就认为事件处理完毕，设置事件处理完成标识。 
49. ​      processedSequence = **true**; 
50. ​    } 
51.   } 
52.   //退出主循环后，如果workHandler实现了LifecycleAware，这里会对其进行一个关闭通知。 
53.   notifyShutdown(); 
54.   //设置当前处理器状态为停止。 
55.   running.set(**false**); 
56. } 

​    说明一下WorkProcessor的主逻辑中几个重点：

​       1.首先，由于是Work模式，必然是多个事件处理者(WorkProcessor)处理同一批事件，那么肯定会存在多个处理者对同一个要处理事件的竞争，所以出现了一个workSequence，所有的处理者都使用这一个workSequence，大家通过对workSequence的原子操作来保证不会处理相同的事件。

​       2.其次，多个事件处理者和事件发布者之间也需要协调，需要等待事件发布者发布完事件之后才能对其进行处理，这里还是使用序列栅栏来协调(sequenceBarrier.waitFor)。

 

​    接下来再看一下Work模式下另一个重要的类-WorkerPool：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** WorkerPool<T>{ 
2.  
3.   //运行状态标识。 
4.   **private** **final** AtomicBoolean started = **new** AtomicBoolean(**false**); 
5.   //工作序列。 
6.   **private** **final** Sequence workSequence = **new** Sequence(Sequencer.INITIAL_CURSOR_VALUE); 
7.   //事件队列。 
8.   **private** **final** RingBuffer<T> ringBuffer; 
9.   //事件处理器数组。 
10.   **private** **final** WorkProcessor<?>[] workProcessors; 
11.  
12.   **public** WorkerPool(**final** RingBuffer<T> ringBuffer, 
13. ​           **final** SequenceBarrier sequenceBarrier, 
14. ​           **final** ExceptionHandler<? **super** T> exceptionHandler, 
15. ​           **final** WorkHandler<? **super** T>... workHandlers){ 
16. ​    **this**.ringBuffer = ringBuffer; 
17. ​    **final** **int** numWorkers = workHandlers.length; 
18. ​    workProcessors = **new** WorkProcessor[numWorkers]; 
19. ​    **for** (**int** i = 0; i < numWorkers; i++){ 
20. ​      workProcessors[i] = **new** WorkProcessor<T>(ringBuffer, 
21. ​                           sequenceBarrier, 
22. ​                           workHandlers[i], 
23. ​                           exceptionHandler, 
24. ​                           workSequence); 
25. ​    } 
26.   } 
27.  
28.   **public** WorkerPool(**final** EventFactory<T> eventFactory, 
29. ​           **final** ExceptionHandler<? **super** T> exceptionHandler, 
30. ​           **final** WorkHandler<? **super** T>... workHandlers){ 
31. ​    ringBuffer = RingBuffer.createMultiProducer(eventFactory, 1024, **new** BlockingWaitStrategy()); 
32. ​    **final** SequenceBarrier barrier = ringBuffer.newBarrier(); 
33. ​    **final** **int** numWorkers = workHandlers.length; 
34. ​    workProcessors = **new** WorkProcessor[numWorkers]; 
35. ​    **for** (**int** i = 0; i < numWorkers; i++){ 
36. ​      workProcessors[i] = **new** WorkProcessor<T>(ringBuffer, 
37. ​                           barrier, 
38. ​                           workHandlers[i], 
39. ​                           exceptionHandler, 
40. ​                           workSequence); 
41. ​    } 
42. ​    ringBuffer.addGatingSequences(getWorkerSequences()); 
43.   } 

​    WorkerPool是Work模式下的事件处理器池，它起到了维护事件处理器生命周期、关联事件处理器与事件队列(RingBuffer)、提供工作序列(上面分析WorkProcessor时看到多个处理器需要使用统一的workSequence)的作用。

 

​    看一下WorkerPool的方法：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** Sequence[] getWorkerSequences(){ 
2.   **final** Sequence[] sequences = **new** Sequence[workProcessors.length + 1]; 
3.   **for** (**int** i = 0, size = workProcessors.length; i < size; i++){ 
4. ​    sequences[i] = workProcessors[i].getSequence(); 
5.   } 
6.   sequences[sequences.length - 1] = workSequence; 
7.   **return** sequences; 
8. } 

​    通过WorkerPool是可以获取内部事件处理器各自的序列和当前的WorkSequence，用于观察事件处理进度。  

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** RingBuffer<T> start(**final** Executor executor){ 
2.   **if** (!started.compareAndSet(**false**, **true**)){ 
3. ​    **throw** **new** IllegalStateException("WorkerPool has already been started and cannot be restarted until halted."); 
4.   } 
5.   **final** **long** cursor = ringBuffer.getCursor(); 
6.   workSequence.set(cursor); 
7.   **for** (WorkProcessor<?> processor : workProcessors){ 
8. ​    processor.getSequence().set(cursor); 
9. ​    executor.execute(processor); 
10.   } 
11.   **return** ringBuffer; 
12. } 

​    start方法里面会初始化工作序列，然后使用一个给定的执行器(线程池)来执行内部的事件处理器。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** drainAndHalt(){ 
2.   Sequence[] workerSequences = getWorkerSequences(); 
3.   **while** (ringBuffer.getCursor() > Util.getMinimumSequence(workerSequences)){ 
4. ​    Thread.yield(); 
5.   } 
6.   **for** (WorkProcessor<?> processor : workProcessors){ 
7. ​    processor.halt(); 
8.   } 
9.   started.set(**false**); 
10. } 

​    drainAndHalt方法会将RingBuffer中所有的事件取出，执行完毕后，然后停止当前WorkerPool。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** halt(){ 
2.   **for** (WorkProcessor<?> processor : workProcessors){ 
3. ​    processor.halt(); 
4.   } 
5.   started.set(**false**); 
6. } 

​    马上停止当前WorkerPool。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **boolean** isRunning(){ 
2.   **return** started.get(); 
3. } 

​    获取当前WorkerPool运行状态。

 

​    好了，有了WorkProcessor和WorkerPool，我们又可以嗨皮的写一下多线程消费的生产者/消费者模式了！

​    还记得WorkProcessor由内部的WorkHandler来具体处理事件吧，所以先写一个WorkHandler：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyDataWorkHandler **implements** WorkHandler<MyDataEvent>{ 
2.    
3.   **private** String name; 
4.    
5.   **public** MyDataWorkHandler(String name) { 
6. ​    **this**.name = name; 
7.   } 
8.   @Override 
9.   **public** **void** onEvent(MyDataEvent event) **throws** Exception { 
10. ​    System.out.println("WorkHandler["+name+"]处理事件"+event.getData()); 
11.   } 
12. } 

​    然后是主程序：(其他的类源码见上篇) 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   //创建一个RingBuffer，注意容量是4。 
3.   RingBuffer<MyDataEvent> ringBuffer =  
4. ​      RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 4); 
5.   //创建3个WorkHandler 
6.   MyDataWorkHandler handler1 = **new** MyDataWorkHandler("1"); 
7.   MyDataWorkHandler handler2 = **new** MyDataWorkHandler("2"); 
8.   MyDataWorkHandler handler3 = **new** MyDataWorkHandler("3"); 
9.   WorkerPool<MyDataEvent> workerPool =  
10. ​      **new** WorkerPool<MyDataEvent>(ringBuffer, ringBuffer.newBarrier(),  
11. ​          **new** IgnoreExceptionHandler(),  
12. ​          handler1,handler2,handler3); 
13.   //将WorkPool的工作序列集设置为ringBuffer的追踪序列。 
14.   ringBuffer.addGatingSequences(workerPool.getWorkerSequences()); 
15.   //创建一个线程池用于执行Workhandler。 
16.   Executor executor = Executors.newFixedThreadPool(4); 
17.   //启动WorkPool。 
18.   workerPool.start(executor); 
19.   //往RingBuffer上发布事件 
20.   **for**(**int** i=0;i<10;i++){ 
21. ​    ringBuffer.publishEvent(**new** MyDataEventTranslatorWithIdAndValue(), i, i+"s"); 
22. ​    System.out.println("发布事件["+i+"]"); 
23.   } 
24.   **try** { 
25. ​    System.in.read(); 
26.   } **catch** (IOException e) { 
27. ​    e.printStackTrace(); 
28.   } 
29. } 

​    输出如下： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. 发布事件[0] 
2. WorkHandler[1]处理事件MyData [id=0, value=0s] 
3. 发布事件[1] 
4. 发布事件[2] 
5. WorkHandler[1]处理事件MyData [id=3, value=3s] 
6. 发布事件[3] 
7. WorkHandler[2]处理事件MyData [id=2, value=2s] 
8. WorkHandler[3]处理事件MyData [id=1, value=1s] 
9. 发布事件[4] 
10. WorkHandler[1]处理事件MyData [id=4, value=4s] 
11. 发布事件[5] 
12. WorkHandler[2]处理事件MyData [id=5, value=5s] 
13. WorkHandler[3]处理事件MyData [id=6, value=6s] 
14. 发布事件[6] 
15. 发布事件[7] 
16. WorkHandler[1]处理事件MyData [id=7, value=7s] 
17. 发布事件[8] 
18. 发布事件[9] 
19. WorkHandler[2]处理事件MyData [id=8, value=8s] 
20. WorkHandler[3]处理事件MyData [id=9, value=9s] 

​    正是我们想要的效果，多个处理器处理同一批事件！

 

- **事件处理者如何等待事件发布者发布事件：**

​    上面的分析中已经提到了事件发布者会使用序列栅栏来等待(waitFor)事件发布者发布事件，但具体是怎么等待呢？是阻塞还是自旋？接下来仔细看下这部分，Disruptor框架对这部分提供了很多策略。

​    首先我们看下框架中提供的SequenceBarrier的实现-ProcessingSequenceBarrier类的waitFor细节：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** ProcessingSequenceBarrier(**final** Sequencer sequencer, 
2. ​                 **final** WaitStrategy waitStrategy, 
3. ​                 **final** Sequence cursorSequence, 
4. ​                 **final** Sequence[] dependentSequences){ 
5.   **this**.sequencer = sequencer; 
6.   **this**.waitStrategy = waitStrategy; 
7.   **this**.cursorSequence = cursorSequence; 
8.   **if** (0 == dependentSequences.length){ 
9. ​    dependentSequence = cursorSequence; 
10.   } 
11.   **else**{ 
12. ​    dependentSequence = **new** FixedSequenceGroup(dependentSequences); 
13.   } 
14. } 
15. @Override 
16. **public** **long** waitFor(**final** **long** sequence) 
17.   **throws** AlertException, InterruptedException, TimeoutException{ 
18.   checkAlert(); 
19.   **long** availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, **this**); 
20.   **if** (availableSequence < sequence){ 
21. ​    **return** availableSequence; 
22.   } 
23.   **return** sequencer.getHighestPublishedSequence(sequence, availableSequence); 
24. } 

​    我们注意到waitFor方法中其实是调用了WaitStrategy的waitFor方法来实现等待，WaitStrategy接口是框架中定义的等待策略接口(其实在RingBuffer构造的可以传入一个等待策略，没显式传入的话，会使用默认的等待策略，可以查看下RingBuffer的构造方法)，先看下这个接口： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** WaitStrategy{ 
2.   /** 
3.    \* 等待给定的sequence变为可用。 
4.    \* 
5.    \* @param sequence 等待(申请)的序列值。 
6.    \* @param cursor ringBuffer中的主序列，也可以认为是事件发布者使用的序列。 
7.    \* @param dependentSequence 事件处理者使用的序列。 
8.    \* @param barrier 序列栅栏。 
9.    \* @return 对事件处理者来说可用的序列值，可能会比申请的序列值大。 
10.    \* @throws AlertException if the status of the Disruptor has changed. 
11.    \* @throws InterruptedException if the thread is interrupted. 
12.    \* @throws TimeoutException 
13.    */ 
14.   **long** waitFor(**long** sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier) 
15. ​    **throws** AlertException, InterruptedException, TimeoutException; 
16.   /** 
17.    \* 当发布事件成功后会调用这个方法来通知等待的事件处理者序列可用了。 
18.    */ 
19.   **void** signalAllWhenBlocking(); 
20. } 

​    **Disruptor框架中提供了如下几种等待策略：**

​       *BlockingWaitStrategy*

​       *BusySpinWaitStrategy*

​       *LiteBlockingWaitStrategy*

​       *SleepingWaitStrategy*

​       *TimeoutBlockingWaitStrategy*

​       *YieldingWaitStrategy*

​       *PhasedBackoffWaitStrategy*

 

​    首先看下BlockingWaitStrategy，也是默认的等待序列： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** BlockingWaitStrategy **implements** WaitStrategy{ 
2.   **private** **final** Lock lock = **new** ReentrantLock(); 
3.   **private** **final** Condition processorNotifyCondition = lock.newCondition(); 
4.   @Override 
5.   **public** **long** waitFor(**long** sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier) **throws** AlertException, InterruptedException{ 
6.  
7. ​    **long** availableSequence; 
8. ​    **if** ((availableSequence = cursorSequence.get()) < sequence){ 
9. ​      //如果RingBuffer上当前可用的序列值小于要申请的序列值。 
10. ​      lock.lock(); 
11. ​      **try**{ 
12. ​        //再次检测 
13. ​        **while** ((availableSequence = cursorSequence.get()) < sequence){ 
14. ​          //检查序列栅栏状态(事件处理器是否被关闭) 
15. ​          barrier.checkAlert(); 
16. ​          //当前线程在processorNotifyCondition条件上等待。 
17. ​          processorNotifyCondition.await(); 
18. ​        } 
19. ​      }**finally**{ 
20. ​        lock.unlock(); 
21. ​      } 
22. ​    } 
23. ​    //再次检测，避免事件处理器关闭的情况。 
24. ​    **while** ((availableSequence = dependentSequence.get()) < sequence){ 
25. ​      barrier.checkAlert(); 
26. ​    } 
27. ​    **return** availableSequence; 
28.   } 
29.   @Override 
30.   **public** **void** signalAllWhenBlocking(){ 
31. ​    lock.lock(); 
32. ​    **try**{ 
33. ​      //唤醒在processorNotifyCondition条件上等待的处理事件线程。 
34. ​      processorNotifyCondition.signalAll(); 
35. ​    }**finally**{ 
36. ​      lock.unlock(); 
37. ​    } 
38.   } 
39. } 

​    可见BlockingWaitStrategy的实现方法是阻塞等待。当要求节省CPU资源，而不要求高吞吐量和低延迟的时候使用这个策略。

 

​    再看下BusySpinWaitStrategy： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** BusySpinWaitStrategy **implements** WaitStrategy{ 
2.   @Override 
3.   **public** **long** waitFor(**final** **long** sequence, Sequence cursor, **final** Sequence dependentSequence, **final** SequenceBarrier barrier)**throws** AlertException, InterruptedException{ 
4.  
5. ​    **long** availableSequence; 
6. ​    **while** ((availableSequence = dependentSequence.get()) < sequence){ 
7. ​      barrier.checkAlert(); 
8. ​      //自旋 
9. ​    } 
10. ​    **return** availableSequence; 
11.   } 
12.   @Override 
13.   **public** **void** signalAllWhenBlocking(){ 
14.   } 
15. } 

​    BusySpinWaitStrategy的实现方法是自旋等待。这种策略会利用CPU资源来避免系统调用带来的延迟抖动，当线程可以绑定到指定CPU(核)的时候可以使用这个策略。

 

​    再看下LiteBlockingWaitStrategy： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** LiteBlockingWaitStrategy **implements** WaitStrategy{ 
2.   **private** **final** Lock lock = **new** ReentrantLock(); 
3.   **private** **final** Condition processorNotifyCondition = lock.newCondition(); 
4.   **private** **final** AtomicBoolean signalNeeded = **new** AtomicBoolean(**false**); 
5.   @Override 
6.   **public** **long** waitFor(**long** sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier) **throws** AlertException, InterruptedException{ 
7. ​    **long** availableSequence; 
8. ​    **if** ((availableSequence = cursorSequence.get()) < sequence) { 
9. ​      lock.lock(); 
10. ​      **try**{ 
11. ​        **do**{ 
12. ​          signalNeeded.getAndSet(**true**); 
13. ​          **if** ((availableSequence = cursorSequence.get()) >= sequence) { 
14. ​            **break**; 
15. ​          } 
16. ​          barrier.checkAlert(); 
17. ​          processorNotifyCondition.await(); 
18. ​        } **while** ((availableSequence = cursorSequence.get()) < sequence); 
19. ​      }**finally**{ 
20. ​        lock.unlock(); 
21. ​      } 
22. ​    } 
23. ​    **while** ((availableSequence = dependentSequence.get()) < sequence){ 
24. ​      barrier.checkAlert(); 
25. ​    } 
26. ​    **return** availableSequence; 
27.   } 
28.   @Override 
29.   **public** **void** signalAllWhenBlocking(){ 
30. ​    **if** (signalNeeded.getAndSet(**false**)){ 
31. ​      lock.lock(); 
32. ​      **try**{ 
33. ​        processorNotifyCondition.signalAll(); 
34. ​      }**finally**{ 
35. ​        lock.unlock(); 
36. ​      } 
37. ​    } 
38.   } 
39. } 

​    相比BlockingWaitStrategy，LiteBlockingWaitStrategy的实现方法也是阻塞等待，但它会减少一些不必要的唤醒。从源码的注释上看，这个策略在基准性能测试上是会表现出一些性能提升，但是作者还不能完全证明程序的正确性。

 

​    再看下SleepingWaitStrategy：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** SleepingWaitStrategy **implements** WaitStrategy{ 
2.  
3.   **private** **static** **final** **int** DEFAULT_RETRIES = 200; 
4.   **private** **final** **int** retries; 
5.   **public** SleepingWaitStrategy(){ 
6. ​    **this**(DEFAULT_RETRIES); 
7.   } 
8.   **public** SleepingWaitStrategy(**int** retries){ 
9. ​    **this**.retries = retries; 
10.   } 
11.   @Override 
12.   **public** **long** waitFor(**final** **long** sequence, Sequence cursor, **final** Sequence dependentSequence, **final** SequenceBarrier barrier) **throws** AlertException, InterruptedException{ 
13.  
14. ​    **long** availableSequence; 
15. ​    **int** counter = retries; 
16. ​    **while** ((availableSequence = dependentSequence.get()) < sequence){ 
17. ​      counter = applyWaitMethod(barrier, counter); 
18. ​    } 
19. ​    **return** availableSequence; 
20.   } 
21.   @Override 
22.   **public** **void** signalAllWhenBlocking(){ 
23.   } 
24.   **private** **int** applyWaitMethod(**final** SequenceBarrier barrier, **int** counter) 
25. ​    **throws** AlertException{ 
26. ​    barrier.checkAlert(); 
27. ​    //从指定的重试次数(默认是200)重试到剩下100次，这个过程是自旋。 
28. ​    **if** (counter > 100){ 
29. ​      --counter; 
30. ​    } 
31. ​    //然后尝试100次让出处理器动作。 
32. ​    **else** **if** (counter > 0){ 
33. ​      --counter; 
34. ​      Thread.yield(); 
35. ​    } 
36. ​    //然后尝试阻塞1纳秒。 
37. ​    **else**{ 
38. ​      LockSupport.parkNanos(1L); 
39. ​    } 
40. ​    **return** counter; 
41.   } 
42. } 

​    SleepingWaitStrategy的实现方法是先自旋，不行再临时让出调度(yield)，不行再短暂的阻塞等待。对于既想取得高性能，由不想太浪费CPU资源的场景，这个策略是一种比较好的折中方案。使用这个方案可能会出现延迟波动。

 

​    再看下TimeoutBlockingWaitStrategy： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** TimeoutBlockingWaitStrategy **implements** WaitStrategy{ 
2.   **private** **final** Lock lock = **new** ReentrantLock(); 
3.   **private** **final** Condition processorNotifyCondition = lock.newCondition(); 
4.   **private** **final** **long** timeoutInNanos; 
5.   **public** TimeoutBlockingWaitStrategy(**final** **long** timeout, **final** TimeUnit units){ 
6. ​    timeoutInNanos = units.toNanos(timeout); 
7.   } 
8.   @Override 
9.   **public** **long** waitFor(**final** **long** sequence, 
10. ​            **final** Sequence cursorSequence, 
11. ​            **final** Sequence dependentSequence, 
12. ​            **final** SequenceBarrier barrier) 
13. ​    **throws** AlertException, InterruptedException, TimeoutException{ 
14. ​    **long** nanos = timeoutInNanos; 
15. ​    **long** availableSequence; 
16. ​    **if** ((availableSequence = cursorSequence.get()) < sequence){ 
17. ​      lock.lock(); 
18. ​      **try**{ 
19. ​        **while** ((availableSequence = cursorSequence.get()) < sequence){ 
20. ​          barrier.checkAlert(); 
21. ​          nanos = processorNotifyCondition.awaitNanos(nanos); 
22. ​          **if** (nanos <= 0){ 
23. ​            **throw** TimeoutException.INSTANCE; 
24. ​          } 
25. ​        } 
26. ​      }**finally**{ 
27. ​        lock.unlock(); 
28. ​      } 
29. ​    } 
30. ​    **while** ((availableSequence = dependentSequence.get()) < sequence){ 
31. ​      barrier.checkAlert(); 
32. ​    } 
33. ​    **return** availableSequence; 
34.   } 
35.   @Override 
36.   **public** **void** signalAllWhenBlocking(){ 
37. ​    lock.lock(); 
38. ​    **try**{ 
39. ​      processorNotifyCondition.signalAll(); 
40. ​    }**finally**{ 
41. ​      lock.unlock(); 
42. ​    } 
43.   } 
44. } 

​    TimeoutBlockingWaitStrategy的实现方法是阻塞给定的时间，超过时间的话会抛出超时异常。

 

​    再看下YieldingWaitStrategy：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** YieldingWaitStrategy **implements** WaitStrategy{ 
2.   **private** **static** **final** **int** SPIN_TRIES = 100; 
3.   @Override 
4.   **public** **long** waitFor(**final** **long** sequence, Sequence cursor, **final** Sequence dependentSequence, **final** SequenceBarrier barrier) **throws** AlertException, InterruptedException{ 
5. ​    **long** availableSequence; 
6. ​    **int** counter = SPIN_TRIES; 
7. ​    **while** ((availableSequence = dependentSequence.get()) < sequence){ 
8. ​      counter = applyWaitMethod(barrier, counter); 
9. ​    } 
10. ​    **return** availableSequence; 
11.   } 
12.   @Override 
13.   **public** **void** signalAllWhenBlocking(){ 
14.   } 
15.   **private** **int** applyWaitMethod(**final** SequenceBarrier barrier, **int** counter) 
16. ​    **throws** AlertException{ 
17. ​    barrier.checkAlert(); 
18. ​    **if** (0 == counter){ 
19. ​      Thread.yield(); 
20. ​    }**else**{ 
21. ​      --counter; 
22. ​    } 
23. ​    **return** counter; 
24.   } 
25. } 

​    SleepingWaitStrategy的实现方法是先自旋(100次)，不行再临时让出调度(yield)。和SleepingWaitStrategy一样也是一种高性能与CPU资源之间取舍的折中方案，但这个策略不会带来显著的延迟抖动。

 

​    最后看下PhasedBackoffWaitStrategy： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** PhasedBackoffWaitStrategy **implements** WaitStrategy{ 
2.   **private** **static** **final** **int** SPIN_TRIES = 10000; 
3.   **private** **final** **long** spinTimeoutNanos; 
4.   **private** **final** **long** yieldTimeoutNanos; 
5.   **private** **final** WaitStrategy fallbackStrategy; 
6.   **public** PhasedBackoffWaitStrategy(**long** spinTimeout, 
7. ​                   **long** yieldTimeout, 
8. ​                   TimeUnit units, 
9. ​                   WaitStrategy fallbackStrategy){ 
10. ​    **this**.spinTimeoutNanos = units.toNanos(spinTimeout); 
11. ​    **this**.yieldTimeoutNanos = spinTimeoutNanos + units.toNanos(yieldTimeout); 
12. ​    **this**.fallbackStrategy = fallbackStrategy; 
13.   } 
14.  
15.   **public** **static** PhasedBackoffWaitStrategy withLock(**long** spinTimeout, 
16. ​                           **long** yieldTimeout, 
17. ​                           TimeUnit units){ 
18. ​    **return** **new** PhasedBackoffWaitStrategy(spinTimeout, yieldTimeout, 
19. ​                       units, **new** BlockingWaitStrategy()); 
20.   } 
21.  
22.   **public** **static** PhasedBackoffWaitStrategy withLiteLock(**long** spinTimeout, 
23. ​                             **long** yieldTimeout, 
24. ​                             TimeUnit units){ 
25. ​    **return** **new** PhasedBackoffWaitStrategy(spinTimeout, yieldTimeout, 
26. ​                       units, **new** LiteBlockingWaitStrategy()); 
27.   } 
28.  
29.   **public** **static** PhasedBackoffWaitStrategy withSleep(**long** spinTimeout, 
30. ​                           **long** yieldTimeout, 
31. ​                           TimeUnit units){ 
32. ​    **return** **new** PhasedBackoffWaitStrategy(spinTimeout, yieldTimeout, 
33. ​                       units, **new** SleepingWaitStrategy(0)); 
34.   } 
35.   @Override 
36.   **public** **long** waitFor(**long** sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier) **throws** AlertException, InterruptedException, TimeoutException{ 
37. ​    **long** availableSequence; 
38. ​    **long** startTime = 0; 
39. ​    **int** counter = SPIN_TRIES; 
40. ​    **do**{ 
41. ​      **if** ((availableSequence = dependentSequence.get()) >= sequence){ 
42. ​        **return** availableSequence; 
43. ​      } 
44. ​      **if** (0 == --counter){ 
45. ​        **if** (0 == startTime){ 
46. ​          startTime = System.nanoTime(); 
47. ​        }**else**{ 
48. ​          **long** timeDelta = System.nanoTime() - startTime; 
49. ​          **if** (timeDelta > yieldTimeoutNanos){ 
50. ​            **return** fallbackStrategy.waitFor(sequence, cursor, dependentSequence, barrier); 
51. ​          }**else** **if** (timeDelta > spinTimeoutNanos){ 
52. ​            Thread.yield(); 
53. ​          } 
54. ​        } 
55. ​        counter = SPIN_TRIES; 
56. ​      } 
57. ​    }**while** (**true**); 
58.   } 
59.   @Override 
60.   **public** **void** signalAllWhenBlocking(){ 
61. ​    fallbackStrategy.signalAllWhenBlocking(); 
62.   } 
63. } 

​    PhasedBackoffWaitStrategy的实现方法是先自旋(10000次)，不行再临时让出调度(yield)，不行再使用其他的策略进行等待。可以根据具体场景自行设置自旋时间、yield时间和备用等待策略。

 

 

- **最后总结：**

​    **1.事件处理者可以通过Event模式或者Work模式来处理事件。**

​    **2.事件处理者可以使用多种等待策略来等待事件发布者发布事件，可按照具体场景选择合适的等待策略。**  