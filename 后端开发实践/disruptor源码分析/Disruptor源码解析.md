### juc下的队列

![640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Disruptor%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.assets/16613eb8f1daea22tplv-t2oaga2asx-watermark.awebp)![2931e5f7539c97fc7003b853334a3c24c793c6c2](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Disruptor%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.assets/16613eb8f1eba43ftplv-t2oaga2asx-watermark.awebp) 1：从上图可以看出，juc下的队列基本采用加锁方式保证线程安全。通过不加锁的方式实现的队列都是无界的（无法保证队列的长度在限定的范围）。而加锁的方式可以实现有界队列。在稳定性要求特别高的系统中，为了防止生产者速度过快，导致内存溢出，只能选择有界队列。    

2：加锁的方式通常严重影响性能。线程会因为竞争不到锁而被挂起，等锁被释放的时候，线程又会被恢复，这个过程中存在着很大的开销，并且通常会有较长时间的中断，因为当一个线程正在等待锁时，它不能做任何其他事情。如果一个线程在持有锁的情况下被延迟执行，例如发生了缺页错误、调度延迟或者其它类似情况，那么所有需要这个锁的线程都无法执行下去。如果被阻塞线程的优先级较高，而持有锁的线程优先级较低，就会发生优先级反转。

3：有界队列通常采用数组实现。但是采用数组实现又会引发另外一个问题false sharing(伪共享)。关于什么是伪共享之前的文章已经讲解。

### Disruptor

#### Disruptor是什么

1：Disruptor是英国外汇交易公司LMAX开发的一个高性能队列，研发的初衷是解决内存队列的延迟问题（在性能测试中发现竟然与I/O操作处于同样的数量级）

2：Disruptor实现对了队列的功能并且是一个有界队列。可以用于生产者-消费者模型。

#### Disruptor为什么快

1：数据结构采用ringbuffer。其实可以理解成一个数组entries。每一个slot存储一个事件对象。初始化时，就已经分配好内存，而且新发布的数据只会覆盖，所以更少的GC。

2：Disruptor采用缓存行填充机制的形式解决了fasle sharing。保证读取变量的时候从cache line读取。

3：Disroptor中维护了一个long类型的sequence(序列)。每次根据位运算操作可以快速定位到实际slot，sequece&(entries.length-1)=index，比如一共有4槽，9&(8-1)=1。提示：队列的大小必须要2^n。

4：线程同时访问，由于他们都通过sequence访问ringBuffer，通过CAS取代了加锁，这也是并发编程的原则：把同步块最小化到一个变量上。这个sequence一直采用自增的形式。

#### Disruptor核心类

1：RingBuffer：Disruptor最主要的组件，仅仅负责存储和更新事件对象。

2：Sequence：Disruptor使用Sequence来表示一个特殊组件处理的序号。和Disruptor一样，每一个消费者（EventProcessor）都维持着一个Sequence。大部分的并发代码依赖这这个值。这个类维护了一个long类型的value，采用的unsafe进行的更新操作。

3：Sequencer：这是Disruptor真正的核心。实现了这个接口的两种生产者（单生产者和多生产者）均实现了所有的并发算法，为了在生产者和消费者之间进行准确快速的数据传递。

4：SequenceBarrier：由Sequencer生成，并且包含了已经发布的Sequence的引用，这些Sequence源于Sequencer和一些独立的消费者的Sequence。它包含了决定是否有供消费者消费的Event的逻辑。用来权衡当消费者无法从RingBuffer里面获取事件时的处理策略。（例如：当生产者太慢，消费者太快，会导致消费者获取不到新的事件会根据该策略进行处理，默认会堵塞）

5：WaitStrategy：决定一个消费者将如何等待生产者将Event置入Disruptor的策略。用来权衡当生产者无法将新的事件放进RingBuffer时的处理策略。（例如：当生产者太快，消费者太慢，会导致生产者获取不到新的事件槽来插入新事件，则会根据该策略进行处理，默认会堵塞）

6：Event：从生产者到消费者过程中所处理的数据单元。Disruptor中没有代码表示Event，因为它完全是由用户定义的。

7：EventProcessor：主要事件循环，处理Disruptor中的Event，并且拥有消费者的Sequence。它有一个实现类是BatchEventProcessor，包含了event loop有效的实现，并且将回调到一个EventHandler接口的实现对象。

8：EventHandler：由用户实现并且代表了Disruptor中的一个消费者的接口。

9：WorkHandler：在work模式下使用。由用户实现并且代表了Disruptor中的多个消费者的接口。

