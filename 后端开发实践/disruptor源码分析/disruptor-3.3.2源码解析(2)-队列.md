## disruptor-3.3.2源码解析(2)-队列

作者：大飞

 

- **Disruptor中的队列-RingBuffer：**

​    RingBuffer是disruptor最重要的核心组件，如果以生产者/消费者模式来看待disruptor框架的话，那RingBuffer就是生产者和消费者的工作队列了。RingBuffer可以理解为是一个环形队列，那内部是怎么实现的呢？看下源码。

 

​    首先，RingBuffer实现了一系列接口，Cursored、EventSequencer和EventSink，Cursored上篇提过了，这里看下后面两个：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** EventSequencer<T> **extends** DataProvider<T>, Sequenced{ 
2. } 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** DataProvider<T>{ 
2.   T get(**long** sequence); 
3. } 

​    EventSequencer扩展了Sequenced，提供了一些序列功能；同时扩展了DataProvider，提供了按序列值来获取数据的功能。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** EventSink<E>{ 
2.   **void** publishEvent(EventTranslator<E> translator); 
3.  
4.   **boolean** tryPublishEvent(EventTranslator<E> translator); 
5.  
6.   <A> **void** publishEvent(EventTranslatorOneArg<E, A> translator, A arg0); 
7.  
8.   <A> **boolean** tryPublishEvent(EventTranslatorOneArg<E, A> translator, A arg0); 
9.  
10.   <A, B> **void** publishEvent(EventTranslatorTwoArg<E, A, B> translator, A arg0, B arg1); 
11.  
12.   <A, B> **boolean** tryPublishEvent(EventTranslatorTwoArg<E, A, B> translator, A arg0, B arg1); 
13.  
14.   <A, B, C> **void** publishEvent(EventTranslatorThreeArg<E, A, B, C> translator, A arg0, B arg1, C arg2); 
15.  
16.   <A, B, C> **boolean** tryPublishEvent(EventTranslatorThreeArg<E, A, B, C> translator, A arg0, B arg1, C arg2); 
17.  
18.   **void** publishEvent(EventTranslatorVararg<E> translator, Object... args); 
19.  
20.   **boolean** tryPublishEvent(EventTranslatorVararg<E> translator, Object... args); 
21.  
22.   **void** publishEvents(EventTranslator<E>[] translators); 
23.  
24.   **void** publishEvents(EventTranslator<E>[] translators, **int** batchStartsAt, **int** batchSize); 
25.  
26.   **boolean** tryPublishEvents(EventTranslator<E>[] translators); 
27.  
28.   **boolean** tryPublishEvents(EventTranslator<E>[] translators, **int** batchStartsAt, **int** batchSize); 
29.  
30.   <A> **void** publishEvents(EventTranslatorOneArg<E, A> translator, A[] arg0); 
31.  
32.   <A> **void** publishEvents(EventTranslatorOneArg<E, A> translator, **int** batchStartsAt, **int** batchSize, A[] arg0); 
33.  
34.   <A> **boolean** tryPublishEvents(EventTranslatorOneArg<E, A> translator, A[] arg0); 
35.  
36.   <A> **boolean** tryPublishEvents(EventTranslatorOneArg<E, A> translator, **int** batchStartsAt, **int** batchSize, A[] arg0); 
37.  
38.   <A, B> **void** publishEvents(EventTranslatorTwoArg<E, A, B> translator, A[] arg0, B[] arg1); 
39.  
40.   <A, B> **void** publishEvents(EventTranslatorTwoArg<E, A, B> translator, **int** batchStartsAt, **int** batchSize, A[] arg0, B[] arg1); 
41.  
42.   <A, B> **boolean** tryPublishEvents(EventTranslatorTwoArg<E, A, B> translator, A[] arg0, B[] arg1); 
43.  
44.   <A, B> **boolean** tryPublishEvents(EventTranslatorTwoArg<E, A, B> translator, **int** batchStartsAt, **int** batchSize, A[] arg0, B[] arg1); 
45.  
46.   <A, B, C> **void** publishEvents(EventTranslatorThreeArg<E, A, B, C> translator, A[] arg0, B[] arg1, C[] arg2); 
47.  
48.   <A, B, C> **void** publishEvents(EventTranslatorThreeArg<E, A, B, C> translator, **int** batchStartsAt, **int** batchSize, A[] arg0, B[] arg1, C[] arg2); 
49.  
50.   <A, B, C> **boolean** tryPublishEvents(EventTranslatorThreeArg<E, A, B, C> translator, A[] arg0, B[] arg1, C[] arg2); 
51.  
52.   <A, B, C> **boolean** tryPublishEvents(EventTranslatorThreeArg<E, A, B, C> translator, **int** batchStartsAt, **int** batchSize, A[] arg0, B[] arg1, C[] arg2); 
53.  
54.   **void** publishEvents(EventTranslatorVararg<E> translator, Object[]... args); 
55.  
56.   **void** publishEvents(EventTranslatorVararg<E> translator, **int** batchStartsAt, **int** batchSize, Object[]... args); 
57.  
58.   **boolean** tryPublishEvents(EventTranslatorVararg<E> translator, Object[]... args); 
59.  
60.   **boolean** tryPublishEvents(EventTranslatorVararg<E> translator, **int** batchStartsAt, **int** batchSize, Object[]... args); 
61. } 

