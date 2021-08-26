## disruptor-3.3.2源码解析(3)-发布事件

作者：大飞

 

- **Disruptor中如何发布事件：**

​    前面两篇看了disruptor中的序列和队列，这篇说一下怎么往RingBuffer中发布事件。这里也需要明确一下，和一般的生产者/消费者模式不同(如果以生产者/消费者的模式来看待disruptor的话)，disruptor中队列里面的数据一般称为事件，RingBuffer中提供了发布事件的方法，另外也提供了专门的处理事件的类。

​    其实在disruptor中，RingBuffer也提供了一部分生产的功能，里面提供了大量的发布事件的方法。

​    上篇看到的RingBuffer的构造方法，需要传一个EventFactory做事件的预填充：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. RingBuffer(EventFactory<E> eventFactory, 
2. ​      Sequencer    sequencer){ 
3.   **super**(eventFactory, sequencer); 
4. } 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. RingBufferFields(EventFactory<E> eventFactory, 
2. ​         Sequencer    sequencer){ 
3.   ... 
4.   //最后要填充事件 
5.   fill(eventFactory); 
6. } 
7. **private** **void** fill(EventFactory<E> eventFactory){ 
8.   **for** (**int** i = 0; i < bufferSize; i++){ 
9. ​    entries[BUFFER_PAD + i] = eventFactory.newInstance(); 
10.   } 
11. } 

​    看下EventFactory： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** EventFactory<T>{ 
2.   /* 
3.    \* Implementations should instantiate an event object, with all memory already allocated where possible. 
4.    */ 
5.   T newInstance(); 
6. } 

​    再看下RingBuffer的发布事件方法：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **void** publishEvent(EventTranslator<E> translator){ 
3.   **final** **long** sequence = sequencer.next(); 
4.   translateAndPublish(translator, sequence); 
5. } 
6. **private** **void** translateAndPublish(EventTranslatorVararg<E> translator, **long** sequence){ 
7.   **try**{ 
8. ​    translator.translateTo(get(sequence), sequence); 
9.   }**finally**{ 
10. ​    sequencer.publish(sequence); 
11.   } 
12. } 

​    在发布事件时需要传一个事件转换的接口，内部用这个接口做一下数据到事件的转换。看下这个接口：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** EventTranslator<T>{ 
2.   /** 
3.    \* Translate a data representation into fields set in given event 
4.    \* 
5.    \* @param event into which the data should be translated. 
6.    \* @param sequence that is assigned to event. 
7.    */ 
8.   **void** translateTo(**final** T event, **long** sequence); 
9. } 

​    可见，具体的生产者可以实现这个接口，将需要发布的数据放到这个事件里面，一般是设置到事件的某个域上。

 

​    好，来看个例子理解一下。

​    首先我们定义好数据： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyData { 
2.   **private** **int** id; 
3.   **private** String value; 
4.   **public** MyData(**int** id, String value) { 
5. ​    **this**.id = id; 
6. ​    **this**.value = value; 
7.   } 
8.  
9.   ...getter setter... 
10.  
11.   @Override 
12.   **public** String toString() { 
13. ​    **return** "MyData [id=" + id + ", value=" + value + "]"; 
14.   } 
15.    
16. } 

​    然后针对我们的数据定义事件：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyDataEvent { 
2.   **private** MyData data; 
3.   **public** MyData getData() { 
4. ​    **return** data; 
5.   } 
6.   **public** **void** setData(MyData data) { 
7. ​    **this**.data = data; 
8.   } 
9.    
10. } 

​    接下来需要给出一个EventFactory提供给RingBuffer做事件预填充：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyDataEventFactory **implements** EventFactory<MyDataEvent>{ 
2.   @Override 
3.   **public** MyDataEvent newInstance() { 
4. ​    **return** **new** MyDataEvent(); 
5.   } 
6. } 

​    好了，可以初始化RingBuffer了：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1.   **public** **static** **void** main(String[] args) { 
2.   RingBuffer<MyDataEvent> ringBuffer =  
3. ​      RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 1024); 
4.   MyDataEvent dataEvent = ringBuffer.get(0); 
5.   System.out.println("Event = " + dataEvent); 
6.   System.out.println("Data = " + dataEvent.getData()); 
7. } 

​    输出如下：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. Event = com.mjf.disruptor.product.MyDataEvent@5c647e05 
2. Data = **null** 

