在使用硬件SPI过程中，会出现的问题可以总结为以下几点：

　　**1.发送数据不成功；**

　　**2.接收数据不成功；**

　　**3.发送的数据有误；**

　　**4.接收的数据有误；**

　　**5.交互的数据一部分是对的，一部分有误；**

　　**6.SPI时钟没有启动。**

 

对于以上解决方法，我总结了一个自己调试时的方法：

　　**1. 先确认自己的SPI配置是否正确，是否满足项目需求；**

　　**2. 确认电路与通信IC无误，注意信号线不要接错；**

　　**3. 重点：调节延时，第一第二步确认无误后，很多时候不成功是由于延时原因造成的，**

　　  **主要是一个数据交互之间的延时，一帧数据发送后跟接收的延时，IC片选的延时，**

　　　**每个数据发送间的延时，IC与MCU交互间的延时。**

 

补充：有时候发送出来的数据不对，有可能是分频因子太小，配置的速度太快导致的，就例如发0x01，收到0x2c之类的；如果发送出来的数据多了几个，可能就是发送数据长度设置不对造成的，例如发送0xaa、0xff 时发送出去多了0x20、0x00之类的，就要考虑是不是数据长度配置不对了，具体配置详细说明如下图：

　　![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\1433771-20181109141044020-1908274468.png)





## 1、SPI协议简介

### **1.1 SPI协议概括**

　　SPI，是英语Serial Peripheral interface的缩写，顾名思义就是串行外围设备接口。是Motorola首先在其MC68HCXX系列处理器上定义的。SPI接口主要应用在 EEPROM，FLASH，实时时钟，AD转换器，还有数字信号处理器和数字信号解码器之间。**SPI，是一种高速的，全双工，同步的通信总线，**并且在芯片的管脚上只占用四根线，节约了芯片的管脚，同时为PCB的布局上节省空间，提供方便，正是出于这种简单易用的特性，现在越来越多的芯片集成了这种通信协议，比如MSP430单片机系列处理器。

**1.2 SPI优点**

1）：支持全双工通信

2）：通信简单

3）：数据传输速率快

**1.3 缺点**
没有指定的流控制，没有应答机制确认是否接收到数据，所以跟IIC总线协议比较在数据
可靠性上有一定的缺陷。

## **2、 特点**

**2.1 采用主-从模式(Master-Slave) 的控制方式**

SPI 规定了两个 SPI 设备之间通信必须由主设备 (Master) 来控制次设备 (Slave)。 一个 Master 设备可以通过提供 Clock 以及对 Slave 设备进行**片选 (Slave Select) 来控制多个 Slave 设备**，SPI 协议还规定 Slave 设备的 Clock 由 Master 设备通过 SCK 管脚提供给 Slave 设备， Slave 设备本身不能产生或控制 Clock，没有 Clock 则 Slave 设备不能正常工作。

### **2.2 采用同步方式(Synchronous)传输数据**

Master 设备会根据将要交换的数据来产生相应的时钟脉冲(Clock Pulse)，时钟脉冲组成了时钟信号(Clock Signal) ，**时钟信号通过时钟极性 (CPOL) 和 时钟相位 (CPHA) 控制着两个 SPI 设备间何时数据交换以及何时对接收到的数据进行采样，来保证数据在两个设备之间是同步传输的**。

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-b866b6b9eaa992e64e7910284dca2de2_720w.jpg)

### **2.3 数据交换(Data Exchanges)**

SPI 设备间的数据传输之所以又被称为**数据交换**，**是因为 SPI 协议规定一个 SPI 设备不能在数据通信过程中仅仅只充当一个 "发送者(Transmitter)" 或者 "接收者(Receiver)"**。在每个 Clock 周期内，**SPI 设备都会发送并接收一个 bit 大小的数据(**不管主设备好还是从设备**)**，相当于该设备有一个 bit 大小的数据被交换了。一个 Slave 设备要想能够接收到 Master 发过来的控制信号，必须在此之前能够被 Master 设备进行访问 (Access)。所以，Master 设备必须首先通过 SS/CS pin 对 Slave 设备进行片选, 把想要访问的 Slave 设备选上。 在数据传输的过程中，每次接收到的数据必须在下一次数据传输之前被采样。如果之前接收到的数据没有被读取，那么这些已经接收完成的数据将有可能会被丢弃，导致 SPI 物理模块最终失效。**因此，在程序中一般都会在 SPI 传输完数据后，去读取 SPI 设备里的数据, 即使这些数据(Dummy Data)在我们的程序里是无用的(**虽然发送后紧接着的读取是无意义的，但仍然需要从寄存器中读出来**)**。

