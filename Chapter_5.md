# 内核之旅

目前为止，通过使用底层的汇编语言，我们知道了很多计算机是如何工作的知识，不过我们也知道了使用这种语言是多么的低效：我们甚至需要仔细思考最简单的控制流结构，并且我们要关系如何最大限度的使用有限数量的寄存器，然后在栈中挣扎。汇编的另一个缺点是与特定架构的 CPU 绑定太过密切，导致让我们的 OS 在其他 CPU 架构上（比如 ARM、RISC、PowerPC）很难运行起来。

幸运的是，其他程序员也受够了汇编的这些缺点，所以决定写一些高层语言的编译器（比如 FORTRAN、C、Pascal 金额 basic 等），它会将更加直观的代码转换成汇编。这些编译器的思想是将高层的结构，比如控制结构和函数调用，映射到汇编的模版代码，所以它的缺点也就是通用的模版很可能（几乎一定存在）对于某个功能来说不是最优的实现。让我们看看 C 代码是如何被转换成汇编代码的来阐述编译器扮演的角色。

## C 编译器

让我们写一些 C 的代码片断，看看它们会生成怎样的汇编代码。这也是很好的学习 C 是如何工作的一种方式。

### 生成原始机器码

```
// Define an empty function that returns an integer
int my_function () {        return 0xbaba;
}
```

保存上述代码，文件名取为 `basic.c`，然后这样编译它：

```
$gcc -ffreestanding -c basic.c -o basic.o
```

这会生成一个目标文件（object file）。编译器输出标记的机器码，而不是直接编译成机器码，这样，元信息比如文本标签会在执行前保持不变，因为最终代码被合成的时候可以更加灵活。这种中间格式最大的优点是当和其他库中的代码链接时，可以更简单的合到一个更大的二进制文件中，这是因为目标文件中的代码使用的是相对地址而不是绝对的内存地址。可以用下列命令查看目标文件的内容：

```
$objdump -d basic.o
```

下面是上述命令的输出：

```
basic.o: file format elf32 -i386

Disassembly of section .text:

00000000 <my_function >:
  0: 55                   push %ebp
  1: 89 e5                mov %esp,%ebp
  3: b8 ba ba 00 00       mov $0xbaba ,% eax
  8: 5d                   pop %ebp
  9: c3                   ret
```

我们可以看到一些汇编代码和一些额外的代码细节。注意，汇编的语法和 `nasm` 稍微有点不同。忽略这些内容，我们后面会看到最为熟悉的格式。

为了创建真实可执行的代码（比如，可以在 CPU 上执行），我们需要使用链接器，它的作用就是将所有的目标文件中的内容链接在一起生成一个可执行二进制文件。有效的将它们串起来，并将这些相对地址转换成最终机器代码中的绝对地址。比如：`call <function_X_label>` 会变成 `call 0x12345`，其中 0x12345 是最终输出文件中 `function_X_label` 标记代码的对应偏移地址。

不过，我们暂时不需要链接任何其他目标文件（等一下会简单的介绍），不过最终链接器会把标记的机器码转换成真实的二机制代码。可以用下列命令生成一个包含真实机器码的 `basic.bin` 文件：

```
$ld -o basic.bin -Ttext 0x0 --oformat binary basic.o
```

注意，就像编译器一样，链接器可以用很多种格式输出可执行的文件，其中一些可能会保留输入的目标文件中的元数据。对于由操作系统处理的执行，这非常有用，比如我们为 Linux 或者 Windows 平台编写的大部分程序。因为元数据可以被保留用于描述应用是如何被加载到内存中的，并且也可用于调试。比如：CPU 在地址 `0x12345678` 处执行崩溃这种信息对程序员来说几乎无用，如果元数据中包含非执行代码的信息的话，则可能前面的崩溃信息会被转换成在 `my_function` 函数，文件 `basic.c` 中，第 3 行崩溃。

因为我们在写一个 OS，在 CPU 上运行混合了元数据的机器代码没啥好处，因为 CPU 会盲目的将每一个字节当作机器代码执行。这就是为什么上面命令中我们指定了输出格式是 `binary`

