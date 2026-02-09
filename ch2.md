在ch1，我还是上帝，代码是我写的，机器是我控制的，想怎么跑就怎么跑。但是在ch2，我的身份发生了转变，我不再是写程序的人，而是管理程序的人，我要从写代码的程序员，变成制订规则的系统管理员。

为什么要引入特权级和应用程序？
ch1的libos中，内核和应用是一体的（os.bin），这种架构有缺陷
1. 安全性缺失
	libos中rust_main直接运行在S模式，如果业务逻辑写错了，如访问了非法内存，CPU就会抛出异常，整个系统直接崩溃。
	需要把业务逻辑关进笼子里，如果它崩溃了，操作系统应该捕捉这个错误，然后杀掉出错的业务，但是操作系统本身必须活着。
	解决办法：特权级机制，让OS跑在S模式，让APP跑在U模式。
2. 通用性与动态性
	libos是静态链接的，如果想运行程序A和程序B，比如把它们的代码分别拷贝到main.rs里，然后每次都要重新编译整个内核。
	操作系统应该是一个平台，可以加载并运行内核二进制可执行文件，而不是需要重新编译内核。
	解决办法：批处理系统，OS负责把不同的APP的二进制文件加载到内存中并运行。
3. 硬件抽象的隔离
	libos中，rust_main直接调用sbi::console_putchar
	APP不应该知道SBI的存在，也不知道硬件的细节，只会知道操作系统提供了一个打印接口，然后调用这个打印接口。
	解决办法：系统调用，APP通过特殊的指令向OS发送请求，OS检查合法性后，再由OS去操作硬件。 

为了实现上述问题，ch2要实现
1. 把APP剥离出去，新建一个user/目录，这里的代码是单独编译的，这是一个超级迷你的自己实现的标准库user_lib，让它也能用println!，但是底层的实现变成了按键呼叫，而不是直接调用SBI。
2. 构建加载器，OS需要从硬盘或内存中的某个地方，找到编译好的APP二进制文件，把它们搬运到内存的指定位置。
3. 实现特权级切换，APP在运行的时候，想打印（ecall）或者出错了，CPU会强制跳转到内核的特定位置，需要用汇编代码__restore和__alltraps来保存上下文。
4. 管理执行流，APP1运行完，OS会把它清理掉，加载APP2。













完成了Ch1的裸机"Hello world"，这是一个独裁者，整个计算机只有这一段代码在跑。
Ch2的目标是引入应用程序APP。

现在计算机里有两个角色：
操作系统 和 应用程序

既然有了两个角色，就不能像ch1那样混用同一块内存，如果APP把OS的栈写坏了，OS就崩溃了。
所以CH2的第一步，就是实现应用程序和操作系统的物理隔离。需要在代码中明确划分出两个区域，一块给APP用，一块给OS用。

# 定义双栈

在os/src/目录下新建batch.rs文件，在这里定义用户栈和内核栈
在RISC-V特权级架构中，特权级切换必然伴随着栈切换。
CPU处于Smode时，使用内核栈来保存函数调用上下文和局部变量。
CPU处于Umode时，使用用户栈。
为什么这么做？
安全性考虑，如果使用同一个栈，APP可以通过修改栈上的返回地址来劫持控制流，或者通过栈溢出覆盖内核的关键数据。因此，必须在内存布局.bss段中静态分配两个独立的数组作为栈空间。
```Rust
const USER_STACK_SIZE: usize = 4096 * 2;
const KERNEL_STACK_SIZE: usize = 4096 * 2;

#[repr(align(4096))]
struct KernelStack {
    data: [u8; KERNEL_STACK_SIZE],
}

#[repr(align(4096))]
struct UserStack {
    data: [u8; USER_STACK_SIZE],
}

static KERNEL_STACK: KernelStack = KernelStack {
    data: [0; KERNEL_STACK_SIZE],
};

static USER_STACK: UserStack = UserStack {
    data: [0; USER_STACK_SIZE],
};

impl KernelStack{
    fn get_sp(&self) -> usize{
        self.data.as_ptr() as usize + KERNEL_STACK_SIZE
    }
}

impl UserStack{
    fn get_sp(&self) -> usize{
        self.data.as_ptr() as usize + USER_STACK_SIZE
    }
}
```
定义栈大小为8kb
确保栈的内存地址是4KB对齐的，这是为后续的分页机制做准备
实例化两个全局静态变量，会被放在.bss段（未初始化数据段），在OS启动时由clear_bss清零。
	Rust中，未初始化或初始化为0的全局变量，默认都会被放在.bss段，.bss段在编译出来的二进制文件里不占空间，只会让预留一块空地。如果放在.data段（初始化数据段），那么这16kb全零就会被写入二进制文件里，导致内核的体积臃肿。再者，栈本来就是空的，就是等待被使用的，也可以理解为等待被填充（被初始化）。
实现获取栈顶的方法，栈顶指针SP=初始地址+大小。

ch1里的.bss.stack又是什么栈？
这是启动栈，它属于裸机程序，在CPU加电启动时，SP寄存器是乱的，或者指向固件的地址，为了能从汇编_start跳转到代码rust_main，需要一个栈来保存函数调用的返回地址和局部变量。如果没有这个启动栈，Rust的第一个函数都调用不了。（ch1的[函数调用](ch1#函数调用)）

kernelstack的作用：
- 保存上下文，当APP从Umode跳到Smode的瞬间，如ecall，会把APP的所有通用寄存器都push到kernelstack的栈顶。
- 运行OS的代码，保存上下文之后，OS也要执行函数，可能也要调用其他函数，也需要栈，这些就是用kernelstack的空间。
userstack的作用：是APP使用的栈，函数调用、局部变量都在这。