10：WorkProcessor：确保每个sequence只被一个processor消费，在同一个WorkPool中的处理多个WorkProcessor不会消费同样的sequence。

11：WorkerPool：一个WorkProcessor池，其中WorkProcessor将消费Sequence，所以任务可以在实现WorkHandler接口的worker之间移交

12：LifecycleAware：当BatchEventProcessor启动和停止时，实现这个接口用于接收通知。

### Sequence(序列)

![c0de56f10c93a2a4588852b072303db10ebf7685](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Disruptor%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.assets/16613eb8f1e1885dtplv-t2oaga2asx-watermark.awebp)

1：Sequence是用来标记事件发布者和事件消费者的位置。    

2：Sequence真正计数的是value，采用缓冲行填充防止false sharing。在value的前后各有7个long型的填充值，这些值在这里的作用是做cpu cache line填充，防止发生伪共享。最坏的情况就是value位于cache line的头或者尾。

### 框架类结构关系图

![50f47b9f5a8f33f299288caed64c374b406e026c](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Disruptor%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.assets/16613eb8f1f0d7ddtplv-t2oaga2asx-watermark.awebp)

### Cursored  获取当前序列值

```
public interface Cursored{
/**

 * 获取当前序列值

 */
long getCursor();

}
复制代码
```

1：Cursored接口只提供了一个获取当前序列值的方法。

### Sequenced 序列的申请及发布

```
    public interface Sequenced{

        //获取队列的大小

        int getBufferSize();

        //判断队列中是否还有可用的容量

        boolean hasAvailableCapacity(final int requiredCapacity);

        //获取队列中剩余的有效容量

        long remainingCapacity();

        //申请下一个sequence，用于事件发布者发布数据，申请失败则自旋

        long next();

        //申请n个sequence，用于事件发布者发布数据，申请失败则自旋

        long next(int n);

        //尝试获取一个sequence

        long tryNext() throws InsufficientCapacityException;

        //尝试获取n个sequence

        long tryNext(int n) throws InsufficientCapacityException;

        //发布sequence

        void publish(long sequence);

        //批量发布sequence

        void publish(long lo, long hi);

    }
复制代码
```

### Sequencer

```
   public interface Sequencer extends Cursored, Sequenced{

        //游标初始值

        long INITIAL_CURSOR_VALUE = -1L;

        //初始化RingBuffer为指定的sequence

        void claim(long sequence);

        //消费者调用，判断sequence是否可以消费

        boolean isAvailable(long sequence);

        //将sequence添加到gating sequences中

        void addGatingSequences(Sequence... gatingSequences);

        //从gating sequences中移除指定的sequence

        boolean removeGatingSequence(Sequence sequence);

        //事件处理者用来追踪ringBuffer中可以用的sequence

        SequenceBarrier newBarrier(Sequence... sequencesToTrack);

        //事件发布者获取gating sequence中最小的sequence

        long getMinimumSequence();

        //消费者用来获取从nextSequence到availableSequence之间最大的sequence。如果是多线程生产者判断nextSequence是否可用，否则返回nextSequence-1。单线程直接返回availableSequence

        long getHighestPublishedSequence(long nextSequence, long availableSequence);

        //我也不知道干啥的

        <T> EventPoller<T> newPoller(DataProvider<T> provider,Sequence... gatingSequences);

    }
复制代码
```

1：Sequencer中的方法大多是给事件发布者使用。newBarrier()给事件处理者使用。

### AbstractSequencer 管理事件处理者序列和事件发布者发布序列。