​    可见，EventSink主要是提供发布事件(就是往队列上放数据)的功能，接口上定义了以各种姿势发布事件的方法。

 

​    了解了RingBuffer的接口功能，下面看下RingBuffer的结构：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **abstract** **class** RingBufferPad{ 
2.   **protected** **long** p1, p2, p3, p4, p5, p6, p7; 
3. } 
4. **abstract** **class** RingBufferFields<E> **extends** RingBufferPad{ 
5.   **private** **static** **final** **int** BUFFER_PAD; 
6.   **private** **static** **final** **long** REF_ARRAY_BASE; 
7.   **private** **static** **final** **int** REF_ELEMENT_SHIFT; 
8.   **private** **static** **final** Unsafe UNSAFE = Util.getUnsafe(); 
9.   **static**{ 
10. ​    **final** **int** scale = UNSAFE.arrayIndexScale(Object[].**class**); 
11. ​    **if** (4 == scale){ 
12. ​      REF_ELEMENT_SHIFT = 2; 
13. ​    } **else** **if** (8 == scale){ 
14. ​      REF_ELEMENT_SHIFT = 3; 
15. ​    }**else**{ 
16. ​      **throw** **new** IllegalStateException("Unknown pointer size"); 
17. ​    } 
18. ​    BUFFER_PAD = 128 / scale; 
19. ​    // Including the buffer pad in the array base offset 
20. ​    REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].**class**) + (BUFFER_PAD << REF_ELEMENT_SHIFT); 
21.   } 
22.   **private** **final** **long** indexMask; 
23.   **private** **final** Object[] entries; 
24.   **protected** **final** **int** bufferSize; 
25.   **protected** **final** Sequencer sequencer; 
26.   RingBufferFields(EventFactory<E> eventFactory, 
27. ​           Sequencer    sequencer){ 
28. ​    **this**.sequencer = sequencer; 
29. ​    **this**.bufferSize = sequencer.getBufferSize(); 
30. ​    **if** (bufferSize < 1){ 
31. ​      **throw** **new** IllegalArgumentException("bufferSize must not be less than 1"); 
32. ​    } 
33. ​    **if** (Integer.bitCount(bufferSize) != 1){ 
34. ​      **throw** **new** IllegalArgumentException("bufferSize must be a power of 2"); 
35. ​    } 
36. ​    **this**.indexMask = bufferSize - 1; 
37. ​    **this**.entries  = **new** Object[sequencer.getBufferSize() + 2 * BUFFER_PAD]; 
38. ​    fill(eventFactory); 
39.   } 
40.   **private** **void** fill(EventFactory<E> eventFactory){ 
41. ​    **for** (**int** i = 0; i < bufferSize; i++){ 
42. ​      entries[BUFFER_PAD + i] = eventFactory.newInstance(); 
43. ​    } 
44.   } 
45.   @SuppressWarnings("unchecked") 
46.   **protected** **final** E elementAt(**long** sequence){ 
47. ​    **return** (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT)); 
48.   } 
49. } 
50.  
51. **public** **final** **class** RingBuffer<E> **extends** RingBufferFields<E> **implements** Cursored, EventSequencer<E>, EventSink<E>{ 
52.   **public** **static** **final** **long** INITIAL_CURSOR_VALUE = Sequence.INITIAL_VALUE; 
53.   **protected** **long** p1, p2, p3, p4, p5, p6, p7; 
54.  
55.   RingBuffer(EventFactory<E> eventFactory, Sequencer sequencer){ 
56. ​    **super**(eventFactory, sequencer); 
57.   } 

​    RingBuffer的内部结构明确了：内部用数组来实现，同时有保存数组长度的域bufferSize和下标掩码indexMask，还有一个sequencer。

​    这里要注意几点：

​       1.整个RingBuffer内部做了大量的缓存行填充，前后各填充了56个字节，entries本身也根据引用大小进行了填充，假设引用大小为4字节，那么entries数组两侧就要个填充32个空数组位。也就是说，实际的数组长度比bufferSize要大。所以可以看到根据序列从entries中取元素的方法elementAt内部做了一些调整，不是单纯的取模。

​       2.bufferSize必须是2的幂，indexMask就是bufferSize-1，这样取模更高效(sequence&indexMask)。

​       3.初始化时需要传入一个EventFactory，用来做队列内事件的预填充。

 

​    ***总结下RingBuffer的特点：内部做了细致的缓存行填充，避免伪共享；内部队列基于数组实现，能很好的利用程序的局部性；队列上的事件槽会不断重复利用，不受JavaGC的影响。***

 

​    从上面的代码看到，RingBuffer对本包外屏蔽了构造方法，那么怎创建RingBuffer实例呢？ 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **static** <E> RingBuffer<E> createMultiProducer(EventFactory<E> factory, 
2. ​                          **int**       bufferSize, 
3. ​                          WaitStrategy  waitStrategy){ 
4.   MultiProducerSequencer sequencer = **new** MultiProducerSequencer(bufferSize, waitStrategy); 
5.   **return** **new** RingBuffer<E>(factory, sequencer); 
6. } 
7.  
8. **public** **static** <E> RingBuffer<E> createMultiProducer(EventFactory<E> factory, **int** bufferSize){ 
9.   **return** createMultiProducer(factory, bufferSize, **new** BlockingWaitStrategy()); 
10. } 
11.  
12. **public** **static** <E> RingBuffer<E> createSingleProducer(EventFactory<E> factory, 
13. ​                           **int**       bufferSize, 
14. ​                           WaitStrategy  waitStrategy){ 
15.   SingleProducerSequencer sequencer = **new** SingleProducerSequencer(bufferSize, waitStrategy); 
16.   **return** **new** RingBuffer<E>(factory, sequencer); 
17. } 
18.  
19. **public** **static** <E> RingBuffer<E> createSingleProducer(EventFactory<E> factory, **int** bufferSize){ 
20.   **return** createSingleProducer(factory, bufferSize, **new** BlockingWaitStrategy()); 
21. } 
22.  
23. **public** **static** <E> RingBuffer<E> create(ProducerType  producerType, 
24. ​                    EventFactory<E> factory, 
25. ​                    **int**       bufferSize, 
26. ​                    WaitStrategy  waitStrategy){ 
27.   **switch** (producerType){ 
28.   **case** SINGLE: 
29. ​    **return** createSingleProducer(factory, bufferSize, waitStrategy); 
30.   **case** MULTI: 
31. ​    **return** createMultiProducer(factory, bufferSize, waitStrategy); 
32.   **default**: 
33. ​    **throw** **new** IllegalStateException(producerType.toString()); 
34.   } 
35. } 

​    可见，RingBuffer提供了静态工厂方法分别针对单事件发布者和多事件发布者的情况进行RingBuffer实例创建。

 

​    RingBuffer方法实现也比较简单，看几个：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** E get(**long** sequence){ 
3.   **return** elementAt(sequence); 
4. } 

​    事件发布者和事件处理者申请到序列后，都会通过这个方法从序列上获取事件槽来生产或者发布事件。

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **long** next(**int** n){ 
3.   **return** sequencer.next(n); 
4. } 

​    next系列的方法是通过内部的sequencer来实现的。  

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **boolean** isPublished(**long** sequence){ 
2.   **return** sequencer.isAvailable(sequence); 
3. } 

​    判断某个序列是否已经被事件发布者发布事件。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **void** addGatingSequences(Sequence... gatingSequences){ 
2.   sequencer.addGatingSequences(gatingSequences); 
3. } 
4.  
5. **public** **long** getMinimumGatingSequence(){ 
6.   **return** sequencer.getMinimumSequence(); 
7. } 
8.  
9. **public** **boolean** removeGatingSequence(Sequence sequence){ 
10.   **return** sequencer.removeGatingSequence(sequence); 
11. } 

​    追踪序列相关的方法。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **void** publishEvent(EventTranslator<E> translator){ 
3.   **final** **long** sequence = sequencer.next(); 
4.   translateAndPublish(translator, sequence); 
5. } 
6. **private** **void** translateAndPublish(EventTranslator<E> translator, **long** sequence){ 
7.   **try**{ 
8. ​    translator.translateTo(get(sequence), sequence); 
9.   }**finally**{ 
10. ​    sequencer.publish(sequence); 
11.   } 
12. } 

​    可见，发布事件分三步：

​       1.申请序列。

​       2.填充事件。

​       3.提交序列。

 

​    其他方法实现都很简单，这里不啰嗦了。

 

- **最后总结：**

​    **有关RingBuffer要记住以下几点：**

​       **1.RingBuffer是协调事件发布者和事件处理者的中间队列，事件发布者发布事件放到RingBuffer，事件处理者从RingBuffer上拿事件进行消费。**

​       **2.RingBuffer可以认为是一个环形队列，底层由数组实现。内部做了大量的缓存行填充，保存事件使用的数组的长度必须是2的幂，这样可以高效的取模(取模本身就包含绕回逻辑，按照序列不断的增长，形成一个环形轨迹)。由于RingBuffer这样的特性，也避免了GC带来的性能影响，因为RingBuffer本身永远不会被GC。**

​       **3.RingBuffer和普通的FIFO队列相比还有一个重要的区别就是，RingBuffer避免了头尾节点的竞争，多个事件发布者/事件处理者之间不必竞争同一个节点，只需要安全申请序列值各自存取事件就好了。**