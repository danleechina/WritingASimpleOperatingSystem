>原文地址 [Writing a Simple Operating System — from Scratch](http://www.cs.bham.ac.uk/~exr/lectures/opsys/10_11/lectures/os-dev.pdf)

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

我们知道通过向显示缓存地址 0xb8000 的内部写入字符即可在屏幕上展示出来，不过我们不想在内核开发的时候一直考虑这些底层的事情。如果我们可以创建一个屏幕抽象层允许我们写 `print("Hello")` 或者 `clear_screen()`这样的代码的话会很不错。如果在一行无法容纳要打印的字符的时候能够滚动到下一行，这样就更好了！这种抽象不仅能让展示内核执行信息的打印变得容易，而且也允许我们以后很容易的用另外一个显示驱动来替换当前的（可能当前的计算机不支持我们当前使用的 VGA 的文字模式）。

### 理解显示设备

与其他硬件比起来，显示设备相当的直接。因为作为内存映射的设备，我们不需要理解任何和控制信息以及硬件 I/O 相关的内容。不过，一个有用的需要 I/O 控制来操作（也就是通过 I/O 端口）的屏幕设备是光标。这对用户很有用，因为它可以引起用户的注意，提醒他们输入一些文本。并且我们也会使用它来作为内部的标记，不管光标是否可以见，程序员不需要总是为在屏幕上打印字符串设置特定的坐标。比如，如果我们写 `print("hello")`，每一个字符都会被打印到屏幕上的特定地方。

### 基本显示驱动实现

虽然我们可以在 `kernel.c`（它包含一个内核的入口函数， `main()`） 中完成所有代码，但是将这些有特定功能的代码组织到它自己的文件会更好，然后这些可以被编译和链接到我们的内核代码，最终效果和将所有代码放到一个文件中是一样的。让我们在 `drivers` 中新建一个新的驱动实现文件 `screen.c`，和一个驱动接口文件 `screen.h`。由于在 makefile 中使用了通配符，`screen.c`（同一目录下的其他 C 文件也一样） 会被自动编译链接到内核中。

首先，让我们在 `screen.h` 定义下列的常量，可以加强我们的代码的可读性：

```
#define VIDEO_ADDRESS 0xb8000
#define MAX_ROWS 25
#define MAX_COLS 80

// Attribute byte for our default colour scheme. 
#define WHITE_ON_BLACK 0x0f

// Screen device I/O ports 
#define REG_SCREEN_CTRL 0x3D4 
#define REG_SCREEN_DATA 0x3D5
```

然后，思考我们将如何写一个函数 `print_char(...)`，将一个字符展示在屏幕的特定的行和列中。我们会在驱动内部使用这个函数（也就是是私有的），我们驱动的公开接口函数（也就是我们希望外部代码使用的函数）会基于此来实现。我们知道视频内存只是一个简单的内存特定范围地址，在这里，每一个字符单元用两个字节表示，第一个字节是字符的 ASCII 值，第二个字节是字符属性的值，允许我们设置每个字符单元的不同的颜色模式。下面代码展示了我们是如何定义这么一个函数的：使用了另外一些我们将要定义的函数（`get_cursor()`、`set_cursor()`、`get_screen_offset()` 以及 `handle_scrolling()`）。

```
/* Print a char on the screen at col, row, or at cursor position */
void print_char(char character, int col, int row, char attribute_byte) {
  /* Create a byte (char) pointer to the start of video memory */ 
  unsigned char *vidmem = (unsigned char *) VIDEO_ADDRESS;

  /* If attribute byte is zero, assume the default style. */ 
  if (!attribute_byte) {
    attribute_byte = WHITE_ON_BLACK; 
  }

  /* Get the video memory offset for the screen location */ 
  int offset;
  /* If col and row are non-negative, use them for offset. */ 
  if (col >= 0 && row >= 0) {
    offset = get_screen_offset(col, row);
  /* Otherwise, use the current cursor position. */ 
  } else {
    offset = get_cursor (); 
  }
  
  // If we see a newline character, set offset to the end of 
  // current row, so it will be advanced to the first col
  // of the next row.
  if (character == ’\n’) {
    int rows = offset / (2* MAX_COLS );
    offset = get_screen_offset(79, rows);
    // Otherwise, write the character and its attribute byte to 
    // video memory at our calculated offset.
  } else {
    vidmem[offset] = character;
    vidmem[offset+1] = attribute_byte; 
  }

  // Update the offset to the next character cell, which is 
  // two bytes ahead of the current cell.
  offset += 2;
  // Make scrolling adjustment, for when we reach the bottom 
  // of the screen.
  offset = handle_scrolling(offset);
  // Update the cursor position on the screen device.
  set_cursor(offset); 
}
```

先实现这些函数中最简单的函数：`get_screen_offset`。这个函数会将行列坐标转换成展示字符单元的距离视频内存开始处的偏移地址。这个转换很简单，注意每一个单元占2个字节。比如，我想要在行3和列4的位置设置一个展示字符，那么这个字符单元会在距离视频内存偏移 488 处（(3 * 80 (i.e. the the row width) + 4) * 2 = 488）。所以 `get_screen_offset` 函数看起来这样：

```
// This is similar to get_cursor, only now we write 
// bytes to those internal device registers. 
port_byte_out(REG_SCREEN_CTRL , 14);
port_byte_out(REG_SCREEN_DATA , (unsigned char)(offset >> 8));
port_byte_out(REG_SCREEN_CTRL , 15);
```

现在看看光标控制函数，`get_cursor()` 和 `set_cursor()`，这两个函数会通过 I/O 端口设置显示控制器的寄存器。使用特定的视频设备 I/O 端口来读写它内部光标相关的寄存器，下面是显示：

```
void set_cursor(int offset) {
  offset /= 2; 
  // Convert from cell offset to char offset. 
  // This is similar to get_cursor, only now we write
  // bytes to those internal device registers.
  cursor_offset -= 2*MAX_COLS;

  // Return the updated cursor position.
  return cursor_offset; 
}

int get_cursor () {

  // The device uses its control register as an index 
  // to select its internal registers, of which we are 
  // interested in:
  // reg 14: which is the high byte of the cursor's offset
  // reg 15: which is the low byte of the cursor's offset
  // Once the internal register has been selected, we may read or
  // write a byte on the data register.

  port_byte_out(REG_SCREEN_CTRL , 14);
  int offset = port_byte_in(REG_SCREEN_DATA) << 8; 
  port_byte_out(REG_SCREEN_CTRL , 15);
  offset += port_byte_in(REG_SCREEN_DATA);
  // Since the cursor offset reported by the VGA hardware is the 
  // number of characters, we multiply by two to convert it to 
  // a character cell offset.
  return offset *2;
}
```

现在，我们有一个函数允许我们在屏幕的特定位置打印一个字符，并且这个函数封装了所以硬件相关的细节。通常，我们不会想在屏幕上一个个打印字符，而是一整条字符串，所以让我们创建一个友好的函数，`print_at(...)`，这个函数会带一个指向字符串第一个字符的指针（即 `char *`），然后一个接一个的打印每一个字符。如果传给函数的坐标是 `(-1, -1)` ，那么它会从当前的光标位置开始打印。代码如下：

```
void print_at(char* message, int col, int row) {
  // Update the cursor if col and row not negative. 
  if (col >= 0 && row >= 0) {
    set_cursor(get_screen_offset(col, row)); 
  }

  // Loop through each char of the message and print it. 
  int i = 0;
  while(message[i] != 0) {
  print_char(message[i++], col, row, WHITE_ON_BLACK); 
  }
}
```

简单起见，为了避免老是使用 `print_at("hello", -1, -1)`，可以定义一个函数 `print`，代码如下所示：

```
void print(char* message) {
  print_at(message , -1, -1);
}
```

另一个有用的简单函数是 `clear_screen(...)`，它会通过在每一个位置写一个空白字符来清理屏幕。代码如下：

```
void clear_screen () { 
  int row = 0;
  int col = 0;

  /* Loop through video memory and write blank characters. */ 
  for (row=0; row<MAX_ROWS; row++) {
    for (col=0; col<MAX_COLS; col++) { 
      print_char(’ ’, col, row, WHITE_ON_BLACK);
    } 
  }

  // Move the cursor back to the top left.
  set_cursor(get_screen_offset(0, 0)); 
}
```

### 滚动屏幕

如果你希望在光标到达屏幕底部的时候自动滚动屏幕，你要记住你必须自己实现这个。这个通常会被忘记，因为屏幕滚动是很自然的，以至于我们以为是理所当然的。不过在底层，我们有对硬件的完全控制，所以我们必须自己实现这个功能。

为了在到底部的时候，让屏幕看上去滚动，我们必须将每一个字符单元向上移动一行，然后清理最后一行，用于新的一行输入（即本来将要被写到屏幕外的那一行）。这意味着，顶部的一行将被第二行覆盖，所以顶部的一行将永远的丢失了，不过我们并不关心这个，因为我们目标是让用户看到计算机最近的活跃信息。

一个不错的想法来实现滚动是在增加光标的位置 `print_char` 之后立即调用一个函数，这个函数我们定义它为 `handle_scrolling`。 `handle_scrolling` 主要用于保证当光标的视频内存偏移被增加到超出屏幕最后一行的时候，所有的行都会向上移动，然后光标被重新定位于最后一个可见的行（也就是新的一行）。

移动一行等价于拷贝它的所有字节（2字节的字符单元，一行80个单元），到上一行的地址处。对此，增加一个通用目的的函数 `memory_copy` 会很有用。因为这个函数很有可能在其他地方也会被用到，所以我们把它加到文件 `kernel/util.c` 中。`memory_copy` 函数会携带源地址和目的地址，以及需要拷贝的字节数，然后循环的一行行拷贝，代码如下：

```
/* Copy bytes from one place to another. */
void memory_copy(char* source, char* dest, int no_bytes) {
  int i;
  for (i=0; i<no_bytes; i++) {
    *(dest + i) = *(source + i); 
    }
  }
```

使用 `memory_copy` 函数的代码如下：

```
/* Advance the text cursor, scrolling the video buffer if necessary. */ 
int handle_scrolling(int cursor_offset) {

  // If the cursor is within the screen, return it unmodified. 
  if (cursor_offset < MAX_ROWS*MAX_COLS*2) {
    return cursor_offset; 
  }

  /* Shuffle the rows back one. */ 
  int i;
  for (i=1; i<MAX_ROWS; i++) {
    memory_copy(get_screen_offset(0,i) + VIDEO_ADDRESS,
                get_screen_offset(0,i-1) + VIDEO_ADDRESS,
                MAX_COLS *2);
  }

  /* Blank the last line by setting all bytes to 0 */
  char* last_line = get_screen_offset(0,MAX_ROWS -1) + VIDEO_ADDRESS; 
  for (i=0; i < MAX_COLS*2; i++) {
    last_line[i] = 0;
  }

  // Move the offset back one row, such that it is now on the last 
  // row, rather than off the edge of the screen.
  cursor_offset -= 2*MAX_COLS;

  // Return the updated cursor position.
  return cursor_offset; 
}
```

## 处理中断

## 键盘驱动

## 硬盘驱动

## 文件系统