```
    public abstract class AbstractSequencer implements Sequencer {

    //用来对gatingSequences做原子操作的。Sequence[]里面存储的是事件处理者处理到的序列。

    //如果不懂AtomicReferenceFieldUpdater请www.google.com

     private static final AtomicReferenceFieldUpdater<AbstractSequencer, Sequence[]> SEQUENCE_UPDATER =

        AtomicReferenceFieldUpdater.newUpdater(AbstractSequencer.class, Sequence[].class, "gatingSequences");

    //队列大小

    protected final int bufferSize;

    //等待策略

    protected final WaitStrategy waitStrategy;

    //事件发布者的已经发布到的sequence       

    protected final Sequence cursor = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);

    //事件处理者处理到的序列对象

    protected volatile Sequence[] gatingSequences = new Sequence[0];


    /**

     *检查队列大小是否是2^n，判断buffersize大小

     */

    public AbstractSequencer(int bufferSize, WaitStrategy waitStrategy) {

        if (bufferSize < 1) {

            throw new IllegalArgumentException("bufferSize must not be less than 1");}

        if (Integer.bitCount(bufferSize) != 1) {

            throw new IllegalArgumentException("bufferSize must be a power of 2"); }

        this.bufferSize = bufferSize;

        this.waitStrategy = waitStrategy;

    }


    /**

     * 获取事件发布者的序列

     */

    @Override

    public final long getCursor() {

        return cursor.get();

    }


    /**

     * 获取大小

     */

    @Override

    public final int getBufferSize() {

        return bufferSize;

    }


    /**

     * 把事件消费者序列维护到gating sequence

     */

    @Override

    public final void addGatingSequences(Sequence... gatingSequences) {

        SequenceGroups.addSequences(this, SEQUENCE_UPDATER, this, gatingSequences);

    }


    /**

     *  从gating sequence移除序列

     */

    @Override

    public boolean removeGatingSequence(Sequence sequence) {

        return SequenceGroups.removeSequence(this, SEQUENCE_UPDATER, sequence);

    }


    /**

     * 获取gating sequence中事件处理者处理到最小的序列值

     */

    @Override

    public long getMinimumSequence() {

        return Util.getMinimumSequence(gatingSequences, cursor.get());

    }


    /**

     * 创建了一个序列栅栏

     */

    @Override

    public SequenceBarrier newBarrier(Sequence... sequencesToTrack) {

        return new ProcessingSequenceBarrier(this, waitStrategy, cursor, sequencesToTrack);

    }


    /**

     * 这个方法不解释，我也不知道目前用来干嘛的。有知道的大佬可以赐教一下。谢谢

     */

    @Override

    public <T> EventPoller<T> newPoller(DataProvider<T> dataProvider, Sequence... gatingSequences) {

        return EventPoller.newInstance(dataProvider, this, new Sequence(), cursor, gatingSequences);

    }

    //重写toString

    @Override

    public String toString() {

        return "AbstractSequencer{" +

                "waitStrategy=" + waitStrategy +

                ", cursor=" + cursor +

                ", gatingSequences=" + Arrays.toString(gatingSequences) +

                '}';

    }

}
复制代码
```

### SingleProducerSequencer 单线程事件发布者。

1：从上面的图可以看出SingleProducerSequencer间接继承了AbstractSequencer。

2：SingleProducerSequencerFields维护事件发布者发布的序列和事件处理者处理到的最小序列。

3：SingleProducerSequencerPad缓冲行填充，防止false sharing。

#### next()申请序列

```
//该方法是事件发布者申请序列
public long next(int n) {

    if (n < 1) {

        throw new IllegalArgumentException("n must be > 0");

    }

     //获取事件发布者发布到的序列值

    long nextValue = this.nextValue;

    long nextSequence = nextValue + n;

    //wrap 代表申请的序列绕一圈以后的位置

    long wrapPoint = nextSequence - bufferSize;

    //获取事件处理者处理到的序列值

    long cachedGatingSequence = this.cachedValue;

    /** 1.事件发布者要申请的序列值大于事件处理者当前的序列值且事件发布者要申请的序列值减去环的长度要小于事件处理者的序列值。

      * 2.满足(1)，可以申请给定的序列。

      * 3.不满足(1)，就需要查看一下当前事件处理者的最小的序列值(可能有多个事件处理者)。如果最小序列值大于等于

      * 当前事件处理者的最小序列值大了一圈，那就不能申请了序列(申请了就会被覆盖)，

      * */

    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue) {

        //wrapPoint > cachedGatingSequence 代表绕一圈并且位置大于事件处理者处理到的序列

        //cachedGatingSequence > nextValue 说明事件发布者的位置位于事件处理者的屁股后面

        //维护父类中事件生产者的序列

        cursor.setVolatile(nextValue);

        long minSequence;

        //如果事件生产者绕一圈以后大于事件处理者的序列，那么会在此处自旋

        while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue))) {

            LockSupport.parkNanos(1L);

        }

        //缓存最小值

        this.cachedValue = minSequence;

    }

    this.nextValue = nextSequence;

    return nextSequence;

}

//事件发布调用的方法。唤醒阻塞的消费者
public void publish(long sequence) {

    cursor.set(sequence);

    waitStrategy.signalAllWhenBlocking();

}
复制代码
```

### 实战单线程生产者