另一个我们使用的选项是 `-Ttext 0x0`，这个和我们之前在汇编代码中使用的 `org` 指令的工作方式一样，允许我们告诉编译器如何偏移代码中的标签得到绝对内存地址（比如，任何我们在代码中指定的数据，比如字符串 “Hello, world”），后面加载的时候，就会加载到相应的内存地址处。现在，这个还不重要，不过当我们开始加载内核到内存中时，将这个设置成我们计划加载的内存地址处很重要。

现在，我们成功的将 C 代码编译生成了可执行的二进制文件。让我们看看它长啥样。因为汇编和相应的机器指令十分接近，所以如果有一个包含机器代码的文件，你可以很简单的反汇编查看。这也对理解汇编有点帮助，因为潜在的，你可以逆向在你电脑上的任何一个软件，如果开发者留下了一些元信息的话，你甚至可能看到原始的代码。反汇编机器代码的唯一一个问题是，有时候一些字节已经被指定为数据了，但是展示出来确实汇编指令。可以用如下命令查看编译器从 C 代码中生成的机器码：

```
$ndisasm -b 32 basic.bin > basic.dis
```

`-b 32` 告诉反汇编器用32位汇编指令解码，我们编译器生成的机器代码也是32位的。下图展示了结果：

```
00000000 55           push ebp
00000001 89E5         mov ebp,esp
00000003 B8BABA0000   mov eax , 0xbaba
00000008 5D           mov eax , 0xbaba
00000009 C3           ret
```

可以看出，gcc 生成的汇编代码和我们自己写的差别不是很大。反汇编器输出的3栏，从左到右，分别是文件中的偏移地址，机器代码，和等价的汇编指令。虽然我们的函数做的事情非常简单，不过，这里有一些额外的代码，似乎是用来管理栈的基底和顶部寄存器： `ebp` 和 `esp`。 C 大量的使用栈来存放本地的变量（比如那些函数返回之后不需要了的变量）。所以进入函数的时候，栈的基底指针（`ebp`）被更新了当前的栈顶，在调用我们函数的函数的栈上，有效的创建和初始化了一个新的空栈。这个操作通常被叫做一个函数在设置它的栈帧，后面任何本地变量都会在栈帧里面分配。不过，如何函数返回的时候，恢复调用者的栈帧失败的话，当调用者试图访问它的本地变量的时候，情况会变得一团糟，所以在更新基底指针为我们自己的栈帧之前，我们需要保存它，所以我们将它保存到栈上（`push ebp`）

准备好栈帧之后（不幸的是，这个简单的函数没有用到任何本地变量，所以也就不会使用栈帧了），我们看到编译器是如何处理 `return 0xbaba;` 的：值 0xbaba 被存放在32位寄存器 `eax` 中，这也是调用者希望返回值（如果有的话）被存放的地方。这和我们之前定义我们寄存器来传递参数的情况很像。比如，`print_string` 例程期望在 `bx` 寄存器中找到将被打印的字符串的地址。

最后，再返回到调用者之前，函数将原先的栈帧基底从栈中 pop 出来（`pop ebp`），这样调用者就不会意识到它的栈帧曾被被调用者改过。注意，我们没有改变栈顶（`esp`），因为我们的栈帧中没有存放任何东西，所以没有改动过的 `esp` 寄存器就不需要恢复。

现在，我们知道 C 代码是如何被转换成汇编代码的。让我们进一步了解编译器，直到我们有足够的知识用 C 来写内核。

### 本地变量

新建一个文件 `local_var.c`，代码如下：

```
// Declare a local variable. int my_function () {
  int my_var = 0xbaba;
  return my_var; 
}
```

然后像之前一样编译、链接和反汇编。

编译器生成的汇编代码如下：

```
00000000 55             push ebp
00000001 89E5           mov ebp,esp
00000003 83EC10         sub esp,byte +0x10
00000006 C745FCBABA0000 mov dword [ebp-0x4],0xbaba
0000000D 8B45FC         mov eax ,[ebp -0x4]
00000010 C9             leave
00000011 C3             ret
```

