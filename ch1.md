# 引言

>操作系统的基本目标：让应用与硬件隔离，简化应用访问硬件的难度和复杂性。实现这些功能的操作系统就是一个函数库，可以被应用访问，并通过函数来访问硬件。

**生成应用程序二进制执行代码所依赖的是以编译器为主的开发环境**
还有链接器，linker.ld文件决定代码每一段（代码段、数据段）会在内存的哪个物理地址上。
**运行应用程序执行代码所依赖的是以操作系统为主的执行环境**
对于普通APP，的确依赖操作系统
但是对于操作系统，依赖的是裸机硬件
对于内核来说：执行环境是以 硬件 + 极简运行时 为主的

**极简运行时**
全套运行时：
写hello world的时候，程序不是从main函数开始执行，在main函数之前，rust的标准库会塞一堆代码进来，这就是运行时。
这些代码的工作：
- 内存管理：准备好堆内存分配器，不然用不了String和Vec。
- 线程管理：准备好主线程的环境
- 异常处理：准备好panic发生时的打印堆栈、回溯机制
- 命令行参数：解析argv和argc
进行完这些工作，才会调用main。所以叫全套运行时。

极简运行时：
对于操作系统内核来说，代码要在裸机上启动。没有操作系统、没有标准库。也就没有了分配堆内存、管理线程等。
在按下电源键时，CPU醒来，状态是混乱的，如内存还没清零，栈指针还没设置，BSS段还没初始化。
BSS段是存放全局变量的地方。
如果这个时候直接跳转到代码里，比如想存个局部变量，或者访问全局变量，会立刻因为内存错乱而崩溃。
所以需要一段精简的代码，插在硬件启动和程序代码之间，这段代码就是极简运行时。
在rCore里，这段代码是汇编语言编写的，叫entry.asm
它的工作是：
- 设置栈
- 清空BSS段：将存放全局变量的内存全部填0
- 跳转Jump：跳转到rust_main

所以rCore的第一章，就是手搓这个精简运行时，好让Rust代码跑起来

# 应用程序执行环境与平台支持

`Cargo.toml`文件中保存项目的配置信息。

## 应用程序执行环境
![ch1-001](ch1-001.md)

应用程序、函数库、内核/操作系统、硬件平台，每一层都是一个执行环境。
API、ABI、ISA是相邻执行环境之间的接口。
越靠下越贴近底层，下层作为上层的执行环境。

>在我们通常不会注意到的地方，这些软件库还会在执行应用之前完成一些初始化工作，并在应用程序执行的时候对它进行监控。

比如main函数并不是程序的第一个执行点，在main之前，还有如初始化堆、清空BSS等。因为要写裸机程序，没有标准库做这些初始化工作，所以需要手写汇编来完成。

>在打印 Hello, world! 时使用的 println! 宏正是由 Rust 标准库 std提供的。

`println!` 不是语言特性，而是库函数，依赖std库，而std库又依赖于操作系统的`write`系统调用，所以在`#![no_std]`环境下，`println!` 宏没了，需要自己实现，让其通过SBI接口往屏幕上实现打印。

>内核作为用户态软件的执行环境，它不仅要提供系统调用接口，还需要对用户态软件的执行进行监控和管理。

内核还要有监控和管理的能力，对应后面的`Trap处理`和`任务调度`，需要编写代码来捕获App的错误，或者强行打断App的运行。

## 目标平台和目标三元组

> 现代编译器工具集的主要工作流程
>1. 源代码——》预处理器——》宏展开的源代码
>2. 宏展开的源代码——》编译器——》汇编程序
>3. 汇编程序——》汇编器——》目标代码
>4. 目标代码——》链接器——》可执行文件



>Rust编译器通过 目标三元组 (Target Triplet) 来描述一个软件运行的目标平台。它一般包括 CPU、操作系统和运行时库等信息，从而控制Rust编译器可执行代码生成。

实验的目标平台是`riscv64gc-unknown-none-elf`
- CPU架构是riscv64gc
- CPU厂商unknow
- 操作系统是none
- elf表示没有标准的运行时库，但可以生成ELF格式的执行程序

Rust有一个core库，core库不需要操作系统的支持。其中包含了一部分rust的核心机制，可以满足大部分需求。

# 移除标准库依赖

## 移除std库依赖

实验需要`rustc`编译器缺省生成RISC-V64的目标代码，先给`rustc`添加target
```bash
rustup target add riscv64gc-unknown-none-elf
```

可以使用如下命令查看target是否正确
```bash
rustup target list --installed
```
配置成功的话，可以正常看到`riscv64gc-unknown-none-elf`
以后编译项目时，只要加上 `--target riscv64gc-unknown-none-elf` 参数，它就生成 RISC-V 的程序了。

