<h1><center>CPU交接：从加电到BIOS再到MBR</center></h1>

## 一、计算机的启动过程

在谈计算机的启动过程之前，我们先来谈一下“载入内存”。

载入内存分为两个阶段：

1. 程序被“加载器”放到内存中
2. cs:ip寄存器指向这段程序的首地址



计算机上电之后，首先执行的是BIOS，那么BIOS是什么：Base Input Output System——基本输入输出系统。

于是我们有3个问题：

1. BIOS是被谁加载的（程序载入内存是被”加载器“加载的，那BIOS呢？）
2. BIOS被加载到哪里 （程序被加载到内存中，那BIOS呢？
3. 执行BIOS的cs:ip寄存器是谁改的 （当我要执行被载入内存中的那段程序时，有一个call调用指令，会将那段程序的首地址写入cs:ip寄存器中，那么计算机刚启动时，执行的第一个程序是BIOS，那么计算机是怎么找到BIOS的呢？也就是说cs:ip寄存中的值是谁写进去的呢？）



## 二、CPU控制权接力第一棒：BIOS

#### （一）实模式下的1MB内存布局

为什么是1MB？因为intel 8086CPU有20根地址线，所以能够访问的范围是$2^{20}B$，也就是$1MB$。这1MB的地址空间被划分成好几个不同的区域，具体如下表：

| 起始地址 | 结束地址 |    内存大小    |                             用途                             |
| :------: | :------: | :------------: | :----------------------------------------------------------: |
|   000    |   3FF    |      1KB       |             Interrupt Vector Table（中断向量表）             |
|   400    |   4FF    |      256B      |                BIOS Data Area （BIOS数据区）                 |
|   500    |   7BFF   |  30468B约30KB  |                           可用区域                           |
|   7C00   |   7DFF   |      512B      |                MBR被BIOS加载到此处，共512字节                |
|   7E00   |  9FBFF   | 622080B约608KB |                           可用区域                           |
|  9FC00   |  9FFFF   |      1KB       |        EBDA（Extended BIOS Data Area）扩展BIOS数据区         |
|  A0000   |  AFFFF   |      64KB      |                        用于彩色适配器                        |
|  B0000   |  B7FFF   |      32KB      |                        用于黑白适配器                        |
|  B8000   |  BFFFF   |      32KB      |                    用于文本模式显示适配器                    |
|  C0000   |  C7FFF   |      32KB      |                        显示适配器BIOS                        |
|  C8000   |  EFFFF   |     160KB      |            映射硬件适配器的 ROM 或内存映射式 I/O             |
|  F0000   |  FFF3F   |    64KB-16B    | 系统 BIOS 范围是 F0000～FFFFF 共 640KB，为说明入口地址，将最上面的 16 |
|   FFF0   |  FFFFF   |      16B       | 字节从此处去掉了，所以此处终止地址是 0XFFFEFBIOS入口地址。此处16字节的内容是跳转指令`jmp f000:e05b` |

可以看到，我们能够使用的内存区域只有两块，`0x500` ~ `0x7BFF` 和 `0x7E00` ~ `0x9FBFF`，这两个内存区域是一定可用的，如果要使用超过 1M 内存区域之外的内存，这样每个机器的内存大小可能都不一样，所以后面的内存就不一定有了。

实模式下，地址总线有20位，意味着最大可以访问1MB的内存空间，但是我们注意到在实模式下的1MB内存布局中，0-9FFFF是映射到DRAM的，并不是1MB内存都映射到物理内存（内存条）的。

这是因为不仅仅是物理内存需要使用地址总线来进行访问，其他的一些外设也需要使用地址总线来进行访问，如下图：

![image-20220502025727324](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020257393.png)

有上图我们可以得到一个结论：物理内存多大都没用，主要是看地线总线的宽度。



![实模式1MB内存思维导图](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020247937.png)



顶部F0000-FFFFF这64KB的内存是分配给了ROM，ROM中保存的就是BIOS的代码。

BIOS的工作：

1. 检测硬件
2. 初始化硬件
3. 建立IVT中断向量表



#### （二）BIOS是如何苏醒的

BIOS是计算机加电后执行的第一个程序，因此，BIOS不能通过软件的方式进行加载、启动，所以BIOS的启动需要硬件支持。

硬件是怎么做的呢？如下：

计算机加电时，CPU的cs:ip寄存器被强制初始化为`0XF000:FFF0`，经过CPU中的段处理器处理后，得到的最终的物理内存地址是0xFFFF0，从我们上面讲得1MB内存分配中我们知道0xFFFF0正是BIOS的入口地址，此16B的内存空间中有一条跳转指令：`jmp far f000：e05b `，那么CPU的代码执行流就到了0xFE05B处，这里才是BIOS真正的代码存放之地。



为了证明我说的是正确的——BIOS确实是通过这种硬件支持来启动的，我们上图为证：

![image-20220502005703316](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020057421.png)

从上图可以看出，当机器刚开始运行时，cs:ip的值是f000:fff0，我们都知道cs:ip存储的是下一条指令的地址，也就是说，计算机下一条执行的指令是在地址ffff0处的，ffff0处距离fffff处只有16B的空间了，这儿的指令是一条跳转指令`jmp far f000:e05b`。跳转到fe0tb后，就开始顺序的执行代码，计算机执行的代码很显然就是BIOS需要做的一些硬件检测和初始化的工作，在BIOS把这些工作都做完以后，最后一条代码又是一条跳转指令`jmp 0:7c00`，这样的话，CPU的执行流就跳转到0x7c00处去执行代码了，0x7c00开始的512个字节的代码就是mbr中的引导代码。



#### （三）为什么是0x7c00







## 三、入门MBR前需要的前置知识

#### （一）`$`、`$$`、section



#### （二）NASM简单用法



#### （三）引入MBR

编写好mbr.S程序后，输入`sudo nasm -o mbr.bin mbr.S`将会生成一个mbr.bin文件。

输入：`sudo dd if=mbr.bin of=/usr/share/bochs/hd60M.img bs=512 count=1 conv=notrunc`，输出如下：

![image-20220502003010570](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020030813.png)

这样我们的mbr.bin就已经成功地写入到mbr主引导扇区了。



接下来我们就可以运行bochs来看看bios将mbr引导程序装在到0x7c00后，运行的效果了。

输入`sudo bin/bochs -f bochsrc.disk`

![image-20220502003505529](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020035678.png)

我们输入`6`：

由于咱们编译的是可调试的版本，所以会停下来，bochs 等待咱们键入下一步的命令，如图所示：

![image-20220502003706190](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020037376.png)

这个左面的黑框框就是bochs模拟的x86机器。

我们在终端中输入`c`，是continue的意思，bochs模拟的机器就会运行了，运行的效果是：

![image-20220502004229465](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020042827.png)

