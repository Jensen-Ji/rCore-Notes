用rust写一个操作系统。
首先，在开发过程中，要舍弃std库的使用，因为std库依赖于操作系统。
而要写的操作系统，是运行在裸机环境上的，无法提供操作系统服务。

先mkdir新建一个目录，起名叫rCore。进入到目录下
使用`cargo new os`新建一个rust项目
其代码在`os/src/main.rs`文件中，内容如下所示
```rust
fn main() {
    println!("Hello, world!");
}
```

# 目标平台和目标三元组

rust编译器通过目标三元组来描述软件运行的目标平台。
该实验的目标平台是`riscv64gc-unknown-none-elf`
- riscv64gc：架构
- unknown：厂商名
- none：操作系统，这里代表没有标准的运行时库，或者没有OS
- elf：二进制接口/格式，表示生成的可执行文件遵循ELF格式标准，且不依赖特定OS的ABI。

实验需要rustc编译器生成RISC-V64的目标代码
先在项目的终端给中rustc添加target
```bash
$ rustup target add riscv64gc-unknown-none-elf
```

然后使用如下命令查看target是否正确配置
```bash
$ rustup target list --installed
riscv64gc-unknown-none-elf
x86_64-unknown-linux-gnu
```
配置成功的话，可以看到`riscv64gc-unknown-none-elf`
以后在编译项目时，只要加上`--target riscv64gc-unknown-none-elf`指定目标，就可以生成RISV-V程序了。

这样每次手动指定太麻烦，所以在os目录下新建`.cargo/config.toml`文件，内容如下
```toml
[build]
target = "riscv64gc-unknown-none-elf"
```
这样在`cargo build`时，就默认指定了target。

实验中需要用到gdb进行调试，教程全程使用的release模式进行构建。
在调试时，就会出现如下信息
```
No debugging symbols found in target/riscv64gc-unknown-none-elf/release/os
```
虽然打开了调试的文件，但是文件里没有调试的信息
因为在cargo build时，虽然用的是release模式，但是没有开启调试符号，release模式为了追求性能，默认会把调试信息扔掉。

所以为了正常进行调试，在各个项目的`Cargo.toml`文件中增加如下内容
```bash
[profile.release]
debug = true
```

从现在开始，所有的构建，都为`cargo build --release`

# 移除标准库依赖

回看src/main.rs文件
在rust中，不使用std库，需要在文件内容前面添加`#![no_std]`属性。
其中`println!`宏也依赖于操作系统，也无法使用，所以要删除那一行。

所以，这个文件就变成了如下：
```rust
#![no_std]
fn main() {

}
```

使用cargo build进行编译会报错
```bash
error: `#[panic_handler]` function required, but not found
```

在使用std库时，程序出现panic时，panic!宏会触发栈回溯并打印错误信息。
因为现在没有std库，编译器不知道如何处理panic。
解决方法是手写一个处理函数，并给它标记`#[panic_handler]`属性，告知编译器，出现panic就找这个函数。

新建一个`os/src/lang_items.rs`文件，将处理panic函数写在这里
先在`os/src/main.rs`中
```rust
#![no_std]

mod lang_items;

fn main() {}
```
然后编写lang_items.rs文件
```
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

什么是language items，是语言项。
语言项就是rust编译器硬性要求必须存在的函数或者类型，如果不提供，编译器就无法生成完整的二进制代码。
panic函数就是必须提供的功能组件，所以放在lang_items文件中。

为什么要新起一个lang_items.rs文件，是代码组织习惯
在技术上，可以把panic函数写在main.rs里，只要加上`#[panic_handler]`属性，编译器都能找到它。
在工程上，随着内核越来越大，main.rs应该负责主要核心启动逻辑。

为什么要在main.rs中先mod lang_items，是Rust模块系统
C/C++的规则是，只要文件在文件夹里，编译器就能找到。
Rust的规则是，哪怕新建了文件，但是没有在main.rs中写mod，rust编译器会完全无视这个文件。