唯一的不同是我们分配了一个本地变量 `my_var`，不过这引发了编译器一个有趣的反应。像前面一样，栈帧被建立起来。`sub esp,byte +0x10` 的意思是将栈顶减去16（0x10）字节。首先，我们要时刻提醒自己，栈是相对于内存地址反方向增长的。这条指令简单的说就是：在栈顶分配16字节。我们在存 `int`，这是个4字节（32位）数据类型，所以为什么对于这个变量是在栈上分配了16字节，为什么不用 `push`（它会自动在栈上分配的新空间）。编译器这么做的原因是出于效率，因为 CPU 操作和内存边界不对齐的数据类型不高效。因为 C 一般会让所有的变量进行对齐，所以它为每个栈元素使用最大的数据类型宽度（比如16字节），代价是浪费一些内存。

下一条指令 `mov dword [ebp-0x4], 0xbaba`，将变量存放在栈上刚分配的空间，但不用 `push`，之前提到是为了效率。我们知道 `mov` 指令的用处，不过，有两点需要解释

- `dword` 显式的表明我们在存放两个字（4字节），也就是 int 类型的大小。所以实际存放的字节是 `0x0000baba`，如果不显式的表示的话，可能存放的是 `0xbaba`（两字节）或者`0x000000000000baba`（8字节）。同样的值，但是长度不一样。
- `[ebp-0x4]` 这是现代 CPU 的一种简写（虽然从汇编代码看起来可能意思不是很明显），被叫做高效地址计算。指令中的一部分计算引用地址是 CPU 在运行时基于当前 `ebp` 寄存器的值进行计算的。咋一看，我们可能觉得汇编器在处理一个常数，因为当我们写比如：`mov ax, 0x5000 + 0x20` 时，我们的汇编器会简单的预处理成 `mov ax, 0x5020`。但这里，只有代码运行的时候才可能知道 ebp 的值，所以这绝对不是预处理。它是 CPU 指令的一部分。用这种地址访问方式，CPU 允许我们在一个指令周期内做的更多，这也是 CPU 硬件迎合开发者的一个例子。我们可以写处等价的，没有地址处理，更低效的下面代码：

```
mov eax, ebp ; EAX = EBP
sub eax, 0x4 ; EAX = EAX - 0x4
mov [eax], 0xbaba ; store 0xbaba at address EAX
```

所以值 `0xbaba` 直接被存到栈的相应位置，所以他会占据基指针上的开始（实际上是下面，因为栈是反方向增长的）4个字节。

作为一个程序，编译器会区分不同的变量名，就像区分不同的数字一样简单，所以当我们说变量 `my_var` 的时候，编译器会想成地址 `ebp-0x4`（栈的开始4字节）。下一条指令 `mov eax, [ebp-0x4]`，意思是，存放 `my_var` 的内容到 `eax` 中，这里也用到了高效地址计算。并且从前面我们知道， `eax` 是用于返回值给调用者的。

在 `ret` 之前，我们看到一个新的东西： `leave` 指令。事实上， `leave` 指令等价于下面的指令：（恢复调用者的栈帧）

```
mov esp, ebp ; Put the stack back to as we found it. 
pop ebp
```

虽然只有一条指令，但是 `leave` 指令有时候比分开来的指令要高效。

### 调用函数

看下面的 C 代码：

```
void caller_function() { 
  callee_function(0xdede);
}
int callee_function(int my_arg) { 
  return my_arg;
}
```

这里有两个函数，第一个函数 `caller_function`，调用另一个函数 `callee_function`，并传递一个 int 参数。被调用函数只是简单的返回它的参数。

编译然后反汇编上述代码，得到类型下面这样的结果：

```
00000000 55               push ebp
00000001 89E5             mov ebp,esp
00000003 83EC08           sub esp,byte +0x8
00000006 C70424DEDE0000   mov dword [esp],0xdede 
0000000D E802000000       call dword 0x14
00000012 C9               leave
00000013 C3               ret
00000014 55               push ebp
00000015 89E5             mov ebp,esp
00000017 8B4508           mov eax,[ebp+0x8]
0000001A 5D               pop ebp
0000001B C3               ret
```

首先，注意我们是如何区分不同函数的汇编代码：通过 `ret` 指令，它总是作为一个函数的最后一条指令出现。其次，注意上面的函数是如何使用汇编指令 `call` 的，这个指令我们知道是用于跳转到另一个例程，并期望从那个例程返回。这个指令一定是 `caller_function`，因为它在 0x14 的机器码偏移处调用 `callee_function`。最有意思的是 call 后面的几行代码，因为它们要保证参数 `my_arg` 被传给 `callee_function`。建立新的栈帧之后，就像前面的一样，`caller_function` 分配 8 字节在栈顶（`sub esp, byte +0x8`)，然后存放我们传过来的值，0xdede，到栈的空间中（`mov dword [esp], 0xdede`）。

