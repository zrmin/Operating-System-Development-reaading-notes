<h1><center>CPU交接：从加电到BIOS再到MBR</center></h1>

## 一、计算机的启动过程

在谈计算机的启动过程之前，我们先来谈一下“载入内存”。

载入内存分为两个阶段：

1. 程序被“加载器”放到内存中
2. cs:ip寄存器指向这段程序的首地址



计算机上电之后，首先执行的是BIOS，那么BIOS是什么：Base Input Output System——基本输入输出系统。

于是我们有3个问题：

1. BIOS是被谁加载的（程序载入内存是被”加载器“加载的，那BIOS呢？）
2. BIOS被加载到哪里 （程序被加载到内存中，那BIOS呢？）
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



#### （二）BIOS是如何被唤醒的（BIOS如何被加载运行的？）

BIOS是计算机加电后执行的第一个程序，因此，BIOS不能通过软件的方式进行加载、启动，所以BIOS的启动需要硬件支持。

硬件是怎么做的呢？如下：

计算机加电时，CPU的cs:ip寄存器被强制初始化为`0XF000:FFF0`，经过CPU中的段处理器处理后，得到的最终的物理内存地址是0xFFFF0，从我们上面讲得1MB内存分配中我们知道0xFFFF0正是BIOS的入口地址，此16B的内存空间中有一条跳转指令：`jmp far f000：e05b `，那么CPU的代码执行流就到了0xFE05B处，这里才是BIOS真正的代码存放之地。



为了证明我说的是正确的——BIOS确实是通过这种硬件支持来启动的，我们上图为证：

![image-20220502005703316](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020057421.png)

从上图可以看出，当机器刚开始运行时，cs:ip的值是f000:fff0，我们都知道cs:ip存储的是下一条指令的地址，也就是说，计算机加电后，计算机下一条执行的指令是在地址ffff0处的，ffff0处距离fffff处只有16B的空间了，这儿的指令是一条跳转指令`jmp far f000:e05b`。跳转到fe05b后，就开始顺序的执行代码，接下来 BIOS 便开始检测内存、显卡等外设信息，当检测通过，并初始化好硬件后，开始在内存中 0x000～0x3FF 处建立数据结构——中断向量表 IVT 并填写中断例程。

在BIOS把这些工作都做完以后，就到了接力的时刻，BIOS已经完成了它的工作，接下来的工作需要把CPU转交给另外的软件，让其他软件来完成响应的工作。这是这场接力赛的第一棒，它将交给谁呢？**所谓的交控制权就是 jmp 过去而已**。

剧透一点点：

**BIOS 最后一项工作——校验启动盘中位于 0 盘 0 道 1 扇区的内容**。如果此扇区末尾的两个字节分别是魔数 0x55 和 0xaa，BIOS 便认为此扇区中确实存在可执行的程序（这个程序就是主引导记录 MBR），便加载到物理地址 0x7c00，随后跳转到此地址，继续执行。

（BIOS的最后一条代码又是一条跳转指令`jmp 0:0x7c00`，CS段寄存器的值由原来的0xf000变成了0。这样的话，CPU的执行流就跳转到0x7c00处去执行代码了，0x7c00开始的512个字节的代码就是mbr中的引导代码。）



#### （三）关于0x7c00的两个疑问

为什么执行完BIOS的代码后，要把MBR引导程序的代码放到0x7c00处呢？为什么不是其他地方呢？

如果我不想告诉你的话，我可以直接说0x7c00是个magic number，是个魔数，就是这么规定的。 原话是：

`OS or bootloader developer must assume that their assembler codes are loaded and start from 0x7C00`。

但是我想告诉你更多。因为我花了一些时间去查阅为什么是0x7c00这个魔数，而不是其他魔数。

这儿有一个网址，大家如果信不过我的解释的话，可以去阅读一下。https://www.glamenv-septzen.net/en/view/6#:~:text=BIOS%20developer%20team%20decided%200x7C00,program%20needed%20more%20512%20bytes.

那么下面我们废话不多说，进入正题。