```
public static void main(String[] args) {

    /**

     * Create a new Disruptor.

     * @param eventFactory 事件对象的数据

     * @param ringBufferSize 数组大小，必须是2^n

     * @param threadFactory 线程工厂

     * @param producerType 生产者策略。ProducerType.SINGLE和ProducerType.MULTI 单个生产者还是多个生产者.

     * @param waitStrategy 等待策略。用来平衡事件发布者和事件处理者之间的处理效率。提供了八种策略。默认是BlockingWaitStrategy

     */

    //初始化的逻辑大概是创建根据ProducerType初始化创造SingleProducerSequencer或MultiProducerSequencer。

   //初始化Ringbuffer的时候会根据buffsiz把事件对象放入entries数组。

   Disruptor<TradeBO> disruptor = new Disruptor<>(() -> new TradeBO(), 2,

            r -> {

                Thread thread = new Thread(r);

                thread.setName("实战单线程生产者");

                return thread;

            }, ProducerType.SINGLE, new BlockingWaitStrategy());

    //关联事件处理者。初始化BatchEventProcessor。把事件处理者加入gating sequence

    disruptor.handleEventsWith(new ConsumerA());

    disruptor.handleEventsWith(new ConsumerB());

    //启动消费者线程。BatchEventProcessor间接实现了Runnable。所以这一步就是启动线程。如果事件发布太快，消费太慢会根据不同的waitstrategy等待。

    disruptor.start();

    //发布事件

    for (int i = 1; i < 10; i++) {

        int finalI = i;

        //初始化了EventTranslator。意思就是给最开始初始化的对象赋值

        EventTranslator eventTranslator = (EventTranslator<TradeBO>) (event, sequence) -> {

            event.setId(finalI);

            event.setPrice((double) finalI);

        };

        //发布首先要申请序列，如果申请不到会自旋。

        disruptor.publishEvent(eventTranslator);

    }

    disruptor.shutdown();

}



  class ConsumerB implements EventHandler<TradeBO> {

   @Override

   public void onEvent(TradeBO event, long sequence,

        boolean endOfBatch) throws Exception {

    System.out.println("ConsumerB id=" + event.getId() + "price=" + event.getPrice());

        }

    }

  class ConsumerA implements EventHandler<TradeBO> {

   @Override

   public void onEvent(TradeBO event, long sequence,

        boolean endOfBatch) throws Exception {

    System.out.println("ConsumerB id=" + event.getId() + "   price=" + event.getPrice());

        }

    }


 @Data

 public class TradeBO {

    private Integer id;

    private Double price;

   }
复制代码
```

### MultiProducerSequencer

#### 成员变量

```
//获取unsafe
private static final Unsafe UNSAFE = Util.getUnsafe();
//获取int[]的偏移量
private static final long BASE = UNSAFE.arrayBaseOffset(int[].class);
//获取元素的大小，也就是int的大小4个字节
private static final long SCALE = UNSAFE.arrayIndexScale(int[].class);
//gatingSequenceCache是gatingSequence。用来标识事件处理者的序列
private final Sequence gatingSequenceCache = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);
//availableBuffer用来追踪每个槽的状态
private final int[] availableBuffer;
private final int indexMask;
//转了几圈
private final int indexShift;
复制代码
```

#### 构造函数

```
public MultiProducerSequencer(int bufferSize, final WaitStrategy waitStrategy) {

    //初始化父类

    super(bufferSize, waitStrategy);

    //初始化availableBuffer

    availableBuffer = new int[bufferSize];

    indexMask = bufferSize - 1;

    indexShift = Util.log2(bufferSize);

    //这个逻辑是。计算availableBuffer中每个元素的偏移量

    //定位数组每个值的地址就是(index * SCALE) + BASE

    initialiseAvailableBuffer();

}
private void initialiseAvailableBuffer() {

    for (int i = availableBuffer.length - 1; i != 0; i--) {

        setAvailableBufferValue(i, -1);

    }

    setAvailableBufferValue(0, -1);

}
private void setAvailableBufferValue(int index, int flag) {

    long bufferAddress = (index * SCALE) + BASE;

    //修改内存偏移地址为bufferAddress的值，改为flag

    UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);

}
复制代码
```

#### next()申请序列