再次cargo build --release构建，报错信息如下
```
error: using `fn main` requires the standard library
```
Rust中的main函数不是程序真正的起点，在有std库时，真正的入口是start语言项，它初始化运行后才调用main。
现在裸机中没有std，编译器不知道如何启动，也不支持标准的main。
解决方法：
1. 禁用main：加上`#![no_main]`属性，告知编译器我会自定义入口。
2. 定义一个`rust_main`，手写一个函数作为Rust代码的起始点。
**物理入口**：`_start`，在entry.asm中，这是cpu执行的第一行代码。
**逻辑入口**：`rust_main`，在main.rs中，这是业务逻辑的起点，也就是内核的起点。
**连接点**：`entry.asm`会执行`call rust_main`

修改main.rs文件
```rust
#![no_std]
#![no_main]

mod lang_items;

#[unsafe(no_mangle)]
pub extern "C" fn rust_main() -> ! {
    loop {}
}
```

`#[unsafe(no_mangle)]`属性，禁止编译器修改函数名字，默认情况下，rust编译后的函数名会变成乱码，加上这个属性，编译后函数名就是rust_main，汇编代码才能call它。
注：`#[unsafe(no_mangle)]`是rust新版本的写法，如果是旧版本使用`#[no_mangle]`

pub：在裸机环境下，有没有pub对程序的运行没有区别。
在实验中，只有汇编代码会调用rust_main函数。
汇编代码不懂rust语法，它只认识符号，只要对象文件的符号表里有一个叫rust_main的全局符号，汇编就能跳转到rust_main。
`#[unsafe(no_mangle)]`的作用就是告知编译器，把函数的名字保持不变，并作为全局符号放到符号表中。
只要有`#[unsafe(no_mangle)]`，无论有无pub，这个函数在二进制层面都是全局可见的，汇编链接器都能找到它。
加上pub之后，语义明确，表明了这个函数是内核的公共入口。

extern "C"，使用C语言的调用约定，确保汇编在调用它时，参数传递和栈的操作方式是兼容的。

现在再次cargo build --release构建即可成功。
空的内核就构建完成了。

# 内核如何启动

内核是运行在Qemu上的，如何让内核对接到Qemu模拟器上？

使用软件`qemu-system-riscv64`来模拟64位的RISC-V架构的计算机。
它的具体配置均通过执行参数选项来调整。

使用如下命令来启动Qemu并运行内核
```bash
$ qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ../bootloader/rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
```
`-machine virt`：表示模拟名为virt的64位RISC-V架构的计算机。
`-nographic`：表示模拟器不需要提供图形界面，只需要对外输出字符流。
`-bios`：设置qemu模拟器开机时用来初始化的引导加载程序（bootloader），这里使用预编译好的`rustsbi-qemu.bin`作为引导加载程序，它需要放在与os同级的`bootloader`目录下。
`-device`：通过loader属性可以在模拟器开机之前将宿主机上的文件载入到qemu内存的指定位置。file属性设置载入文件的路径，addr属性设置载入的物理地址。这里载入的os.bin是前文最小内核可执行文件os的后续，最小内核经过处理就能得到os.bin。

启动流程分为四步：
首先是加载，Qemu模拟的virt计算机，物理内存的起始地址是0x80000000。通过前面的命令启动qemu，在Qemu开始执行任何指令前，会先把两个文件加载到qemu的物理内存中。将rustsbi-qemu.bin加载到0x80000000。将os.bin加载到0x80200000。

然后是固件启动， qemu会执行固化在qemu内部的一段汇编程序，PC值为0x1000，这是qemu CPU上电复位后的默认值，执行几条最基本的指令，然后跳转到物理地址0x80000000，也就是rustsbi所在的位置，这个0x80000000是写死在qemu里的。

然后是引导加载程序，也就是执行rustsbi，PC值为0x80000000，进行硬件初始化、建立系统服务（如字符输出），RustSBI还负责将CPU从M模式切换到S模式，然后跳转到0x80200000，也就是os.bin所在的位置，Rustsbi约定好了内核必须在0x80200000。

最后是内核启动，执行os.bin，pc值为0x80200000，执行第一条指令_start，这个指令在entry.asm中，然后设置栈指针sp，然后跳转到rust_main，开始运行操作系统逻辑。

为什么需要entry.asm，而不是直接启动编写的最小内核直接。