`callee_function` 是如何访问参数的呢？从偏移 `0x14`，我们看到 `callee_function` 创建了它的栈帧，不过注意 `eax` 寄存器中存储的内容（这个寄存器前面我们知道是用于保存函数返回值的）：它存放了地址 `[ebp + 0x8]` 的内容，再次提醒一下，栈是相对于内存反方向增长的，所以 `ebp + 0x8` 指的是在栈底下面的8字节，所以我们实际上访问了上一个栈帧的内容来得到参数的值。这是我们期望的结果，因为调用者将参数放到了它自己的栈帧的顶部，而我们将我们自己的栈帧建立在它们之上。

理解高层语言编译器的调用约定对于理解它生成的汇编代码很有帮助。比如，C 默认的参数调用约定是将参数反向放在栈上，所以第一个参数在栈顶。搞乱了参数的次序很可能会导致程序执行错误，进而发生 crash。

### 指针、地址和数据

当使用高层语言，我们经常会忘了一个事实是：变量其实只是分配在内存地址中内容的引用而已，包含足够的空间来适应不同的数据类型。因为，我们处理变量的大部分情况，我们只是关心它们持有的值，而不是它们位于内存的哪里。考虑下面这样的 C 代码：

```
int a = 3;
int b = 4;
int total = a + b;
```

既然我们不知道编译器是如何处理这些 C 代码的，我们可以做一个常规的假设，`int a = 3;` 这个指令包含两部：第一，至少4字节（32位）将被保留（可能在栈上）用于存放值。然后，值`3` 将被存放到保留的地址处。第二行代码情况差不多。对于 `int total = a + b;`，更多的空间将被保留用于变量 `total`，并且将存放 a 和 b 的和的值。

现在，假设我们想在内存的特定地址存放一个值。比如，像我们之前没法用 BIOS 然后用汇编写一个字符直接存放到图像内存的地址 `0xb8000` 中一样。我们如何用 C 来做呢（好像任何我们想要存的值的地址都是由编译器决定的）？事实上，在高级语言中是不允许我们这样做的，因为这几乎是违反了高级语言的抽象。幸运的是！C 允许我们使用指针变量（一种用于存放地址而不是值的数据类型），借此，我们可以读写数据到任何这个变量指向的地方。

技术上，所有的指针变量是同一种数据类型（比如，32位的内存地址），不过通常我们希望读写指针指向的某一特定数据类型的数据。所以我们要告诉编译器，比如，这个指针是指向一个 char 类型的数据，然后那个是指向一个 int 类型的数据。这只是一种方便手段，因为这样我们就不需要总是告诉编译器它要读写指向内存地址的多少字节。定义和使用指针的语法如下：

```
// Here, the star following the type means that this is not a variable to hold 
// a char (i.e. a single byte) but a pointer to the ADDRESS of a char,
// which, being an address, will actually require the allocation of at least 
// 32 bits.
char* video_address = 0xb8000;

// If we’d like to store a character at the address pointed to, we make the 
// assignment with a star-prefixed pointer variable. This is known as
// dereferencing a pointer, because we are not changing the address held by 
// the pointer variable but the contents of that address.
*video_address = ’X’;

// Just to emphasise the purpose of the star, an ommision of it, such as:
video_address = ’X’;
// would erroneously store the ASCII code of ’X’ in the pointer variable, 
// such that it may later be interpretted as an address.
```

在 C 代码中，我们经常看到 `char*` 变量用于字符串，让我们想想为什么。如果我们希望存放单一一个 int 或者 char，那么我们知道它们都是固定大小的数据类型（即，我们知道它们会使用多少字节），不过一个字符串通常是 char 的数组，这种数据类型的长度可以是任意一个长度。所以单一一个数据类型是没法存放完整的字符串的，只能存放它的一个元素。所以，我们可以使用指向 char 的指针，然后设置它为字符串中第一个字符的内存地址。这其实就是我们在之前的汇编代码中做的，比如 `print_string`，这个汇编中，我们在某个地方分配了一个字符串（“Hello, World”），然后为了打印一个字符串，我们通过 `bx` 寄存器传递字符地址。