```
public long next(int n) {

    if (n < 1) {

        throw new IllegalArgumentException("n must be > 0");

    }

    long current;

    long next;

    do {

        //获取事件发布者发布序列

        current = cursor.get();

        //新序列位置

        next = current + n;

        //wrap 代表申请的序列绕一圈以后的位置

        long wrapPoint = next - bufferSize;

        //获取事件处理者处理到的序列值

        long cachedGatingSequence = gatingSequenceCache.get();

        /** 1.事件发布者要申请的序列值大于事件处理者当前的序列值且事件发布者要申请的序列值减去环的长度要小于事件处理者的序列值。

         * 2.满足(1)，可以申请给定的序列。

         * 3.不满足(1)，就需要查看一下当前事件处理者的最小的序列值(可能有多个事件处理者)。如果最小序列值大于等于

         * 当前事件处理者的最小序列值大了一圈，那就不能申请了序列(申请了就会被覆盖)，

         * */

        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current) {

            //wrapPoint > cachedGatingSequence 代表绕一圈并且位置大于事件处理者处理到的序列

            //cachedGatingSequence > current 说明事件发布者的位置位于事件处理者的屁股后面


            //获取最小的事件处理者序列

            long gatingSequence = Util.getMinimumSequence(gatingSequences, current);

            if (wrapPoint > gatingSequence) {

                LockSupport.parkNanos(1);

                continue;

            }

            //赋值

            gatingSequenceCache.set(gatingSequence);

            //通过cas修改

        } else if (cursor.compareAndSet(current, next)) {

            break;

        }

    }

    while (true);


    return next;

}
复制代码
```

#### publish()事件发布

```
 public void publish(final long sequence) {

    //这里的操作逻辑大概是修改数组中的序列值

    setAvailable(sequence);

    waitStrategy.signalAllWhenBlocking();

}


 private void setAvailable(final long sequence) {

    setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence));

}

 //计算数组中位置 sequence&(buffsize-1)

 private int calculateIndex(final long sequence) {

     return ((int) sequence) & indexMask;

}

 //计算数组中的存储的数据

 private int calculateAvailabilityFlag(final long sequence) {

    return (int) (sequence >>> indexShift);

}

private void setAvailableBufferValue(int index, int flag) {

    long bufferAddress = (index * SCALE) + BASE;

    UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);

}
复制代码
```

### MultiProducerSequencer和SingleProducerSequencer区别

1：SingleProducerSequencer内部维护cachedValue(事件消费者序列)，nextValue(事件发布者序列)。并且采用padding填充。这个类是线程不安全的。
 2：MultiProducerSequencer每次获取序列都是从Sequence中获取的。Sequence中针对value的操作都是原子的。

### RingBuffer

![2849df4f0b227c47db68b75ca32f4db3ada9ef8d](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Disruptor%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.assets/16613eb913e0a802tplv-t2oaga2asx-watermark.awebp)

#### EventSequencer

```
 //这个接口是一个空方法

 public interface EventSequencer<T> extends DataProvider<T>, Sequenced{  

   }  
复制代码
```

#### DataProvider

```
 //DataProvider 提供了根据序列获取对应的对象

 //有两个地方调用。这个Event对象需要被生产者获取往里面填充数据。第二个是在消费时，获取这个Event对象用于消费。

 public interface DataProvider<T>{

     T get(long sequence);

 }
复制代码
```

#### EventSink  这个类提供了各种发布的姿势。

1：EventSink接口是用来发布Event的，在发布的同时，调用绑定的Translator来初始化并填充Event。

2：填充Event是通过实现EventTranslator，EventTranslatorOneArg，EventTranslatorTwoArg，EventTranslatorThreeArg，EventTranslatorVararg这些EventTranslator来做的。

3：发布流程：申请下一个序列->申请成功则获取对应槽的Event->利用translator初始化并填充对应槽的Event->发布Event 。translator用户实现，用于初始化Event。

#### RingBufferPad  用于缓存行填充

#### RingBufferFields 这个类的逻辑比较重要，讲解了event在数组中存储位置

```
 abstract class RingBufferFields<E> extends com.lmax.disruptor.RingBufferPad {
//Buffer数组填充
private static final int BUFFER_PAD;
//Buffer数组起始基址
private static final long REF_ARRAY_BASE;
//数组引用每个引用占用的大小=2^REF_ELEMENT_SHIFT
private static final int REF_ELEMENT_SHIFT;
private static final Unsafe UNSAFE = Util.getUnsafe();

static {

    //获取Object[]引用大小。我本机4字节

    final int scale = UNSAFE.arrayIndexScale(Object[].class);

    if (4 == scale) {

        REF_ELEMENT_SHIFT = 2;

    } else if (8 == scale) {

        REF_ELEMENT_SHIFT = 3;

    } else {

        throw new IllegalStateException("Unknown pointer size");

    }

    //填充32或者16

    BUFFER_PAD = 128 / scale;

    // 计算Buffer数组起始基址。我本机是从32开始

    REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);

}

private final long indexMask;
//保存了RingBuffer每个槽的Event对象。这个entries不会被修改。ps:引用不会被修改
private final Object[] entries;
protected final int bufferSize;
//sequencer=SingleProducerSequencer or MultiProducerSequencer的引用
protected final Sequencer sequencer;


RingBufferFields(

        EventFactory<E> eventFactory,

        Sequencer sequencer) {

    this.sequencer = sequencer;

    this.bufferSize = sequencer.getBufferSize();


    if (bufferSize < 1) {

        throw new IllegalArgumentException("bufferSize must not be less than 1");

    }

    if (Integer.bitCount(bufferSize) != 1) {

        throw new IllegalArgumentException("bufferSize must be a power of 2");

    }


    this.indexMask = bufferSize - 1;

    this.entries = new Object[sequencer.getBufferSize() + 2 * BUFFER_PAD];

    fill(eventFactory);

}
//填充entries
private void fill(EventFactory<E> eventFactory) {

    for (int i = 0; i < bufferSize; i++) {

        entries[BUFFER_PAD + i] = eventFactory.newInstance();

    }

}

@SuppressWarnings("unchecked")
protected final E elementAt(long sequence) {

    return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));

  }

}
复制代码
```