1. 谁决定了0x7c00

   * IBM PC 5150 BIOS 开发团队，
   * “0x7C00”最早出现在 IBM 公司出产的个人电脑 PC5150 的 ROM BIOS 的 INT19H 中断处理程序中
   * 计算机通电开机之后，BIOS 处理程序开始自检，随后，调用BIOS 中断 0x19h，即 call int 19h。在此中断处理函数中，BIOS 要检测这台计算机有多少硬盘或软盘，如果检测到了任何可用的磁盘，BIOS 就把它的第一个扇区加载到 0x7c00。

2. 0x7c00 = 32KB - 1KB有什么内涵

   * 这个问题其实等价于为什么 IBM PC 5150 ROM BIOS INT 19h 将 MBR 加载到 7C00H (0x7C00)

   * 第一个问题——32KB的由来：

     * 个人计算机肯定要运行操作系统，在IBM PC5150计算机上，运行的操作系统是 DOS 1.0， PC 5150 BIOS 研发工程师David Bradley就假定其是 32KB 的，因此，需要给DOS操作系统留下32KB大小的内存空间。所以此版本 BIOS是按最小内存 32KB 研发的。

   * 第二个问题——-1KB的由来：

     * 给DOS1.0操作系统流出足够的空间来加载，避免MBR自己被过早覆盖导致MBR中的512字节的引导程序无法执行。We wanted to **leave as much room as possible for the OS to load itself within the 32KB**. 

     * 8086CPU 要求物理地址 0x0～0x3FF 存放中断向量表，然后是BIOS数据区，所以要把MBR加载到其他地方。The 808x Intel architecture used up the first portion of the memory range for software interrupts, and the BIOS data area was after it. So we **put the bootstrap load at 0x7C00 (32KB-1KB) to leave all the room in between（在中间留出足够的空间） for the OS to load. **

     * MBR 本身也是程序，是程序就要用到栈，栈也是在内存中的，MBR 虽然本身只有 512 字节，但还要为其所用的栈分配点空间，所以其实际所用的内存空间要大于 512 字节，估计 1KB 内存够用了。The **boot sector was 512 bytes**, and when it executes **it'll need some room for data and a stack**, so that's the other 512 bytes. 

     * So the memory map looks like this after INT 19H executes:

       ![image-20220502152156378](C:\Users\张润民\AppData\Roaming\Typora\typora-user-images\image-20220502152156378.png)

       * 综上 ，选择32KB中的最后1KB最为合适，那此地址是多少呢？32KB换算为十六进制为0x8000，减去 1KB(0x400)的话，等于 0x7c00。这就是倍受质疑的 0x7c00 的由来，这下清楚了。

可见，加载 MBR 的位置取决于操作系统本身所占内存大小和内存布局。



## 三、入门MBR前需要的前置知识

#### （一）`$`、`$$`、section

标号：

```assembly
code_start: ; code_start 这个标号被 nasm 认为是一个地址，此地址便是“mov ax，0”这条指令所在的地址，即mov ax, 0的机器码存放的内存位置是 code_start
; nasm 会用具体的地址来替换标号 code_start，到了 CPU 手中，已经被替换为有意义的数字形式的地址了
mov ax, 0
```

我们上面讲得`code_start`是显式标号，而`$`和`$$`是隐式标号。

1. `$`是本行的地址
2. `$$`是本section的地址

默认情况下，它们的值是相对于本文件开头的偏移量。至于实际安排的是多少，还要看section 的vstart。这个关键字可以影响编译器安排地址的行为，如果该 section 用了 vstart=xxxx 修饰，则：

* $$的值是此 section 的虚拟起始地址xxxx。
* $的值是以 xxxx 为起始地址的顺延。
* 如果用了 vstart 关键字，想获得本 section 在文件中的真实偏移量（真实地址）：section.节名.start。

* 如果没有定义 section，nasm 默认全部代码同为一个 section，起始地址为 0。

3. section 是给开发人员在逻辑上规划代码用的，只起到思路清晰的作用，最终还是在编译阶段由nasm在物理上的规划说了算



