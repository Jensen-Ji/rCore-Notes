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

# 建立和初始化栈

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



什么是符号表
符号表的本质是一个数据结构，存在编译出来的目标文件中，记录了代码里所有名字和地址的对应关系。
作用：链接器要把目标文件链接在一起，遇到函数调用时，如call命令，就会去查表，然后把call指令的目标填上函数在符号表中的地址，这就是链接的过程。