### SequenceBarrier接口 消费者使用

```
    public interface SequenceBarrier {


        /**

         * 等待一个序列变为可用，然后消费这个序列。消费线程中使用

         */

        long waitFor(long sequence) throws AlertException, InterruptedException, TimeoutException;


        /**

         * 获取当前可以读取的序列值。

         */

        long getCursor();

        /**

         * 当前栅栏是否发过通知。

         */

        boolean isAlerted();           

       /**

         * 通知消费者状态变化，然后停留在这个状态上，直到状态被清除。

         */

        void alert();

        /**

         * 清楚通知状态。

         */

        void clearAlert();

        /**

         * 检测是否发生了通知，如果已经发生了抛出AlertException异常。

         */

        void checkAlert() throws AlertException;

    } 
复制代码
```

### ProcessingSequenceBarrier

```
final class ProcessingSequenceBarrier implements SequenceBarrier {
//等待策略
private final WaitStrategy waitStrategy;
//当消费者之前没有依赖关系的时候，那么dependentSequence=cursorSequence
//存在依赖关系的时候，dependentSequence 里存放的是一组依赖的Sequence，get方法得到的是最小的序列值
//所谓的依赖关系是有两个消费者A、B，其中B需要在A之后进行消费，这A的序列就是B需要依赖的序列，因为B的消费速度不能超过A。
private final Sequence dependentSequence;
//判断是否执行shutdown
private volatile boolean alerted = false;
//cursorSequence 代表的是写指针。代表事件发布者发布到那个位置
private final Sequence cursorSequence;
//sequencer=SingleProducerSequencer or MultiProducerSequencer的引用
private final Sequencer sequencer;


ProcessingSequenceBarrier(

        final Sequencer sequencer,

        final WaitStrategy waitStrategy,

        final Sequence cursorSequence,

        final Sequence[] dependentSequences) {

    this.sequencer = sequencer;

    this.waitStrategy = waitStrategy;

    this.cursorSequence = cursorSequence;

    if (0 == dependentSequences.length) {

        dependentSequence = cursorSequence;

    } else {

        dependentSequence = new FixedSequenceGroup(dependentSequences);

    }

}

@Override
public long waitFor(final long sequence)

        throws AlertException, InterruptedException, TimeoutException {

    //检查是否中断

    checkAlert();

    //根据不同的策略获取可用的序列

    long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);

    //判断申请的序列和可用的序列大小

    if (availableSequence < sequence) {

        return availableSequence;

    }

    //如果是单线程生产者直接返回availableSequence

    //多线程生产者判断是否可用，不可用返回sequence-1

    return sequencer.getHighestPublishedSequence(sequence, availableSequence);

}
//获取当前序列
@Override
public long getCursor() {

    return dependentSequence.get();

}
//判断是否中断
@Override
public boolean isAlerted() {

    return alerted;

}
//中断
@Override
public void alert() {

    alerted = true;

    waitStrategy.signalAllWhenBlocking();

}
//清除中断
@Override
public void clearAlert() {

    alerted = false;

}
//检查是否中断
@Override
public void checkAlert() throws AlertException {

    if (alerted) {

        throw AlertException.INSTANCE;

     }

   }

}
复制代码
```

### 事件处理 EventProcessor

```
 public interface EventProcessor extends Runnable{

   //获取事件处理器使用的序列引用。 

   Sequence getSequence();

   //中断

   void halt();

   //判断是否运行

   boolean isRunning();

 }
复制代码
```

### BatchEventProcessor event模式单线程处理