#### （二）NASM简单用法

目前只要会一个nasm的用法就可以了：

```assembly
nasm -f <format> <filename> [-o <output>]
```

* -f 是用来指定输出文件的格式

* format有哪些：

  * bin：bin 是默认输出格式，所以不用`-f bin` 来明确指定了，以后咱们只在输出 elf 格式时才用-f 指定
  * elf32
  * elf64
  * elfx32
  * 其他，不需要了解

* -o 是用来指定输出的可执行文件的名称

  实例：
  
  ```shell
  nasm -o mbr.bin mbr.S # 使用mbr.S代码生成mbr.bin文件
  ```
  
  

## 四、初探MBR

编写mbr.S：暂时看一下就行，不要求全部理解，知道这段mbr.S代码的功能是什么就行了，不需要理解代码。理解代码的事情，我们以后会谈）

```assembly
 1 ;主引导程序
 2 ;------------------------------------------------------------ 
 3 SECTION MBR vstart=0x7c00 
 4 mov ax,cs 
 5 mov ds,ax 
 6 mov es,ax 
 7 mov ss,ax 
 8 mov fs,ax 
 9 mov sp,0x7c00 
10 
11 ; 清屏利用 0x06 号功能,上卷全部行,则可清屏｡
12 ; ----------------------------------------------------------- 
13 ;INT 0x10 功能号:0x06 功能描述:上卷窗口
14 ;------------------------------------------------------ 
15 ;输入: 
16 ;AH 功能号= 0x06 
17 ;AL = 上卷的行数(如果为 0,表示全部) 
18 ;BH = 上卷行属性
19 ;(CL,CH) = 窗口左上角的(X,Y)位置
20 ;(DL,DH) = 窗口右下角的(X,Y)位置
21 ;无返回值: 
22 mov ax, 0x600 
23 mov bx, 0x700 
24 mov cx, 0 ; 左上角: (0, 0) 
25 mov dx, 0x184f ; 右下角: (80,25), 
26 ; VGA 文本模式中,一行只能容纳 80 个字符,共 25 行｡
27 ; 下标从 0 开始,所以 0x18=24,0x4f=79 
28 int 0x10 ; int 0x10 
29 
30 ;;;;;;;;; 下面这三行代码获取光标位置 ;;;;;;;;; 
31 ;.get_cursor 获取当前光标位置,在光标位置处打印字符｡
32 mov ah, 3 ; 输入: 3 号子功能是获取光标位置,需要存入 ah 寄存器
33 mov bh, 0 ; bh 寄存器存储的是待获取光标的页号
34 
35 int 0x10 ; 输出: ch=光标开始行,cl=光标结束行
36 ; dh=光标所在行号,dl=光标所在列号
37 
38 ;;;;;;;;; 获取光标位置结束 ;;;;;;;;;;;;;;;; 
39 
40 ;;;;;;;;; 打印字符串 ;;;;;;;;;;; 
41 ;还是用 10h 中断,不过这次调用 13 号子功能打印字符串
42 mov ax, message 
43 mov bp, ax ; es:bp 为串首地址,es 此时同 cs 一致, 
44 ; 开头时已经为 sreg 初始化
45 
46 ; 光标位置要用到 dx 寄存器中内容,cx 中的光标位置可忽略
47 mov cx, 6 ; cx 为串长度,不包括结束符 0 的字符个数
48 mov ax, 0x1301 ;子功能号 13 显示字符及属性,要存入 ah 寄存器, 
49 ; al 设置写字符方式 ah=01: 显示字符串,光标跟随移动
50 mov bx, 0x2 ; bh 存储要显示的页号,此处是第 0 页, 
51 ; bl 中是字符属性,属性黑底绿字(bl = 02h) 
52 int 0x10 ; 执行 BIOS 0x10 号中断
53 ;;;;;;;;; 打字字符串结束 ;;;;;;;;;;;;;;; 
54 
55 jmp $ ; 使程序悬停在此
56 
57 message db "My MBR" 
58 times 510-($-$$) db 0 
59 db 0x55,0xaa
```