因为Rust函数的执行依赖于栈，而栈的初始化无法在Rust内部完成。
Rust编译器生成的代码假设栈指针SP已经指向了合法的内存区域，而在内核启动的瞬间，硬件并未自动设置好栈。因此需要一段手写的汇编代码entry.asm作为极简运行时，负责在执行Rust代码前建立栈环境。
SP指针的变化，在执行RustSBI时，SP指向RustSBI内部的栈，属于M模式资源，在跳转进入内核时，也就切换到了S模式，此时SP寄存器的值是过时的，这是一个特权级越界的危险指针，不可用，必须将SP重置为属于内核的栈顶地址。

# 内存布局和栈

内存布局分为两部分，数据部分和代码部分
代码部分：
- .text代码段存放所有的汇编代码
数据部分：
- .rodata常量数据段，只读，存放常量、常量字符串
- .data已初始化数据段，可读写，存放写代码时赋了初值的全局变量。
- .bss未初始化数据段，可读写，存放声明了但是未赋初值（或赋值为0）的全局变量。且在程序执行前，这段内存会被自动清零。
- heap堆，存放程序运行时动态分配的数据，向高地址增长。
- stack栈，用于函数调用上下文的保存和恢复，存放局部变量，向低地址增长。
![ch1-002](ch1-002.md)

变量存储位置
局部变量：存在栈或寄存器，访问方式是SP寄存器+偏移量
全局变量：存放在.data/.bss，访问方式是GP寄存器+偏移量
动态变量：存放在堆，间接访问，先读取栈/全局段上的指针，再跳到堆地址。

RISC-V架构的CPU有32个寄存器，CPU处理任何数据，都要先将数据放进寄存器里。

entry.asm代码如下：
```
    .section .text.entry
    .globl _start
_start:
    la sp, boot_stack_top
    call rust_main

    .section .bss.stack
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16

    .globl boot_stack_top
boot_stack_top:
```

RISC-V架构的栈，是从高地址向低地址增长的。
所以在初始化SP指针时，必须把它指向最高的那个地址。

`.section`是段，是目标文件内部组织数据的最小逻辑单元
常见的段有.text，.data，.bss，存放的内容在前文提过。

`.section .text.entry`的含义是：定义了一个段，段名是.text.entry。
单独定义这个段，是为了控制内存布局的顺序
当定义这个段后，后面的代码都会放在这个段内，直到定义下一个段。
后面和链接器脚本linker.ld配合，会把目标文件中名为.text.entry的段，全部放在最终二进制文件的开头。

`.globl _start`：将`_start`符号的可见性标记为全局可见
默认情况下，汇编里面符号是local的，只在当前.asm文件内有效
加上globl后，这个符号就被写进了目标文件的**符号表**。
链接器在进行符号解析时，只能看见全局符号，它需要找到程序的入口点，默认的入口名字就是_start
如果不将_start设置为全局可见，编译器就找不到入口，会报错。

`_start :`：符号标记的位置，冒号`:`是定义标签的语法，代表的就是内存地址，假设代码被加载到了0x80200000，那么_start就对于0x80200000。

`la sp boot_stack_top`：设置SP指向栈顶。
`la`：是一个伪指令，RISC-V的单条指令长度固定是32位，装不下64位的完整地址。所以编译器会把la替换成两条真指令，auipc加载高位+addi加载低位。

`.space`：占空间

为什么要把栈放在.bss段，前面说.bss存放未初始化数据，为什么栈要放在这
如果把栈放在.data段，这64kb的全0数据就会占满ELF文件体积。
放在.bss段，在ELF文件里只要记录需要64kb，不占磁盘空间。
虽然转成bin后还是占的。

什么是符号表
符号表的本质是一个数据结构，存在编译出来的目标文件中，记录了代码里所有名字和地址的对应关系。
作用：链接器要把目标文件链接在一起，遇到函数调用时，如call命令，就会去查表，然后把call指令的目标填上函数在符号表中的地址，这就是链接的过程。

# 嵌入汇编

栈的创建和初始化写好了，但是Rust编译器不认识汇编文件，只认识rs文件。那么entry.asm怎么编译进去
解决方法：嵌入汇编
嵌入汇编：告知编译器在编译的时候，把entry.asm这个文件的文本内容，原封不动的粘贴到Rust代码的全局作用域中，并且把它当成全局汇编代码处理。

在main.rs文件中加入以下代码：
```rust
use core::arch::global_asm;
global_asm!(include_str!("entry.asm"));
```
include_str!：这是一个宏，会在编译期读取entry.asm的内容，把它变成字符串。等同于直接在rs文件里写汇编，不过现在这样更优雅。
global_asm!：这个内联汇编宏，接收一个字符串，然后将字符串扔给底层的汇编器。