也可以在os目录下新建`.cargo`目录，目录下新建`config.toml`文件，内容如下
```toml
[build]
target = "riscv64gc-unknown-none-elf"
```
这样就默认了target，`cargo build`无需额外加参数

此时`cargo build`会报错，报错信息是没找到std标准库。
```
error[E0463]: can't find crate for `std`
```

解决的方法是，在src/main.rs文件中，添加`#![no_std]`属性来告知编译器不适用std库，转而使用core库。

## 提供panic_handler功能

再次`cargo build`出现报错，因为在使用std库时，程序出现panic时，标准库会帮助打印错误信息、回溯堆栈、然后退出程序。这个过程是`Panic处理机制`。
```
error: `#[panic_handler]` function required, but not found
```

因为现在没有std库， 编译器不知道该如何处理panic，需要手写一个函数，并给它贴上`#[panic_handler]`属性，告知编译器出现panic就找它。

在`os/src/`目录下新建一个语言项文件`lang_items.rs`
```rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

## 修改main函数

再次cargo build，这次的报错信息如下
```
error: using `fn main` requires the standard library
```

在rust中，main()函数并不是程序真正的第一个执行点。真正的入口是标准库中一个叫start的语言项，标准库在做完初始化以后，才会调用main。
现在程序里没有标准库，编译器不知道该如何处理main函数的调用。因为在写操作系统内核，不需要按照指定的那套标准规则去执行，可以自己定义一个真正的入口。
解决方法：
1. 加上`#![no_main]`属性，告知编译器不需要找`main`函数了，入口由自己接管。
2. 移除`main`函数，定义一个真正的入口`_start`，这个名字符合C语言调用约定，为了方便汇编调用，也是链接器默认找的名字。

```rust
#![no_std]
#![no_main]

mod lang_items;

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

## 分析被移除标准库的程序

// todo

# 内核第一条指令

## Qemu

如何启动Qemu，在各个章节的`os/Makefile`文件中

启动流程：

内存布局
![ch1-002](ch1-002.md)

代码部分：代码段存放所有的汇编代码。
数据部分：
- .rodata常量数据段：只读，存放常量、常量字符串
- .data已初始化的数据段：可读写，存放写代码时赋了初值的全局变量
- .bass未初始化的数据段：可读写，存放声明了但是未赋初值的全局变量
- heap堆：存放程序运行时动态分配的数据，向高地址增长
- stack栈：用于函数调用上下文的保存和恢复，存放局部变量，向低地址增长。

// todo 不理解
局部变量：存放在栈或寄存器，访问方式是sp寄存器+偏移量
全局变量：存放在.data/.bss，访问方式是gp寄存器+偏移量
动态变量：存放在堆，间接访问，先读取栈/全局段上的指针，再跳到堆地址

# 内核第一条指令 实践

```
    .section .text.entry
    .globl _start
_start:
    li x1, 100
```

RISC-V架构的CPU里有32个寄存器，分别是x0，x1，...，x31，CPU处理任何数据，都要先将数据放进寄存器里。

.section是`段`，是目标文件内部组织数据的最小逻辑单元。
常见段有：`.text`，`.data`，`.bss`，存放内容和前文一样
`.section .text.entry`的含义是：定义了一个自定义段，段名是`.text.entry`
单独定义这个段，是为了控制内存布局的顺序。
当定义这个段后，后面所有的代码都会放在这个段内，这种状态会持续到定义下一个段。
后面和链接器脚本配合（linker.ld），会把目标文件中，名为`.text.entry`的段，全部收集起来，放在最终二进制文件的开头。

globl：将一个符号的可见性标记为全局可见。
默认情况下，汇编里面的标签是local的，只在当前.asm文件内有效。
加上globl后，这个符号就被写进了目标文件的**符号表**
链接器在进行符号解析时，只能看见全局符号。它需要找到程序的入口点，默认的入口名字就是`_start`。
如果不把`_start`设为全局可见（也称导出），编译器就找不到入口会报错。

`_start:`：符号标记的位置，冒号`:`是定义标签的语法，代表的就是内存地址，假设代码被加载到了0x8000，那么_start就等于0x8000。

`li x1, 100`是伪指令
真指令：CPU硬件电路真正实现了的指令，如addi
伪指令：是汇编器提供的语法糖，为了方便程序员编写代码
汇编器会自动把`li x1, 100`翻译为`addi x1, x0, 100`把x0加上100，结果存入x1，因为x0的值恒为0，所以结果就是100。
为什么使用伪指令：如果立即数很大，一条addi放不下，这时改写为li，汇编器会自动拆成lui + addi两条指令，屏蔽了底层指令长度限制的复杂性。

什么是符号表：
符号表的本质是一个数据结构，存在编译出来的目标文件中，记录了代码里所有名字和地址的对应关系。
作用：链接器要把目标文件链接在一起时，遇到函数调用，如call main，就回去查表，然后把call指令的目标填上main函数在符号表中的地址。这就是链接的过程。


```rust
use core::arch::global_asm;
global_asm!(include_str!("entry.asm"));
```
Rust编译器不认识汇编文件，只认识rs文件，那么`entry.asm`怎么编译进去？
解决方法：嵌入汇编
嵌入汇编：告知编译器在编译的时候，把entry.asm这个文件里的文本内容，原封不动的粘贴到这个，并且把它当成全局汇编代码处理。
include_str!：这是一个宏，会在编译器读取entry.asm的内容，把它变成字符串。
global_asm!：这个内联汇编宏，接收一个字符串，然后将字符串扔给底层的汇编器。


在默认情况下，Rust编译器rustc和链接器linker有自己的一套规则，会把程序随便放在内存的某个地方。但是要写的是OS，必须自己掌控内存的内容。

```toml
 [target.riscv64gc-unknown-none-elf]
 rustflags = [
     "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
 ]