```
  //重点讲run方法，其它方法都比较简单

  public final class BatchEventProcessor<T>

    implements EventProcessor {


  public void run() {

    //启动任务

    if (running.compareAndSet(IDLE, RUNNING)) {

    //清除中断状态

    sequenceBarrier.clearAlert();

    //判断一下消费者是否实现了LifecycleAware ,如果实现了这个接口，那么此时会发送一个启动通知

    notifyStart();

    try {

         //判断任务是否启动

        if (running.get() == RUNNING) {

            //处理事件

            processEvents();

        }

    } finally {

         //判断一下消费者是否实现了LifecycleAware ,如果实现了这个接口，那么此时会发送一个停止通知

        notifyShutdown();

        //重新设置状态

        running.set(IDLE);

    }

} else {

    // 线程已经启动

    if (running.get() == RUNNING) {

        throw new IllegalStateException("Thread is already running");

    } else {

        //这里就是  notifyStart();notifyShutdown();

        earlyExit();

      }

   }

}


 private void processEvents() {
//定义一个event

T event = null;
//获取要申请的序列

long nextSequence = sequence.get() + 1L;
//循环处理事件。除非超时或者中断。
while (true) {

    try {

        //根据等待策略来等待可用的序列值。 

        final long availableSequence = sequenceBarrier.waitFor(nextSequence);

        if (batchStartAware != null) {

            batchStartAware.onBatchStart(availableSequence - nextSequence + 1);

        }

         //根据可用的序列值获取事件。批量处理nextSequence到availableSequence之间的事件。

        while (nextSequence <= availableSequence) {

            //获取事件

            event = dataProvider.get(nextSequence);

            //触发事件

            eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);

            nextSequence++;

        }

        //设置事件处理者处理到的序列值。事件发布者会根据availableSequence判断是否发布事件 

        sequence.set(availableSequence);

    } catch (final TimeoutException e) {

        //超时异常

        notifyTimeout(sequence.get());

    } catch (final AlertException ex) {

        //中断异常

        if (running.get() != RUNNING) {

            break;

        }

    } catch (final Throwable ex) {

        //这里可能用户消费者事件出错。如果自己实现了ExceptionHandler那么就不会影响继续消费

        exceptionHandler.handleEventException(ex, nextSequence, event);

        //如果出现异常则设置为nextSequence

        sequence.set(nextSequence);

        nextSequence++;

       }

   }

}
复制代码
```

### WorkProcessor  work模式多线程处理

```
public void run() {

    //判断线程是否启动

    if (!running.compareAndSet(false, true)) {

        throw new IllegalStateException("Thread is already running");

    }

    //清除中断状态

    sequenceBarrier.clearAlert();

    //判断一下消费者是否实现了LifecycleAware ,如果实现了这个接口，那么此时会发送一个启动通知

    notifyStart();

    //事件处理标志

    boolean processedSequence = true;

    long cachedAvailableSequence = Long.MIN_VALUE;

    long nextSequence = sequence.get();

    T event = null;

    while (true) {

        try {

           //判断上一个事件是否已经处理完毕。  

            if (processedSequence) {

                //置为false

                processedSequence = false;

                do {

                    //获取下一个序列

                    nextSequence = workSequence.get() + 1L;

                    //更新当前已经处理到的

                    sequence.set(nextSequence - 1L);

                }

            //多个WorkProcessor共享一个workSequence，可以实现互斥消费，因为只有一个线程可以CAS更新成功

                while (!workSequence.compareAndSet(nextSequence - 1L, nextSequence));

            }

           //检查序列值是否需要申请。

            if (cachedAvailableSequence >= nextSequence) {

                //获取事件

                event = ringBuffer.get(nextSequence);

               //交给workHandler处理事件。  

                workHandler.onEvent(event);

               //设置事件处理完成标识

                processedSequence = true;

            } else {

                //申请可用序列

                cachedAvailableSequence = sequenceBarrier.waitFor(nextSequence);

            }

        } catch (final TimeoutException e) {

            notifyTimeout(sequence.get());

        } catch (final AlertException ex) {

            if (!running.get()) {

                break;

            }

        } catch (final Throwable ex) {

            //设置异常事件处理

            exceptionHandler.handleEventException(ex, nextSequence, event);

            processedSequence = true;

        }

    }

    //同上

    notifyShutdown();

    //停止

    running.set(false);

}
复制代码
```

### WorkerPool

1：多个WorkProcessor组成一个WorkerPool。
 2：维护workSequence事件处理者处理的序列。

### waitStrategy 等待策略