​    首先要注意，RingBuffer里面是MydataEvent，而不是MyData；其次我们构造好了RingBuffer，里面就已经填充了事件，我们可以取一个事件出来，发现里面的数据是空的。

 

​    下面就是怎么往RingBuffer里面放数据了，也就是事件发布者要干活了。我们上面看到了，要使用RingBuffer发布一个事件，需要一个事件转换器接口，针对我们的数据实现一个： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyDataEventTranslator **implements** EventTranslator<MyDataEvent>{ 
2.   @Override 
3.   **public** **void** translateTo(MyDataEvent event, **long** sequence) { 
4. ​    //新建一个数据 
5. ​    MyData data = **new** MyData(1, "holy shit!"); 
6. ​    //将数据放入事件中。 
7. ​    event.setData(data); 
8.   } 
9. } 

​    有了转换器，我们就可以嗨皮的发布事件了：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   RingBuffer<MyDataEvent> ringBuffer =  
3. ​      RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 1024); 
4.   //发布事件!!! 
5.   ringBuffer.publishEvent(**new** MyDataEventTranslator()); 
6.    
7.   MyDataEvent dataEvent0 = ringBuffer.get(0); 
8.   System.out.println("Event = " + dataEvent0); 
9.   System.out.println("Data = " + dataEvent0.getData()); 
10.   MyDataEvent dataEvent1 = ringBuffer.get(1); 
11.   System.out.println("Event = " + dataEvent1); 
12.   System.out.println("Data = " + dataEvent1.getData()); 
13. } 

​    输出如下：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. Event = com.mjf.disruptor.product.MyDataEvent@5c647e05 
2. Data = MyData [id=1, value=holy shit!] 
3. Event = com.mjf.disruptor.product.MyDataEvent@33909752 
4. Data = **null** 

​    可见，我们已经成功了发布了一个事件到RingBuffer，由于是从序列0开始发布，所以我们从序列0可以读出这个数据。因为只发布了一个，所以序列1上还是没有数据。

 

​    当然也有其他姿势的转换器： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **class** MyDataEventTranslatorWithIdAndValue **implements** EventTranslatorTwoArg<MyDataEvent, Integer, String>{ 
2.   @Override 
3.   **public** **void** translateTo(MyDataEvent event, **long** sequence, Integer id, 
4. ​      String value) { 
5. ​    MyData data = **new** MyData(id, value); 
6. ​    event.setData(data); 
7.   } 
8. } 

​    当然也可以直接利用RingBuffer来发布事件，不需要转换器：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   RingBuffer<MyDataEvent> ringBuffer =  
3. ​      RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 1024); 
4.   **long** sequence = ringBuffer.next(); 
5.   **try**{ 
6. ​    MyDataEvent event = ringBuffer.get(sequence); 
7. ​    MyData data = **new** MyData(2, "R u kidding me?"); 
8. ​    event.setData(data); 
9.   }**finally**{ 
10. ​    ringBuffer.publish(sequence); 
11.   } 
12. } 

 

- **单线程发布事件和多线程发布事件：**

​    前面我们构造RingBuffer使用的是单线程发布事件的模式：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. RingBuffer<MyDataEvent> ringBuffer =  
2. ​    RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 1024); 

​    RingBuffer也支持多线程发布事件模式，还记得上一篇分析的RingBuffer代码吧： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** <E> RingBuffer<E> createMultiProducer(EventFactory<E> factory, **int** bufferSize){ 
2.   **return** createMultiProducer(factory, bufferSize, **new** BlockingWaitStrategy()); 
3. } 

​    当然也提供了比较全面的构造方法：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** <E> RingBuffer<E> create(ProducerType  producerType, 
2. ​                    EventFactory<E> factory, 
3. ​                    **int**       bufferSize, 
4. ​                    WaitStrategy  waitStrategy){ 
5.   **switch** (producerType){ 
6.   **case** SINGLE: 
7. ​    **return** createSingleProducer(factory, bufferSize, waitStrategy); 
8.   **case** MULTI: 
9. ​    **return** createMultiProducer(factory, bufferSize, waitStrategy); 
10.   **default**: 
11. ​    **throw** **new** IllegalStateException(producerType.toString()); 
12.   } 
13. } 