为什么用`global_asm!`而不是`asm宏!`
`asm!`是局部内联汇编，它嵌入在某个函数内部，它的限制是函数的执行需要栈帧，但是entry.asm的任务正是初始化栈。
`global_asm!`全局汇编，它定义的汇编代码是脱离在任何Rust函数之外的，不依赖栈或上下文。

# 链接脚本

在默认情况下，Rust编译器和链接器生成的是适用于通用操作系统的程序，其内存布局是动态或者虚拟的。
但是要写的是OS，必须自己掌控内存的内容，需要强制将内核的第一条指令固定在0x80200000，因此就需要编写链接脚本。

修改.cargo/config.toml文件，通过传递参数调整链接器行为。
```toml
[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]
```
`[target.riscv64gc-unknown-none-elf]`表示下面的配置仅对目标平台生效。
`rustflags = [...]`：`[...]`的内容是要传递给编译器的额外参数。
`"-Clink-arg=-Tsrc/linker.ld"`：让编译器告知链接器，使用脚本的规则。
- -C：Codegen（代码生成）选项
- link-arg：告知编译器，后面的参数转交给链接器
- -T：指定Script脚本
- src/linker.ld：要传递的脚本文件
`"-Cforce-frame-pointers=yes"`：强制生成**帧指针**，作用是方便以后的调试。
如果不生成帧指针，编译器为了优化性能，可能会把记录函数调用链的fp寄存器另作他用。这样的话，如果函数崩溃，很难查出问题。生成栈指针，虽然性能微跌，但是可以完美回溯堆栈。
没有FP时，栈帧之间没有固定的链表，编译器只知道当前栈顶在哪里，一旦发生panic，就回不去上一层的函数了。
有FP时，FP寄存器就是导航，当前函数的FP保存着调用者函数的FP位置，只要顺着FP找下去，就能把整个调用栈串起来。

告知Rust工具链，用提供的脚本来安排内存地址，保留栈指针来查Bug。

然后就是编写脚本了

```
OUTPUT_ARCH(riscv)
ENTRY(_start)
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
    . = ALIGN(4K);
    etext = .;
    
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    . = ALIGN(4K);
    erodata = .;
    
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    . = ALIGN(4K);
    edata = .;
    
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }
    . = ALIGN(4K);
    ebss = .;
    
    ekernel = .;
    
    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

这个脚本的作用就是，把代码里的各种段，按照顺序摆在正确位置
`.`：当前地址的指针，代表当前数据填充到内存的哪个地址了。
`*`：通配符，代表所有的，`*(.text)`就表示文件里的所有代码段。
`ALIGN(4K)`：对齐，摆放完一个代码段，必须凑整，凑成4096字节的倍数。

`OUTPUT_ARCH(riscv)`：确认架构
`ENTRY(_START)`：告知GDB调试器，程序的第一条指令在哪里。Qemu其实不看这个，但是调试有用。
`BASE_ADDRESS = 0x80200000`：定义一个常量

`. = 0x80200000`： 此时指针指向0x80200000
`stext = .` 给0x80200000起了个符号叫stext，现在stext就代表这个地址
`.text : {...}`：塞入一堆代码，占用了一些字节，此时.会跟着变化
`etext = . `：给当前地址起名叫etext，etext以后就代表这个地址

`.text : { *(.text.entry) *(.text .text.*) }`
冒号左边`.text`，这是输出端，是指最终生成的os.bin文件里，这个大段的名字。
冒号右边`{}`内，这是输入段，是指从各个目标文件里抓取来的段。
创建一个叫.text的大空间，先在目标文件里把叫.text.entry的文件放在最前面，然后再把其他所有目标文件里叫.text的放在后面。

为什么要`ALIGN(4K)`
为了后续的分页机制，后续会将.text段设为只读、可执行。.data段设为可读写、不可执行。
RISC-V的页表权限控制粒度是4KB，如果代码和数据挤在同一个页面里，就没法单独给它们设置不同的权限了。

为什么sbss放在.bss.stack后面？
目的是让后续的clear_bss函数在清零内存时，自动跳过内核栈，只清零真正的全局变量部分。

```
/DISCARD/ : {
        *(.eh_frame)
    }