让我们看一个编译器为我们设置字符串变量的例子。在下面代码中，我们定义了一个简单的函数，只是分配了一个字符串给一个变量：

```
void my_function () {
  char* my_string = "Hello";
}
```

像前面一样，我们可以反汇编出下面这样的代码：

```
00000000 55               push ebp
00000001 89E5             mov ebp,esp
00000003 83EC10           sub esp,byte +0x10
00000006 C745FA48656C6C   mov dword [ebp-0x6],0x6c6c6548
0000000D 66C745FE6F00     mov word [ebp -0x2],0x6f
00000013 C9               leave
00000014 C3               ret
```

首先，看到 `ret` 指令。一般这表示一个函数的结束。可以看到前面两行指令设置了一个栈帧。下一个指令，我们以前也看到过，`sub esp,byte +0x10`，分配16字节在栈上，用于存放本地变量。再下一条指令，`mov dword [ebp-0x4],0xf`，应该很熟悉，因为它存放一个值到变量中。不过为什么它存放数字 `0xf`？我们没有告诉它这样做啊？存放这个之后，我们看到函数正常的将栈恢复（`leave`），然后返回（`ret`）。不过注意，还有5条更多的指令在函数的结尾后面！你觉得 `dec eax` 在做什么？可能它将 `eax` 中的值减少了1，为什么？然后其他指令又是什么意思？

到这里，我们需要记住，反汇编器不会区分代码和数据。所以，代码的某处一定是我们定义的字符串的数据。我们知道我们的函数占据了代码的前半部分，因为这些汇编指令对我们来说很熟悉，并且也 `ret` 结尾。如果我们现在假设代码的其余部分实际上是我们的数据，那么可以的存放在我们的变量中的数字：`0xf`，就很清楚了。因为这是距离我们的代码开始处的数据起始的偏移：我们的指针变量正在被设置为数据的地址。如果我们查看 “Hello” 的 ASCII值，它们是 `0x48`、`0x65`、`0x6c`、`0x6c` 和`0x6f`。现在清楚了，因为看反汇编器输出的中间一栏（一些很奇怪的机器指令），我们看到最后一个字节是 `0x0`，这是 C 自动添加到字符串末尾的（像之前我们的汇编代码 `print_string` 一样，在处理阶段，我们可以容易的检测到我们是否已经到达一个字符串的末尾了）。

## 执行内核代码

理论已经够了，让我们启动和执行用 C 写的最简单的内核吧。这一步会用到所有我们学过的内容，并会加快开发我们自己的 OS 功能的进度。

包含一下几步：

- 编写内核代码，并编译
- 编写启动代码
- 创建一个内核镜像，包含启动代码和已经编译的内核代码
- 加载内核代码到内存中
- 切换到32位模式
- 开始执行内核代码

### 编写内核

这一步不会花太多时间，因为，现在，我们内核的主要功能只是让我们知道它已经成功的加载和执行了。后面我们可以再完善内核，注意保持事情简单很重要。保存下面的代码在文件 `kernel.c` 中：

```
void main () {
  // Create a pointer to a char, and point it to the first text cell of
  // video memory (i.e. the top-left of the screen)
  char* video_memory = (char*) 0xb8000;
  // At the address pointed to by video_memory, store the character ’X’ 
  // (i.e. display ’X’ in the top-left of the screen).
  *video_memory = ’X’;
}
```

用下列的命令编译成二进制文件：

```
$gcc -ffreestanding -c kernel.c -o kernel.o
$ld -o kernel.bin -Ttext 0x1000 kernel.o --oformat binary
```

注意，现在我们告诉链接器一旦加载它到内存中，我们的代码将在 `0x1000` 处，所以它知道基于此纠正内部的偏移地址，就像我们使用 `[org 0x7c00]`（BIOS 会加载到 0x7c00 处然后开始执行）。

### 创建启动代码加载内核