BlockingWaitStrategy：默认的等待策略。利用锁和等待机制的WaitStrategy，CPU消耗少，但是延迟比较高

BusySpinWaitStrategy：自旋等待。这种策略会利用CPU资源来避免系统调用带来的延迟抖动，当线程可以绑定到指定CPU(核)的时候可以使用这个策略。

LiteBlockingWaitStrategy：实现方法也是阻塞等待

SleepingWaitStrategy：是另一种较为平衡CPU消耗与延迟的WaitStrategy，在不同次数的重试后，采用不同的策略选择继续尝试或者让出CPU或者sleep。这种策略延迟不均匀。

TimeoutBlockingWaitStrategy：实现方法是阻塞给定的时间，超过时间的话会抛出超时异常。

YieldingWaitStrategy：实现方法是先自旋(100次)，不行再临时让出调度(yield)。和SleepingWaitStrategy一样也是一种高性能与CPU资源之间取舍的折中方案，但这个策略不会带来显著的延迟抖动。

PhasedBackoffWaitStrategy：实现方法是先自旋(10000次)，不行再临时让出调度(yield)，不行再使用其他的策略进行等待。可以根据具体场景自行设置自旋时间、yield时间和备用等待策略。

### 实战多线程消费者

```
  public static void main(String[] args) {

    //创建一个RingBuffer，注意容量是2。

    RingBuffer<TradeBO> ringBuffer = RingBuffer.createSingleProducer(() -> new TradeBO(), 2);

    //创建2个WorkHandler其实就是创建2个WorkProcessor

    WorkerPool<TradeBO> workerPool =

            new WorkerPool<TradeBO>(ringBuffer, ringBuffer.newBarrier(),

                    new IgnoreExceptionHandler(),

                    new ConsumerC(), new ConsumerD());

    //将WorkPool的工作序列集设置为ringBuffer的追踪序列。

    ringBuffer.addGatingSequences(workerPool.getWorkerSequences());

    //创建一个线程池用于执行Workhandler。

    Executor executor = Executors.newFixedThreadPool(4);

    //启动WorkPool。

    workerPool.start(executor);

    //往RingBuffer上发布事件

    for (int i = 0; i < 4; i++) {

        int finalI = i;

        EventTranslator eventTranslator = (EventTranslator<TradeBO>) (event, sequence) -> {

            event.setId(finalI);

            event.setPrice((double) finalI);

        };

        ringBuffer.publishEvent(eventTranslator);

        System.out.println("发布[" + finalI + "]");

    }

}

  //程序执行结果。可以看出，多个线程消费者处理位于不同位置的事件

  发布[0]

  ConsumerC id=0   price=0.0

  发布[1]

  发布[2]

  ConsumerC id=2   price=2.0

  ConsumerD id=1   price=1.0

  ConsumerC id=3   price=3.0

  发布[3]
复制代码
```

### DSL

1:所谓DSL我的理解就是消费者这里相互依赖。

![640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/9/26/16613eb91644602a~tplv-t2oaga2asx-watermark.awebp)

![86efd3e723b80a7decf98eadaa58b167e212b47f](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Disruptor%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.assets/16613eb91f8d86edtplv-t2oaga2asx-watermark.awebp)

```
 dw.consumeWith(handler1a, handler2a);

 dw.after(handler1a).consumeWith(handler1b);

 dw.after(handler2a).consumeWith(handler2b);

 dw.after(handler1b, handler2b).consumeWith(handler3);

 ProducerBarrier producerBarrier = dw.createProducerBarrier();
复制代码
```

### 消费者的等待策略

| 名称                        | 措施                      | 适用场景                                                     |
| :-------------------------- | :------------------------ | :----------------------------------------------------------- |
| BlockingWaitStrategy        | 加锁                      | CPU资源紧缺，吞吐量和延迟并不重要的场景                      |
| BusySpinWaitStrategy        | 自旋                      | 通过不断重试，减少切换线程导致的系统调用，而降低延迟。推荐在线程绑定到固定的CPU的场景下使用 |
| PhasedBackoffWaitStrategy   | 自旋 + yield + 自定义策略 | CPU资源紧缺，吞吐量和延迟并不重要的场景                      |
| SleepingWaitStrategy        | 自旋 + yield + sleep      | 性能和CPU资源之间有很好的折中。延迟不均匀                    |
| TimeoutBlockingWaitStrategy | 加锁，有超时限制          | CPU资源紧缺，吞吐量和延迟并不重要的场景                      |
| YieldingWaitStrategy        | 自旋 + yield + 自旋       | 性能和CPU资源之间有很好的折中。延迟比较均匀                  |