```
Rust编译器默认会生成.eh_frame用于堆栈展开（处理panic时的报错信息）。
但是在内核里，手写了panic处理，用不上这个复杂的标准机制。
为了减小内核的体积，直接丢弃

![ch1-006](ch1-006.md)

现在链接脚本完成了，去`cargo build --release`，生成os

# 手动加载内核可执行文件

使用file工具查看刚才生成的文件
```bash
$ file target/riscv64gc-unknown-none-elf/release/os
target/riscv64gc-unknown-none-elf/release/os: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, with debug_info, not stripped
```
- ELF 64-bit：格式是ELF，是linux中通用的可执行文件格式。64位。
- LSB：字节序是小端序
- executable：可执行文件
- UCB RISC-V：架构，UCB代表UC Berkeley，是给RISC-V处理器跑的
- RVC：特性，RISC-V压缩指令集的缩写，说明它启用了压缩指令集，能把部分32位指令压缩16位。能显著减小内核的体积，从而提高Cache的命中率。
- double-float ABI：浮点，它支持双精度浮点运算，对应了target名字里面的gc
- statically linked：链接方式是静态链接，意味着没有任何外部依赖
- not stripped：状态，未去符号，说明文件里还保留着所有的函数名、变量名。对于调试GDB非常重要，但是在最后生成`.bin`镜像时，这些信息会被丢弃。

此时，如果直接用qemu加载该程序
如果是真实机器上，会崩溃。因为CPU会直接从0x80200000取指令执行，但是ELF文件开头是文件头（Header），里面只有一堆描述信息，而不是汇编指令。CPU会把Header数据当指令去执行，触发非法指令异常，直接崩溃。
在Qemu上，Qemu比较聪明，自带ELF解析器，如果使用`-kernel os`启动，qemu会自动读取ELF头，把代码搬运到正确的内存位置，跳过Header。
但是为了模拟真实的流程，以及配合RustSBI，需要把纯净的代码烧录到内存。

怎么从ELF剥离到需要的bin文件
使用rust-objcopy工具，这步过程叫提取二进制镜像。
ELF文件是给调试器GDB看的，内容包含文件头、段表、符号表等元数据。
Bin是给CPU跑的，里面只有指令和数据。这个文件的第一个字节，就是链接脚本放在BASE_ADDR处的.text.entry段的第一条指令。Qemu会将.bin文件加载到目标地址。

使用如下命令可以丢弃内核可执行文件中的元数据得到内核镜像
```
$ rust-objcopy --strip-all target/riscv64gc-unknown-none-elf/release/os -O binary target/riscv64gc-unknown-none-elf/release/os.bin
```

可以使用stat工具比较ELF和Bin文件的大小。
```
$ stat target/riscv64gc-unknown-none-elf/release/os
$ stat target/riscv64gc-unknown-none-elf/release/os.bin
```

在os目录下通过如下命令启动Qemu并加载RustSBI和内核镜像os.bin
```
$ qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ../bootloader/rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000 \
    -s -S
```
-s可以使Qemu监听本地TCP端口1234等待GDB客户端连接。
-S可以使Qemu在收到GDB的请求后再开始运行。
所以执行完此条命令，Qemu没有任何输出。如果想直接运行，只需要去掉-s -S即可

# 工程化构建

使用Makefle，把编译、裁剪、运行这一套流程简化成简单的命令
首先在和os同目录下创建bootloader目录，然后安装rustsbi。
在os目录下，创建Makefile文件
```
# --- 变量定义 ---
# 目标平台架构
TARGET := riscv64gc-unknown-none-elf
# 编译模式 (release / debug)
MODE := release
# 内核 ELF 文件路径
KERNEL_ELF := target/$(TARGET)/$(MODE)/os
# 内核 Bin 文件路径
KERNEL_BIN := $(KERNEL_ELF).bin
# Bootloader 路径
BOOTLOADER := ../bootloader/rustsbi-qemu.bin

# --- 命令封装 ---

# 默认目标：只编译
all: build

# 1. 编译：执行 cargo build
build:
	@echo "Building kernel..."
	cargo build --$(MODE)

# 2. 转换：依赖 build，生成 .bin 文件
binary: build
	@echo "Stripping binary..."
	rust-objcopy --strip-all $(KERNEL_ELF) -O binary $(KERNEL_BIN)

