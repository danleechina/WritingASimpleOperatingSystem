# 设备驱动和文件系统

## 硬件输入输出

将东西输出到屏幕的时候，我们实际上已经遇到关于硬件 I/O 的问题了，这也被叫做内存映射 I/O，在这里数据被直接写到某一个主内存地址范围中，实际上是被写到屏幕设备的内部内存缓存中。现在我们需要了解更多这类 CPU 和硬件设备的交互。

让我们用现在流行的 TFT 显示器为例。屏幕的显示面板被划分成一个矩阵，每一个元素是一个背光单元。液晶层在偏振光和背光单元之间，形成类似三明治的形状，光穿透每个单元的数量可以被电场控制，因为液晶有种属性，当电场发生变化时，它们也会以相应的方式发生变化。当液晶发生变化时，它们会引起光波的震动方向的变化，这样有些光就会被显示面板的偏振片给挡住。为了展示颜色，每个背光单元进一步被划分成3个区域，每个区域过滤一种颜色的光，比如红、蓝、绿。

根据电场的变化保证每一个单元以及颜色单元发生相应的变化，以此在屏幕上呈现出不同的颜色是硬件的责任。这些是电子工程师的责任，不过这里也会有控制芯片，一般都会附带完善的使用手册。这种芯片可以在设备上，也可以在主板上，然后 CPU 可以借助它来控制该设备。现实中，为了向后兼容，TFT 显示器通常也可以模拟老式的 CRT 显示器，所以可以被主板上的标准 VGA 控制器驱动。VGA 可以生成复杂的模拟信号，使用电子束来扫描荧光屏，不过因为并没有真实的 CRT 电子束可以使用，TFT 设备会聪明的将模拟信号解读为数字信号。

控制芯片一般有几个寄存器可以被读写（甚至被 CPU 读写），寄存器的状态指导控制芯片做什么（比如，要执行内部的什么功能）。举个例子，从 Intel 广泛使用的 `82077AA` 单芯片软盘控制器的使用手册中，我们可以发现有一个 PIN 识别码（识别码57，标签为 `MEO`）驱动了第一个软盘设备的马达（单个控制器可以驱动好几个类似设备）：当识别码被打开时，马达开始转动；当被重置时，马达停止转动。这个识别码的状态是和控制器内置寄存器（叫做数字输出寄存器（digital output register，DOR））的某个特定位绑定的。通过设置寄存器的值来设置其每一位的状态（比如软盘驱动这个是第4位），across the chip’s data pins, labelled DB0--DB7, and using the chip’s register selection pins, A0--A2, to select the DOR register by its internal address 0x2.

### I/O 总线

虽然出于历史原因 CPU 可以直接和设备控制器交流，但是会导致 CPU 降低运行速度来匹配低速的设备。所以对 CPU 更实用的做法是通过在高层的高速总线上发出 I/O 指令来指导芯片控制器工作。然后总线控制器负责兼容不同设备的速度，将指令传输给特定设备的控制器。为了避免高层总线为慢速的设备降速，另一个总线控制器可能被加进来充当设备，这就是现代计算机的总线层次结构。

### I/O 编程

现在问题是，我们如何读写设备控制器的寄存器呢（即控制设备的工作）？在 Intel 的架构系统中，设备控制器的寄存器会被映射到 I/O 地址空间（这些地址空间独立于主内存的地址空间），然后使用 I/O 指令 `in` 和 `out` 来从 I/O 地址读写数据，这些 I/O 地址映射到不同的控制器寄存器中。比如，前面提到的软盘控制器通常将 `DOR` 寄存器映射到 I/O 地址 `0x3f2`，所以可以通过下列代码来控制马达：

```
mov dx, 0x3f2     ; Must use DX to store port address
in al, dx         ; Read contents of port (i.e. DOR) to AL 
or al, 00001000b  ; Switch on the motor bit
out dx, al        ; Update DOR of the device.
```

在老式系统中，比如 Industry Standard Architecture（ISA，工业标准架构）总线，端口地址会被静态赋值给设备，不过现代的即插即用总线，比如 Peripheral Component Interconnect （PCI 互连外围设备），BIOS 可以在启动系统之前动态的分配 I/O 地址给大多数设备。这种动态分配需要设备通过总线告知配置信息比如：要为寄存器保留多少 I/O 端口，需要多少内存映射空间，以及给这个硬件类型分配一个唯一的 ID。这些信息用于 OS 找到合适的驱动。

I/O 端口的一个问题是我们无法用 C 语言表达这些底层的指令。所以我们需要了解一些内敛汇编的知识：大部分编译器允许你注入一块汇编代码到函数的主体中，gcc 的实现如下：

```
unsigned char port_byte_in(unsigned short port) {
  // A handy C wrapper function that reads a byte from the specified port 
  // "=a" (result) means: put AL register in variable RESULT when finished 
  // "d" (port) means: load EDX with port
  unsigned char result;
  __asm__("in %%dx, %%al" : "=a" (result) : "d" (port));
  return result;
}
```

注意这里的汇编 `in %%dx, %%al` 看起来有点奇怪。这是因为 gcc 使用不同的汇编语法（名字叫做 GAS），目标和目的操作数和 nasm 语法一样，`%` 被用于表示一个寄存器，不过需要 `%%`，因为 `%` 在 C 语言中是一个逃逸字符，所以用 `%%` 才能表示一个 `%` 字符。

因为这些底层端口的 I/O 函数会被我们内核中的大部分的硬件驱动使用，所以我们可以将它们收集到一个文件中 `kernel/low_level.c`：

```
unsigned char port_byte_in(unsigned short port) {
// A handy C wrapper function that reads a byte from the specified port 
// "=a" (result) means: put AL register in variable RESULT when finished 
// "d" (port) means: load EDX with port
  unsigned char result;
  __asm__("in %%dx, %%al" : "=a" (result) : "d" (port));
  return result;
}

void port_byte_out(unsigned short port, unsigned char data) { 
  // "a" (data) means: load EAX with data
  // "d" (port) means: load EDX with port
  __asm__("out %%al, %%dx" : :"a" (data), "d" (port));
}

unsigned short port_word_in(unsigned short port) { 
  unsigned short result;
  __asm__("in %%dx, %%ax" : "=a" (result) : "d" (port)); 
  return result;
}

void port_word_out(unsigned short port, unsigned short data) { 
  __asm__("out %%ax, %%dx" : :"a" (data), "d" (port));
}
```

### DMA

因为 I/O 端口用于直接读写字节数据，介于磁盘设备和内存的大量数据交换很可能会花费大量的 CPU 时间。这个问题可以用一个工具来帮助 CPU 完成这些单调乏味的任务，也就是 直接内存访问 (DMA,direct memory access) 控制器。

DMA 的一个好的比喻是，一个建筑师想要将一堵墙从一个地方移到另一个地方，建筑师清楚的知道需要做什么，不过她还要考虑很多其他重要的事情，而不是搬每一个砖头。所以她可以指挥一个建筑工人去搬砖头，然后在完成或者出现了一些问题导致被打断的时候告知她即可。

## 屏幕驱动

目前我们的内核支持在屏幕的角落打印字符 ‘X’。这足够我们知道我们的内核是否已经被成功的加载或者执行，不过并没有告诉我们更多别的信息。