​    这个方法支持传入一个枚举来选择使用哪种模式： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **enum** ProducerType{ 
2.   /** Create a RingBuffer with a single event publisher to the RingBuffer */ 
3.   SINGLE, 
4.   /** Create a RingBuffer supporting multiple event publishers to the one RingBuffer */ 
5.   MULTI 
6. } 

​    上面看过了单线程发布事件的例子，接下来看个多线程发布事件的：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   **final** RingBuffer<MyDataEvent> ringBuffer =  
3. ​      RingBuffer.createMultiProducer(**new** MyDataEventFactory(), 1024); 
4.   **final** CountDownLatch latch = **new** CountDownLatch(100); 
5.   **for**(**int** i=0;i<100;i++){ 
6. ​    **final** **int** index = i; 
7. ​    //开启多个线程发布事件。 
8. ​    **new** Thread(**new** Runnable() { 
9. ​      @Override 
10. ​      **public** **void** run() { 
11. ​        **long** sequence = ringBuffer.next(); 
12. ​        **try**{ 
13. ​          MyDataEvent event = ringBuffer.get(sequence); 
14. ​          MyData data = **new** MyData(index, index+"s"); 
15. ​          event.setData(data); 
16. ​        }**finally**{ 
17. ​          ringBuffer.publish(sequence); 
18. ​          latch.countDown(); 
19. ​        } 
20. ​      } 
21. ​    }).start(); 
22.   } 
23.   **try** { 
24. ​    latch.await(); 
25. ​    //最后观察下发布的时间。 
26. ​    **for**(**int** i=0;i<100;i++){ 
27. ​      MyDataEvent event = ringBuffer.get(i); 
28. ​      System.out.println(event.getData()); 
29. ​    } 
30.   } **catch** (InterruptedException e) { 
31. ​    e.printStackTrace(); 
32.   } 
33. } 

 

​    如果多线程环境下使用单线程发布模式会有上面问题呢？

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** **void** main(String[] args) { 
2.   **final** RingBuffer<MyDataEvent> ringBuffer =  
3. ​            //这里是单线程模式!!! 
4. ​      RingBuffer.createSingleProducer(**new** MyDataEventFactory(), 1024); 
5.   **final** CountDownLatch latch = **new** CountDownLatch(100); 
6.   **for**(**int** i=0;i<100;i++){ 
7. ​    **final** **int** index = i; 
8. ​    //开启多个线程发布事件。 
9. ​    **new** Thread(**new** Runnable() { 
10. ​      @Override 
11. ​      **public** **void** run() { 
12. ​        **long** sequence = ringBuffer.next(); 
13. ​        **try**{ 
14. ​          MyDataEvent event = ringBuffer.get(sequence); 
15. ​          MyData data = **new** MyData(index, index+"s"); 
16. ​          event.setData(data); 
17. ​        }**finally**{ 
18. ​          ringBuffer.publish(sequence); 
19. ​          latch.countDown(); 
20. ​        } 
21. ​      } 
22. ​    }).start(); 
23.   } 
24.   **try** { 
25. ​    latch.await(); 
26. ​    //最后观察下发布的时间。 
27. ​    **for**(**int** i=0;i<100;i++){ 
28. ​      MyDataEvent event = ringBuffer.get(i); 
29. ​      System.out.println(event.getData()); 
30. ​    } 
31.   } **catch** (InterruptedException e) { 
32. ​    e.printStackTrace(); 
33.   } 
34. } 

​    输出如下： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. ... 
2. MyData [id=92, value=92s] 
3. MyData [id=93, value=93s] 
4. MyData [id=94, value=94s] 
5. MyData [id=95, value=95s] 
6. MyData [id=96, value=96s] 
7. MyData [id=97, value=97s] 
8. MyData [id=99, value=99s] 
9. MyData [id=98, value=98s] 
10. **null** 
11. **null** 

​    会发现，如果多线程发布事件的环境下，使用单线程发布事件模式，会有数据被覆盖的情况。所以使用时应该按照具体情况选择合理发布模式。

 

- **最后总结：**

​    **如何往RingBuffer中发布事件：**

​       **1.定义好要生产的数据和相应的事件类(里面存放数据)。**

​       **2.定于好事件转换器或者直接用RingBuffer进行事件发布。**

​       **3.明确发布场景，合理的选择发布模式(单线程还是多线程)。** 