# 3. 运行：依赖 binary，启动 QEMU
run: binary
	@echo "Running QEMU..."
	qemu-system-riscv64 \
		-machine virt \
		-nographic \
		-bios $(BOOTLOADER) \
		-device loader,file=$(KERNEL_BIN),addr=0x80200000

# 4. 调试：依赖 binary，启动 QEMU 并开启 GDB 监听
debug: binary
	@echo "Running QEMU in debug mode..."
	qemu-system-riscv64 \
		-machine virt \
		-nographic \
		-bios $(BOOTLOADER) \
		-device loader,file=$(KERNEL_BIN),addr=0x80200000 \
		-s -S

# 清理构建
clean:
	cargo clean

# 声明伪目标 (防止目录下有同名文件导致冲突)
.PHONY: all build binary run debug clean
```

此后，可以使用make build，make run等命令直接构建或运行内核

# 初始化栈

现在内核以及可以在qemu虚拟机上运行了。

但是.bss段还未初始化，现在初始化.bss。
在main.rs里写一个clear_bss函数
```rust
fn clear_bss() {
    unsafe extern "C" {
        fn sbss();
        fn ebss();
    }

    let start_addr = sbss as *const () as usize;
    let end_addr = ebss as *const () as usize;

    for a in start_addr..end_addr {
        unsafe {
            let ptr = a as *mut u8;
            ptr.write_volatile(0);
        }
    }
}
```
`fn sbss()`：这里的sbss虽然声明为函数，但是不用调用它，而是要获取它在内存中的位置。ebss同理。

`let ptr = a as *mut u8`：把数字地址转换为裸指针（`*mut u8`）。

`ptr.write_volatile(0)`：为什么不写`*ptr = 0`，如果这样写，编译器可能会自作聪明，它发现我向内存写入了数据，但是并没有读取这块数据，就会觉得这行代码多余，然后把整个循环当空气。Volatile的作用是告诉编译器，不管后面有没有用到这里，比如实打实的写入。

# 函数调用

从call rust_main这一刻，栈的生命开始了

涉及三个角色：
- 硬件（CPU）：执行指令，操作寄存器
- 调用者（caller）：发起调用的函数或汇编代码
- 被调用者（callee）：被调用的函数或编译器生成的代码

从RustSBI跳转到0x80200000，也就是_start处，开始执行指令
`la sp, boot_stack_top`：SP指针指向栈顶，栈的内容是全空的或者是乱码。

`call rust_main`：此时CPU在操作，栈没有变化，call指令不压栈，它只改变寄存器。ra寄存器保存了下一条指令的地址（call rust_main的下一条指令），pc寄存器直接跳到了rust_main的第一行代码。

此时跳到了rust_main的第一行，编译器生成的代码开始执行。
编译器可能会用到寄存器，但是它不能破坏之前的数据，还要保存回家的路（ra寄存器的值）。
编译器生成的汇编就会进行如下操作
1. 开辟空间，`addi sp, sp, -16`，假设分配16个字节，那么SP会从`top`下降到`top-16`。这16个字节就是rust_main的栈帧。
2. 保存现场
-  `sd ra, 8(sp)`，功能是保存返回地址。此时是rust_main在操作，也就是软件在操作，把CPU存在ra寄存器里的地址，也就是回家的路抄写在栈上。
- `sd s0, 0(sp)`，功能是保存旧的栈底指针FP，虽然这里_start没有FP。

然后rust_main会调用clear_bss，此时pc指向的指令是`call clear_bss`
CPU再次将当前的pc+4的地址写入ra寄存器，也就是call clear_bss的下一条指令。此时ra的值就被覆盖了，不过刚才rust_main将其保存在了栈中。

然后CPU进入了clear_bss，新建栈帧开始构建，开辟空间，SP继续下降。
保存ra也就是回rust_main的地址，也就是call clear_bss的下一条指令。
保存s0寄存器，也就是rust_main的栈帧基址。
保存局部变量，如果寄存器不够用，编译器会把start_addr和end_addr这些变量也写进栈里。

然后执行函数的内容，初始化bss

clear_bss函数完成，函数要返回
恢复现场：把回rust_main的地址读回寄存器ra，也就是call clear_bss的下一条指令
回收空间：把SP指针向上收回，回到rust_main栈帧的边界
跳转回家：CPU根据ra寄存器跳回call clear_bss的下一条指令

![ch1-007](ch1-007.md)

# 基于SBI实现输出和关机

什么是SBI，为什么要用SBI实现
实现输出和关机，本质上是在调用BIOS固件提供的服务，在RISC-V架构中，这个机制叫做SBI。
内核是运行在S模式，但是直接控制硬件，比如通过给串口寄存器写数据来打印字符，或者控制电源管理芯片关机，都需要更高的权限，也就是M模式。
SBI就是为了解决这个问题诞生的中间层。
RustSBI是M模式，运行在最高权限，知道底层的硬件细节。
内核是S模式，不知道硬件细节，想要打印字符，只需通过SBI让RustSBI打印即可。

修改os/Cargo.toml，加入依赖，之后cargo会下载这个包。
```toml
[dependencies]
sbi-rt = { version = "0.0.2", features = ["legacy"] }
```
目前sbi-rt有新版本，但是教程里的体系就是0.0.2版本，因此`features=["legacy"]`是需要的，因为`console_putchar`和`shutdown`都属于legacy扩展，默认是不开启的。

在main.rs里声明sbi模块
```
mod sbi;
```
新建一个os/sbi.rs文件，用于编写sbi相关功能
```rust
pub fn console_putchar(c:usize){
    #[allow(deprecated)]
    sbi_rt::legacy::console_putchar(c);
}
```
`#[allow(deprecated)]`：因为用的是旧版本的标准，编译器看到会提示，加上属性表示坚持要用旧的接口。