```

`[target.riscv64gc-unknown-none-elf]`：意思是下面的配置只对目标平台生效。
`rustflags = [...]`：`[...]`的内容是要传递给编译器的额外参数。
`"-Clink-arg=-Tsrc/linker.ld"`：让编译器告知链接器，使用我脚本里写的规则。
- `-C`：Codegen（代码生成）选项。
- `link-arg`：告知编译器，后面的参数转交给链接器
- `-T`：指定Script脚本
- `src/linker.ld`：要传递的脚本文件
`"-Cforce-frame-pointers=yes"`：强制生成**帧指针**，作用是方便以后的调试。
如果不生成帧指针，编译器为了优化性能，可能会把记录函数调用链的fp寄存器另作他用。如此，一旦程序崩溃，很难查出问题。生成栈指针，虽然性能微跌，但是可以完美回溯堆栈。

总结：告知Rust工具链，用我的linker.ld来安排内存地址，保留栈指针，以便查Bug。


下面代码的作用就是：把代码里的各种段，按什么顺序、摆在什么位置。
`.`：当前地址指针
`*`：通配符，代表所有的。如`*(.text)`就表示文件里所有的代码段。
`ALIGN(4K)`：对齐，摆放完一个代码段，必须凑整，以4096字节的倍数。

```
OUTPUT_ARCH(riscv)           # 设置目标平台为riscv
ENTRY(_start)                # 程序的入口为_start
BASE_ADDRESS = 0x80200000;   # 从0x80200000地址开始

SECTIONS
{
    . = BASE_ADDRESS;        # 此时,指向0x80200000地址
    skernel = .;             # start kernel,表示内核从此开始

    stext = .;               # start text,表示.text代码段的起点
    .text : {                # .text表示段名,与{}内的意义不同
        *(.text.entry)       # 先将.text.entry代码摆放在这里,确保位置在最前面
        *(.text .text.*)     # 然后将剩余的.text代码放在其后
    }
    . = ALIGN(4K);           # 凑整,把指针位置指向4K对齐的地址
    etext = .;               # end text,表示.text代码段终点
    
    srodata = .;             # 只读数据段的起点
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    . = ALIGN(4K);
    erodata = .;
    
    sdata = .;               # 可读写数据段的起点
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    . = ALIGN(4K);
    edata = .;
     
    .bss : {                 # 未初始化数据段的起点
        *(.bss.stack)        # 先放内核栈,这是rCore特有的安排,这里不会被清零
        sbss = .;            # 插旗 需要清零的区域的起点
        *(.bss .bss.*)       # 放普通的未初始化全局变量
        *(.sbss .sbss.*)     # 放小变量
    }
    . = ALIGN(4K);
    ebss = .;                # 插旗 清零区域结束
    
    ekernel = .;             # 内核结束

    /DISCARD/ : {
        *(.eh_frame)         # 将其他不需要的调试信息扔掉
    }
}
```

空白的纸，最上面画个线条，说这是入口，下面再画一条短一点的线，这条线叫text，下面是text段的代码，先放.text.entry，剩下的随意，然后把大小凑成整数倍，然后画条线，说text到这结束了。下面同理。


然后去到os目录，编译程序
```bash
cargo build --release
```

使用file工具查看属性
```bash
file target/riscv64gc-unknown-none-elf/release/os
```

结果如下：
```
target/riscv64gc-unknown-none-elf/release/os: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, not stripped
```
- ELF 64-bit：格式是ELF，是linux中通用的可执行文件格式。64位。
- LSB：字节序是小端序
- executable：可执行文件
- UCB RISC-V：架构，UCB代表UC Berkeley，是给RISC-V处理器跑的
- RVC：特性，RISC-V的缩写，说明它启用了压缩指令集，能把部分32位指令压缩16位
- double-float ABI：浮点，它支持双精度浮点运算，对应了target名字里面的gc
- statically linked：链接方式是静态链接，意味着没有任何外部依赖
- not stripped：状态，未去符号，说明文件里还保留着所有的函数名、变量名。对于调试GDB非常重要，但是在最后生成`.bin`镜像时，这些信息会被丢弃。

// todo 0x80200000可否改为其他地址

// todo 静态链接和动态链接


## 手动加载内核可执行文件

错误做法：直接加载ELF
因为ELF由两部分组成，metadata元数据和sections段。
其中sections段的内容才是CPU要执行的指令和读写的数据。
Qemu的-device loader是不具备解析ELF格式的能力，只会将整个ELF文件视作一个连续的二进制数据块，逐字节赋值到内存的目标地址`BASE_ADDR`，CPU就会将ELF头的数据作为指令来解码和执行，会导致错误。

正确做法：加载剥离后
是圆通rust-objcopy工具，从ELF文件中提取出所有可加载的段（由medatada元数据中的程序头表定义），按照它们在虚拟内存中的布局，拼接成一个连续的、不含元数据的二进制文件，也就是.bin文件。这个文件的第一个字节，就是链接脚本放在BASE_ADDR处的.text.entry段的第一条指令。Qemu会将.bin文件加载到目标地址。

使用如下命令可以丢弃内核可执行文件中的元数据得到内核镜像
```
rust-objcopy --strip-all target/riscv64gc-unknown-none-elf/release/os -O binary target/riscv64gc-unknown-none-elf/release/os.bin
```

然后比较两个os文件，一个os是ELF文件，一个os.bin是最后生成的内核镜像
```bash
stat target/riscv64gc-unknown-none-elf/release/os
  # File: target/riscv64gc-unknown-none-elf/release/os
  # Size: 1312            Blocks: 8          IO Block: 4096   regular file