### **2.5 SPI只有主模式和从模式之分**

**SPI没有读和写的说法，**因为实质上每次SPI是主从设备在交换数据。也就是说，你发一个数据必然会收到一个数据；你要收一个数据必须也要先发一个数据。

## **3、 工作机制**

**3.1 概述**

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-daba5c2b915b794e24ff0a480ea7a3c0_720w.jpg)

上图只是对 SPI 设备间通信的一个简单的描述, 下面就来解释一下图中所示的几个组件:

- SSPBUF：泛指 SPI 设备里面的内部缓冲区，一般在物理上是以 FIFO 的形式，保存传输过程中的临时数据；
- SSPSR：泛指 SPI 设备里面的移位寄存器，它的作用是根据设置好的数据位宽(bit-width) 把数据移入或者移出 SSPBUF；
- Controller：泛指 SPI 设备里面的控制寄存器，可以通过配置它们来设置 SPI 总线的传输模式。

通常情况下，我们只需要对上图所描述的四个管脚(pin) 进行编程即可控制整个 SPI 设备之间的数据通信：

- SCK：主要的作用是 Master(主)设备往 Slave(从)设备传输时钟信号, 控制数据交换的时机以及速率；
- SS/CS：用于 Master(主)设备片选 Slave (从)设备，使被选中的 Slave(从)设备能够被 Master(主)设备所访问；
- SDO/MOSI：在 Master(主)上面也被称为 Tx-Channel，作为数据的出口，主要用于 SPI 设备发送数据；
- SDI/MISO：在 Master(主)上面也被称为 Rx-Channel，作为数据的入口，主要用于SPI 设备接收数据；

SPI 设备在进行通信的过程中，Master 设备和 Slave 设备之间会产生一个数据链路回环(Data Loop)，就像上图所画的那样, 通过 SDO 和 SDI 管脚, SSPSR 控制数据移入移出 SSPBUF，Controller 确定 SPI 总线的通信模式，SCK 传输时钟信号。

**SDI(数据输入)、SDO(数据输出)、SCK(时钟)、CS(片选)**

(1)、SDO/MOSI – 主设备数据输出，从设备数据输入；

(2)、SDI/MISO – 主设备数据输入，从设备数据输出；

(3)、SCK – 时钟信号，由主设备产生；

(4)、CS/SS – 从设备使能信号，由主设备控制。当有多个从设备的时候，因为每个从设备上都有一个片选引脚接入到主设备机中，当我们的主设备和某个从设备通信时将需要将从设备对应的片选引脚电平拉低或者是拉高。

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-ffd6d862406c5201c9148bd5138df467_720w.jpg)

### **3.2 SPI相关的缩写或说法**

SPI的极性Polarity和相位Phase，最常见的写法是**CPOL**和**CPHA**，不过也有一些其他写法，简单总结如下：

- CKPOL (Clock Polarity) = CPOL = POL = Polarity = （时钟）极性
- CKPHA (Clock Phase) = CPHA = PHA = Phase = （时钟）相位
- **SCK=SCLK=SPI的时钟**
- Edge=边沿，即时钟电平变化的时刻，即上升沿(rising edge)或者下降沿(falling edge)对于一个时钟周期内，有两个edge，分别称为：
- Leading edge=前一个边沿=第一个边沿，对于开始电压是1，那么就是1变成0的时候，对于开始电压是0，那么就是0变成1的时候；
- Trailing edge=后一个边沿=第二个边沿，对于开始电压是1，那么就是0变成1的时候（即在第一次1变成0之后，才可能有后面的0变成1），对于开始电压是0，那么就是1变成0的时候；