然后实现关机功能
```rust
pub fn shutdown(failure: bool) -> ! {
    use sbi_rt::{NoReason, Shutdown, SystemFailure, system_reset};
    if !failure {
        system_reset(Shutdown, NoReason);
    } else {
        system_reset(Shutdown, SystemFailure);
    }
    unreachable!()
}
```
返回值类型是`!`，是Never Type，意味着这个函数绝不允许返回。
相同的还有`#[panic_handler]`标记的函数，返回值也必须是`!`
`system_reset`调用后，控制权交给RustSBI，RustSBI会关闭机器，CPU停止工作，永远执行不到`unreachable!()`。
但是Rust编译器更严谨，如果`system_reset`真的返回了，那么`unreachable!()`会触发panic，确保程序在这里结束。

修改main.rs中的main函数
```Rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_main() -> ! {
    clear_bss();
    sbi::console_putchar('O' as usize);
    sbi::console_putchar('k' as usize);
    sbi::console_putchar('\n' as usize);
    sbi::shutdown(false);
}
```
然后make run，就能在终端看到输出内容，然后qemu关机
```
Ok
```

# 格式化输出

如果仅使用console_putchar输出，太麻烦了。而且它只认识ASCII码。
要实现格式化输出。

在标准库不可用的裸机环境下，必须利用rust核心库提供的`core::fmt`模块，Rust编译器和核心库内置了强大的格式化逻辑，只要提供一个底层的字符写入接口，`core::fmt`就能自动处理格式化工作。

接下来的任务就是实现接口：`core::fmt::Write` 

新建os/src/console.rs文件
```rust
use core::fmt::{self, Write};

struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for c in s.chars() {
            crate::sbi::console_putchar(c as usize);
        }
        Ok(())
    }
}
```

`struct Stdout`是一个单元结构体，它没有字段，是空的。
在Rust里，这种结构体被称为零大小类型，在运行时，Stdout完全不占用内存空间，编译器在优化阶段，会把所有设计Stdout的操作直接优化成函数调用，不会有任何传递结构体的开销。
需要一个东西来代表控制台或者屏幕这个设备，因为不需要记录屏幕当前的状态，不需要缓冲区，不需要记录光标的位置，所以它不需要任何字段。
它的意义就是一个载体，要在它身上实现Write这个Trait。

`impl Write for Stdout`：给Stdout这个结构体实现Write这个Trait
什么是Trait，Trait是一种能力，Write代表能被写入字符的能力。
规定了任何想要打印能力的设备，必须提供一个`write_str`函数，用来接收字符串数据。并且只要实现了`write_str`，那么核心库core就会免费赠送`write_fmt`的实现，`write_fmt`就是专门负责调用`write_str`的。

