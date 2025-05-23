# **后端基础流水线**

处理器的后端有4个独立的功能单元，分别是：

* 算数逻辑运算单元（第一运算单元）：处理算数逻辑指令和计数器读取指令
* 算术逻辑&分支跳转单元（第二运算单元）：处理算数逻辑指令和分支跳转指令
* 乘除&特权处理单元：处理乘除法和特权指令
* 访存处理单元：处理器加载、存储和同步指令

## **1 发射与唤醒**

处理器的发射队列采用压缩队列进行实现，其中算数逻辑运算单元和分支跳转单元的发射队列为乱序发射队列，而为了保证特权指令的顺序和龙芯架构32位精简版指令集中对于“强序非缓存”的定义，乘除和访存单元的发射队列为顺序发射队列。

### **1.1 发射**

发射队列的每个表项包括：

* 指令包：包含当前指令在后端所需的全部信号
* 寄存器唤醒标记：表示当前寄存器是否被前序指令唤醒
* 寄存器由加载指令唤醒标记：表示当前寄存器是否由加载指令唤醒

发射操作由发射队列和选择模块协同完成。这两个模块间使用Request-Acknowledge协议进行通讯。

顺序发射队列仅会在寄存器已经就绪时，使最旧的指令向选择模块发送发射请求，若后续流水线无阻塞，则选择模块会响应这个请求，并将指令发射到后续流水线中。而在乱序发射队列中，每一个有效表项都会向选择模块发送发射请求，而选择模块会使用优先编码器，响应其中最旧指令的请求，这是因为：

* 最旧的指令往往处于相关链顶端，发射后可以唤醒更多指令
* 对于读计数器指令，这种策略可以保证连续两条读取指令读取到的数据是循环递增的

### **1.2 唤醒**

发射队列中的每一个表项都会将自身的两个源寄存器编号与唤醒总线上的4个寄存器编号进行比较，若相等则将唤醒标记置位。在处理器中，有两处唤醒操作较为重要：

* **推测唤醒**：由于加载指令往往处于相关链的顶端，因此访存处理单元采取了推测唤醒的方式。在访存的第一个周期，该指令便开始唤醒其他指令，若唤醒成功则将加载指令唤醒标记置位。若下一个周期发现该指令发生了高速缓存缺失，则所有被当前指令唤醒的指令应当在缺失处理完成前滞留在发射队列中，否则正常发射即可。
* **相关指令连续发射**：算数逻辑指令占据了程序绝大部分比重，因此这些指令若能够在相关时连续发射，就能够对处理器性能有较大提升，故该处理器中算术逻辑队列的发射与唤醒在同一个周期内完成。

## **2 读寄存器堆**

为了适应四个发射队列独立读取和写入，寄存器堆共有8个读端口和4个写端口，每一个读端口都配备了写优先。由于后端所有指令不可能出现目的寄存器相同的两条指令，因此写优先至多命中一个写数据，故可以将四个写入数据前递到每一个读端口上。

## **3 算数执行**

该处理器共有两个ALU，支持龙芯架构32位精简版指令集规定的全部11种运算。其中：

* 第一运算单元承担了读取计数器值的功能
* 第二运算单元承担了为BL和JIRL指令计算PC+4的功能

## **4 乘法与除法**

### **4.1 华莱士树流水乘法**

处理器的乘法单元采用了3级流水的华莱士树流水乘法，使用33位扩展乘法来实现龙芯架构32位精简版指令集中全部乘法指令。具体如下：

* 第一级：2位Booth编码，使用乘数将被乘数编码，将部分和的数量由32削减至16
* 第二级：8层66位保留进位加法器树，每层将部分和削减三分之一，直至剩余两个部分和
* 第三级：64位全加器，将最后两个部分和相加，得到最后结果。

### **4.2 对数加速移位除法**

除法和取余运算本质相同，都类似于竖式除法的方式进行移位相减。

当被除数较小时，竖式除法是没有必要使用32个周期完成运算的。因此，处理器在第一个周期取被除数的对数（即寻找最高位的1是第几位），记录在计数器中，除法开始后，只需要运行被除数的对数个周期，即可完成运算。

## **5 存储器访问**

考虑到时序问题，访存流水线的设计与其余流水线有较大差别，也引入了一些特殊的部件。下面一一进行介绍

### **5.1 发射与读取寄存器堆**

由于访存队列是压缩的且是顺序发送的，因此寄存器堆的读地址只会来自发射队列的第0个表项，是相对固定的，故发射队列和寄存器堆之间没有段间寄存器堆，指令发射与读取寄存器堆是同步进行的。

### **5.2 地址计算与TLB命中判断**

TLB的全相连结构使其前后都不可有任何其他的组合逻辑，若存在组合逻辑，则需要对TLB读取进行切割，因此和取指不同，访存的TLB采用了两级切割。第一个流水级，访存单元计算出虚拟地址，并和TLB的每一个表项进行命中判断，将命中信息独热码送入段间寄存器。

在这个流水级，虚拟地址也被送入数据高速缓存（DCache），根据虚拟Index物理Tag原理，这个周期可以使用虚拟地址中的Index部分开始查找Tag和Data表所在的BRAM。

### **5.3 TLB读取与标签比较**

段间寄存器中的TLB命中独热码可以作为独热码，对TLB表项进行索引。索引出的结果将送入DCache，和读出的Tag进行比较，并将命中信息送入下一段间寄存器。

由于此阶段的访存请求都是推测的，因此写操作不能在此时产生效果，故这个阶段需要将写请求写入一个写缓冲（Store Buffer）中，当存储指令被提交时，该指令才会对高速缓存发起写操作。在执行加载指令时，也要同步查找写缓冲，查找方式如下：

* 对当前请求地址对应的4个字节分别进行全相连查找，将命中信息锁存一个周期
* 利用命中信息读取对应的写缓冲数据，当有多项命中时，根据尾指针的位置找到最近写入的数据

### **5.4 高速缓存缺失判断与数据重组**

为优化时序，高速缓存在获取到命中信息后，并不会立刻使用状态机进行缺失判断，而是锁存一个周期后送入状态机。

为了使得龙芯架构中的强序非缓存访问能够按照顺序确定地执行，所有的非可缓存访问都会滞留在这个阶段，直至这个访问被确认为是最旧的一条指令，流水线才会继续流动。

对于加载指令，其数据可能来自于高速缓存，也可能来自于写缓冲，因此在这个阶段需要根据写缓冲的命中情况，对数据进行拼接。

### **5.5 写回阶段**

写回阶段将会对物理寄存器堆发起写操作，并标记重排序缓存对应表项的完成状态。

为了加速运算指令的执行，这个阶段在第一、二运算和访存流水线中架设了旁路网络：

* 第一、第二运算流水线可以相互前递数据，使得二者间的相关指令可以连续发射；
* 第一、第二运算和访存流水线可以将数据前递回访存流水线的地址计算阶段，压缩了访存的流水线级数

## **6 提交阶段**

处理器的提交阶段负责维护处理器状态，主要包括重排序缓存（ROB）和状态恢复元件（ARAT）。

* ROB采用伪双端口交叠FIFO的形式实现，每个周期最多可以提交两条指令，在基础架构中，若头部指令不为跳转指令，则若ROB队列头部两条指令均完成执行，则会提交两条指令。
* ARAT中维护了寄存器映射表的有效位，可在分支预测失败时恢复到重命名单元中。同时，ARAT还维护了分支预测器返回地址栈的栈顶指针，在分支预测失败时，也会恢复到分支预测器中，对分支预测正确率有一定提升。