stat target/riscv64gc-unknown-none-elf/release/os.bin
  # File: target/riscv64gc-unknown-none-elf/release/os.bin
  # Size: 4               Blocks: 8          IO Block: 4096   regular file
```
os.bin内核镜像的大小仅为4个字节，因为在entry.asm文件中有效的指令只有`li x1, 100`，这条伪指令翻译为真指令为`addi x1, x0, 100`，RISC-V架构中，一条标准的指令长度就是4字节。

## 基于GDB验证
调试需要用到gdb，安装在环境配置中，安装解压后，要进行其他配置
**注意** 教程全程使用release模式进行构建，为了正常进行调试，请确认各项目（如 os , user 和 easy-fs ）的 Cargo.toml 中包含如下配置：
```
[profile.release]
debug = true
```
否则在后面调试的时候，会如下：
```
No debugging symbols found in target/riscv64gc-unknown-none-elf/release/os
```
虽然打开了调试的文件，但是里面没有调试信息。就是因为在cargo build时，用的是--release模式，并且没有开启调试符号，release模式为了追求性能，默认会把调试信息扔掉。
解决方法就是修改toml文件后，重新`cargo build --release`

在os目录下通过如下命令启动Qemu并加载RustSBI和内核镜像os.bin
```
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ../bootloader/rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000 \
    -s -S
```
-s可以使Qemu监听本地TCP端口1234等待GDB客户端连接。
-S可以使Qemu在收到GDB的请求后再开始运行。
所以执行完此条命令，Qemu没有任何输出。如果想直接运行，只需要去掉-s -S即可

// todo

# 为内核支持函数调用

如何使得函数返回时能够跳转到调用该函数的下一条指令？
通过ra寄存器（return address），函数调用前，先把下一条指令的地址存进ra寄存器，当调用的函数结束，直接执行jr ra跳转到ra指向的地址。

如何保证寄存器内容不变？
情况1：调用者caller保存
在函数调用前，先将寄存器内容压入栈中，等函数调用结束，从栈中恢复。
情况2：被调用者callee保存
函数调用期间，寄存器变化前， 先保存寄存器内容，使用完寄存器进行恢复。

在RISC-V上如何划分？
ra用于返回地址，caller负责保存
sp用于栈指针，callee负责保存
t0-t6是临时寄存器，caller负责保存
a0-a7是参数/返回值寄存器，caller负责保存，用于传参
s0-s11由callee负责保存

如何传递参数？
富裕时：前8个参数，依次存在a0-a7寄存器
返回值：放在a0（和a1）里
穷困时：参数>8个时，寄存器放不下了，放在栈上。

>将由于函数调用，在控制流转移前后需要保持不变的寄存器集合称之为 **函数调用上下文**。

>在调用子函数之前，我们需要在物理内存中的一个区域 保存 (Save) 函数调用上下文中的寄存器；而在函数执行完毕后，我们会从内存中同样的区域读取并 恢复 (Restore) 函数调用上下文中的寄存器。