编写好mbr.S程序后，输入`sudo nasm -o mbr.bin mbr.S`将会生成一个mbr.bin文件。（不需要-f bin指定输出的文件格式，因为bin格式是默认的）

我们生成了mbr.bin文件，那么如何将这个mbr.bin文件写入 0 盘 0 道 1 扇区？——使用dd命令。

```shell
if=FILE 
read from FILE instead of stdin
# 此项是指定要读取的文件。
```

```shell
of=FILE 
write to FILE instead of stdout 
# 此项是指定把数据输出到哪个文件。
```

```shell
bs=BYTES 
read and write BYTES bytes at a time (also see ibs=,obs=) 
# 此项指定块的大小，dd 是以块为单位来进行 IO 操作的，得告诉人家块是多大字节。
# 此项是统计配置了输入块大小 ibs 和输出块大小 obs。这两个可以单独配置
```

```shell
count=BLOCKS 
copy only BLOCKS input blocks 
# 此项是指定拷贝的块数。
```

```shell
seek=BLOCKS 
skip BLOCKS obs-sized blocks at start of output 
# 此项是指定当我们把块输出到文件时想要跳过多少个块。
```

```shell
conv=CONVS 
convert the file as per the comma separated symbol list 
# 此项是指定如何转换文件。
```

```shell
append append mode (makes sense only for output; conv=notrunc suggested) 
# 这句话建议在追加数据时，conv 最好用 notrunc 方式，也就是不打断文件
```



输入：`sudo dd if=mbr.bin of=/usr/share/bochs/hd60M.img bs=512 count=1 conv=notrunc`，输出如下：

结合实例分析：

* if=mbr.bin  输入文件是我们刚刚编译生成的mbr.bin
* of=/usr/share/bochs/hd60M.img  输出文件是我们虚拟出来的硬盘 hd60M.img
* bs=512  块大小是512字节
* count=1  操作的块数是1块，即总共 1*512=512 字节



执行`sudo dd if=mbr.bin of=/usr/share/bochs/hd60M.img bs=512 count=1 conv=notrunc`命令后，输出如下：

![image-20220502003010570](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020030813.png)

这样我们的mbr.bin就已经写进 hd60M.img 的第 0 块了。



ps: hd60M.img是我们前面使用`bin/bximage -hd -mode="flat" -size=60 -q hd60M.img`这个命令创建的启动盘。

回忆一下上节课，当我们通过`bin/bochs `启动bochs时，提示报错：`could not read the boot disk`——找不到启动盘。

![image-20220501132501146](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205011325559.png)

于是我们通过`bin/bximage -hd -mode="flat" -size=60 -q hd60M.img`命令创建了启动盘hd60M.img。这个启动盘的容量是60M。我们的mbr.bin写入它的前512个字节。



接下来我们就可以运行bochs来看看bios将mbr引导程序装载到0x7c00后，运行的效果了。

输入`sudo bin/bochs -f bochsrc.disk`

![image-20220502003505529](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020035678.png)

我们输入`6`：

由于咱们编译的是可调试的版本，所以会停下来，bochs 等待咱们键入下一步的命令，如图所示：

![image-20220502003706190](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020037376.png)

这个左面的黑框框就是bochs模拟的x86机器。

我们在终端中输入`c`，是continue的意思，bochs模拟的机器就会运行了，运行的效果是：

![image-20220502004229465](https://raw.githubusercontent.com/zrmin/BlogImages/master/images/202205020042827.png)



## 五、总结

也就是说当计算机加电启动时，cs:ip被强制赋值为：`0xF000:0xfff0`，然后CPU执行地址0xfff0处的那16字节的跳转指令`jmp far f000：e05b`，于是CPU执行0xfe05b处的BIOS代码，于是BIOS开始检测硬件、初始化硬件、创建IVT中断向量表，然后将hd60M.img中的前512个字节的代码加载到0x7c00处开始执行。然后就是执行我们上面写的mbr.bin的代码（本质是mbr.S中的代码）。于是最终在屏幕上打印出“My MBR”这个字符串。