**3.3 SPI总线的极性和相位**

CPOL配置SPI总线的极性，CPHA配置SPI总线的相位。

**3.3.1 SPI总线的极性**

极性，会直接影响SPI总线空闲时的时钟信号是**高电平**还是**低电平**。

CPOL = 1：表示空闲时是高电平；

CPOL = 0：表示空闲时是低电平。

由于数据传输往往是从跳变沿开始的，也就表示开始传输数据的时候，是下降沿还是上升沿。如下图：

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-a373c92ee485b1911caadf3ed81c0ee7_720w.jpg)

**3.3.2 SPI总线的相位**

一个时钟周期会有2个跳变沿。而相位，直接决定SPI总线从那个跳变沿开始采样数据。

CPHA = 0：表示从第一个跳变沿开始采样；

CPHA = 1：表示从第二个跳变沿开始采样。

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-db50e4c4ad830877d9d57faa6cad657c_720w.jpg)

至于跳变沿究竟是上升沿还是下降沿，这取决于 CPOL。记住， CPHA 只决定是哪个跳变沿采样。

**3.4 SPI总线传输的四种模式**

CPOL 和 CPHA 的不同组合，形成了SPI总线的不同模式。

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-bcb17956e11dfdc372c8ebed86b4c691_720w.jpg)

时钟极性CPOL是用来配置SCLK的电平出于哪种状态时是空闲态或者有效态，时钟相位CPHA
是用来配置数据采样是在第几个边沿：
CPOL=0，表示当SCLK=0时处于空闲态，所以有效状态就是SCLK处于高电平时。
CPOL=1，表示当SCLK=1时处于空闲态，所以有效状态就是SCLK处于低电平时。
CPHA=0，表示数据采样是在第1个边沿，数据发送在第2个边沿。
CPHA=1，表示数据采样是在第2个边沿，数据发送在第1个边沿。

例如：
CPOL=0，CPHA=0：此时空闲态时，SCLK处于低电平，数据采样是在第1个边沿，也就是
SCLK由低电平到高电平的跳变，所以数据采样是在上升沿，数据发送是在下降沿。

CPOL=0，CPHA=1：此时空闲态时，SCLK处于低电平，数据发送是在第1个边沿，也就是
SCLK由低电平到高电平的跳变，所以数据采样是在下降沿，数据发送是在上升沿。

CPOL=1，CPHA=0：此时空闲态时，SCLK处于高电平，数据采集是在第1个边沿，也就是
SCLK由高电平到低电平的跳变，所以数据采集是在下降沿，数据发送是在上升沿。

CPOL=1，CPHA=1：此时空闲态时，SCLK处于高电平，数据发送是在第1个边沿，也就是
SCLK由高电平到低电平的跳变，所以数据采集是在上升沿，数据发送是在下降沿。

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-048c4a8e026abf7761c1be5ad9a1d499_720w.jpg)

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-9501c10cdacce1cb7830f8f0773524f7_720w.jpg)

需要注意的是：我们的主设备能够控制时钟，因为SPI通信并不像UART或者IIC通信那样有专门的通信周期，有专门的通信起始信号，有专门的通信结束信号；所以SPI协议能够通过控制时钟信号线，当没有数据交流的时候我们的时钟线要么是保持高电平要么是保持低电平。

------

**更新：**

首先说明一点，capture strobe = latch = read = sample，都是表示数据采样，数据有效的时刻。相位，对应着数据采样是在第几个边沿（edge），是第一个边沿还是第二个边沿，0对应着第一个边沿，1对应着第二个边沿。

对于：

CPHA=0，表示第一个边沿：

对于CPOL=0，idle时候的是低电平，第一个边沿就是从低变到高，所以是上升沿；

对于CPOL=1，idle时候的是高电平，第一个边沿就是从高变到低，所以是下降沿；

CPHA=1，表示第二个边沿：