现在我们看是写启动代码，它将会从磁盘加载内核并且执行内核代码。因为内核是用32位指令编译的，我们需要在执行内核代码之前，切换到32位模式。在计算机启动的时候，我们知道 BIOS 会加载我们的启动代码（磁盘的前面512字节），而不会加载我们的内核。不过前面的章节中我们已经知道如何使用 BIOS 磁盘例程来让我们的启动代码加载额外的磁盘扇区内容，并且我们也稍微了解到，当我们切换到32位模式以后，缺少 BIOS 的帮助，我们很难使用磁盘了：我们需要自己写一个软盘驱动或者硬盘驱动！！！

为了简化从哪个磁盘的哪个扇区加载内核代码的问题，启动代码和 OS 的内核可以被放置到一起叫做内核镜像，可以将镜像写到启动磁盘的初始扇区中，这样启动代码将总是在内核镜像的最前面。一旦我们按这一小节的方式编译好了启动代码，我们可以用如下命令创建内核镜像：

```
cat boot_sect.bin kernel.bin > os-image
```

下面代码展示了如何在启动代码中加载内核： `os-image`

```
; A boot sector that boots a C kernel in 32-bit protected mode
[org 0x7c00]
KERNEL_OFFSET equ 0x1000  ; This is the memory offset to which we will load our kernel

  mov [BOOT_DRIVE], dl    ; BIOS stores our boot drive in DL, so it’s
                          ; best to remember this for later.

  mov bp, 0x9000          ; Set-up the stack.
  mov sp, bp

  mov bx, MSG_REAL_MODE   ; Announce that we are starting
  call print_string       ; booting from 16-bit real mode

  call load_kernel        ; Load our kernel

  call switch_to_pm       ; Switch to protected mode, from which
                          ; we will not return

  jmp $

; Include our useful, hard-earned routines 
%include "print/print_string.asm"
%include "disk/disk_load.asm"
%include "pm/gdt.asm"
%include "pm/print_string_pm.asm" 
%include "pm/switch_to_pm.asm"

[bits 16]

; load_kernel
load_kernel:
  mov bx, MSG_LOAD_KERNEL   ; Print a message to say we are loading the kernel
  call print_string

  mov bx, KERNEL_OFFSET     ; Set -up parameters for our disk_load routine , so
  mov dh , 15               ; that we load the first 15 sectors (excluding
  mov dl, [BOOT_DRIVE]      ; the boot sector) from the boot disk (i.e. our
  call disk_load            ; kernel code) to address KERNEL_OFFSET

  ret

[bits 32]
; This is where we arrive after switching to and initialising protected mode.

BEGIN_PM:
mov ebx, MSG_PROT_MODE      ; Use our 32-bit print routine to
call print_string_pm        ; announce we are in protected mode

call KERNEL_OFFSET          ; Now jump to the address of our loaded
                            ; kernel code , assume the brace position ,
                            ; and cross your fingers. Here we go!

jmp $                       ; Hang.

; Global variables
BOOT_DRIVE      db 0
MSG_REAL_MODE   db "Started in 16-bit Real Mode", 0
MSG_PROT_MODE   db "Successfully landed in 32-bit Protected Mode", 0
MSG_LOAD_KERNEL db "Loading kernel into memory.", 0

; Bootsector padding 
times 510-($-$$) db 0 
dw 0xaa55
```

在运行 Bochs 命令之前，确保 Bochs 配置文件有启动磁盘设置到你的内核镜像文件中：

```
floppya: 1_44=os-image, status=inserted 
boot: a
```

你可能会问，为什么我们从启动磁盘中加载15个段（512*15字节），毕竟我们的内核镜像实际上只有1个扇区不到的大小，所以加载1个扇区就够了？原因是读取额外的磁盘扇区并没有什么坏处，即使它们没有被初始化为任何数据，不过在后面，当尝试检测到我们没有读取足够的扇区时，这可能会有害（并增加了内存占用大小）：此时，计算机会没有任何警告的停下来，可能在读取的半途中失败。

如果一个 ‘X’ 在屏幕的左上脚打印了的话，那么恭喜你成功了！虽然看起来没有什么，但是这是我们开始之前的一大步：我们现在已经能启动到高级语言写的代码了，可以考虑更少的汇编编程，能够专注于如何编写我们的 OS，而且当然，再学一点 C 的内容。这是学习 C 的最佳方式，因为 Java 和其他脚本语言（比如 Python、PHP 等）是高级语言的更大抽象。

### 内核编程