下一步是封装对外接口，宏不方便直接调用结构体的方法，所以要封装一个函数作为中间层。
```Rust
pub fn print(args:fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}
```
`write_fmt`的工作是解析args里的格式占位符，把每一部分转换成字符串，然后多次调用`write_str`。
`fmt::Arguments`：是一个结构体，是`format_args!`宏生成的产物，它是为了保证在没有操作系统、没有内存分配的情况下，依然能做复杂的格式化输出。

下面是宏
```rust
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```
`#[macro_export]`：默认情况下，宏只在定义它的模块里可见，加上这个属性，表示把这个宏导出到整个crate包的最顶层。
`macro_rules! print`：定义声明宏的语法

宏的核心是模式匹配和代码替换
`($fmt:literal $(,$($arg:tt)+)?)`：这部分是正则表达式的变体
- `$fmt：literal`是第一个参数，命名为`$fmt`，`:literal`是约束条件，表示这个参数必须是一个字面量，不能是变量。
- `$(...)?`：`?`表示前面括号里的内容可以出现0次或1次。
	- 如print!("hi")，这是没有后续参数。
	- 如print!("hi",x)，有后续参数
- `,`：逗号分隔符，格式化字符串后面必须紧跟一个逗号。
- `$($arg: tt)+`：这是最内部的一层参数
	- `$arg:tt`表示一个参数名为`$arg`，`:tt`表示token tree，是宏匹配中最强大的匹配器之一，几乎任何合法的Rust代码片段，如变量、表达式、函数调用等。
	- `+`：重复，表示前面的$arg至少出现一次，且可以出现多次。
意思是在调用宏时，必须先给一个字面量`$fmt`，后面可以跟一个逗号`,`和一堆参数`$arg`，也可以不跟。
当编译器匹配成功后，会把代码替换为`=>`右边的样子
`$crate::console::print(format_args!($fmt $(, $($arg)+)?));`
- `$crate::console::print`：调用函数路径
- `format_args!`：是Rust编译器内置的宏，它把`$fmt`和`$args`组合起来，生成前文提到的`fmt::Aruguments`结构体，它不拼接字符，不分配堆内存，只是负责打包整理。
- `, $($arg)+)?`：把上面捕获到的参数，原封不动的塞进`format_args!`里，如果有参数，就放参数，没有参数就是空的。

`format_args!(concat!($fmt, "\n") $(, $($arg)+)?)`：
- `concat!($fmt, "\n")`：编译器宏，在编译时进行拼接
`println!`比`print!`的区别就是在格式化字符串`$fmt`后拼接了一个换行符。

用`print!("Hi {}","os")`举例子
编译时，编译器看到这段代码，不会把它编译成生成一个字符串的代码，而是宏展开。如下是伪代码
```
{
	let args = format_args!("hi,{}","os");
	$crate::console::print(args);
}
```
先由format_args!宏负责打包，然后调用封装的print函数。
format_args!做了什么，它在编译期间分析字符串，发现它由两部分组成
静态部分："hi, "
动态部分：“os” 填充{}的位置
编译器会生成指令，构建一个fmt::Arguments结构体
编译器会把字符串都放在只读数据段，这部分内存是程序一启动就占用的。
当CPU执行到第一行时，会在栈上创建一个fmt::Arguments结构体。这个结构体非常小，它不存放字符串的内容，存放的是引用（指针）。也就是字符串所在的地址。
执行到print函数时，栈上的结构体被传给了print函数。在print函数内部，又调用write_fmt函数，参数传递给write_fmt函数，它不会把内容拼好，而是读取Arguments里的第一部分的指针，然后就去调用write_str函数打印指针指向的内容，然后读下一部分，看到第二部分是占位符{}，对应的数据是字符串"os"，再调用write_str函数打印"os"。

# 处理致命错误

现在系统遇到不可恢复的错误的处理方式，只是通过死循环让系统卡住。
现在的目标让内核借助刚实现print!和shutdown函数，打印错误的信息并关机

在lang_items.rs文件里进行修改
```rust
use crate::{println, sbi::shutdown};
use core::panic::PanicInfo;

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    if let Some(location) = info.location() {
        println!(
            "Paniced at {}:{} {}",
            location.file(),
            location.line(),
            info.message()
        );
    } else {
        println!("{}", info.message());
    }
    shutdown(true)
}
```