对于CPOL=0，idle时候的是低电平，第二个边沿就是从高变到低，所以是下降沿；

对于CPOL=1，idle时候的是高电平，第一个边沿就是从低变到高，所以是上升沿；

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-c74a5cdb51dc6e824418f6f3893aaaf0_720w.jpg)

------

**3.4 SSPSR**

![img](C:\Users\putao\Documents\Blog\嵌入式知识\SPI相关.assets\v2-daba5c2b915b794e24ff0a480ea7a3c0_720w.jpg)

SSPSR 是 SPI 设备内部的移位寄存器(Shift Register)。它的主要作用是根据 SPI 时钟信号状态，往 SSPBUF 里移入或者移出数据，每次移动的数据大小由 Bus-Width 以及 Channel-Width 所决定。

Bus-Width 的作用是指定地址总线到 Master(主)设备之间数据传输的单位。

例如, 我们想要往 Master(主)设备里面的 SSPBUF 写入 16 Byte 大小的数据：首先，给 Master(主)设备的配置寄存器设置 Bus-Width 为 Byte；然后往 Master 设备的 Tx-Data 移位寄存器在地址总线的入口写入数据，每次写入 1 Byte 大小的数据(使用 writeb 函数)；写完 1 Byte 数据之后，Master 设备里面的 Tx-Data 移位寄存器会自动把从地址总线传来的1 Byte 数据移入 SSPBUF 里；上述动作一共需要重复执行 16 次。

Channel-Width 的作用是指定 Master(主)设备与 Slave(从)设备之间数据传输的单位。与 Bus-Width 相似，Master(主)设备内部的移位寄存器会依据 Channel-Width 自动地把数据从 Master-SSPBUF 里通过 Master-SDO 管脚搬运到 Slave(从)设备里的 Slave-SDI 引脚, Slave－SSPSR 再把每次接收的数据移入 Slave-SSPBUF里.通常情况下，Bus-Width 总是会大于或等于 Channel-Width，这样能保证不会出现因 Master 与 Slave 之间数据交换的频率比地址总线与 Master 之间的数据交换频率要快，导致 SSPBUF 里面存放的数据为无效数据这样的情况。

**3.5 SSPBUF**

我们知道, 在每个时钟周期内，Master(主)与 Slave(从)之间交换的数据其实都是 SPI 内部移位寄存器从 SSPBUF 里面拷贝的。我们可以通过往 SSPBUF 对应的寄存器 (Tx-Data / Rx-Data register) 里读写数据，间接地操控 SPI 设备内部的 SSPBUF。

例如，在发送数据之前，我们应该先往 Master(主)的 Tx-Data 寄存器写入将要发送出去的数据，这些数据会被 Master-SSPSR 移位寄存器根据 Bus-Width 自动移入 Master-SSPBUF 里，然后这些数据又会被 Master-SSPSR 根据 Channel-Width 从 Master-SSPBUF 中移出, 通过 Master-SDO 管脚传给 Slave-SDI 管脚，Slave-SSPSR 则把从 Slave-SDI 接收到的数据移入 Slave-SSPBUF 里。与此同时，Slave-SSPBUF 里面的数据根据每次接收数据的大小(Channel-Width)，通过 Slave-SDO 发往 Master-SDI，Master-SSPSR 再把从 Master-SDI 接收的数据移入 Master-SSPBUF.在单次数据传输完成之后，用户程序可以通过从 Master 设备的 Rx-Data 寄存器读取 Master 设备数据交换得到的数据。

**3.6 Controller**

Master(主)设备里面的 Controller 主要通过时钟信号(Clock Signal)以及片选信号(Slave Select Signal)来控制 Slave(从)设备。Slave 设备会一直等待，直到接收到 Master 设备发过来的片选信号，然后根据时钟信号来工作。

Master(主)设备的片选操作必须由程序所实现。例如: 由程序把 SS/CS 管脚的时钟信号拉低电平，完成 SPI 设备数据通信的前期工作；当程序想让 SPI 设备结束数据通信时，再把 SS/CS 管脚上的时钟信号拉高电平。

