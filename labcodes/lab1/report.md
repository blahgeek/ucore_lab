% Lab1 Report
% 2011011262 赵一开
% \today

# 通过 make 生成执行文件的过程

- 编译相关的`.c`和`.S`文件至`.o`文件，相关的编译选项：（在makefile加入了注释）

```makefile
HOSTCC      := gcc
HOSTCFLAGS  := -g -Wall -O2
# -g: 包含调试信息
# -Wall: 输出所有warning
# -O2: 优化选项

CC      := $(GCCPREFIX)gcc
CFLAGS  := -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
# -fno-builtin: 不接受不是__builtin_开头的内建函数
# -Wall: 输出所有warning
# -ggdb: 生成gdb使用的调试信息
# -m32: 生成32位代码
# -gstabs: 调试相关选项
# -nostdinc: 不使用系统文件夹中的头文件，只用-I定义的文件夹
CFLAGS  += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
# -fno-stack-protector: 有些版本的编译器默认开启栈保护选项，将其关闭
```

- 链接生成kernel，链接时使用`kernel.ld`脚本来确定各个section的位置，生成kernel的符号表和汇编代码
- 链接生成bootblock，并使用`sign`工具将生成的二进制文件扩展为512字节，最后两个字节填上特殊字符
- 使用dd工具将bootblock和kernel合并生成`ucore.img`，其中bootblock占第一个block（512字节）

一个被系统认为是符合规范的硬盘主引导扇区前512字节为引导代码，其中第511、512两个字节内容固定为`0x55AA`，表示这是一个合法的引导扇区。

# 使用gdb调试启动过程

系统复位后，通过gdb查看得cs寄存器的值为`0xf000`，eip寄存器的值为`0xfff0`，则第一条指令的位置是`0xffff0`，通过`x/i 0xffff0`查看到该条指令为`ljmp $0xf000,$0xe05b`，即跳转到`0xfe05b`地址。

> 可能是gdb不支持CPU的实模式，显示的PC值即为eip寄存器的值，是不正确的。

这部分的代码都属于BIOS，最后BIOS会将操作系统的bootloader加载至`0x7c00`位置内存，并跳转。使用`b * 0x7c00`设置断点后，可以看到该处的反汇编代码与`bootasm.S`的代码相同，如下：

```asm
(gdb) x/10i 0x7c00
=> 0x7c00:  cli
   0x7c01:  cld
   0x7c02:  xor    %ax,%ax
   0x7c04:  mov    %ax,%ds
   0x7c06:  mov    %ax,%es
   0x7c08:  mov    %ax,%ss
   0x7c0a:  in     $0x64,%al
   0x7c0c:  test   $0x2,%al
   0x7c0e:  jne    0x7c0a
   0x7c10:  mov    $0xd1,%al
```

# 分析进入保护模式的过程

- `lgdt gdtdesc`：Load Global Descriptor Table，载入全局描述符表。`gdtdesc`包含了全局描述符表，第一个word为`0x17`表示共有3个（`(0x17+1)/8`）描述符表，之后跟随着gdt的地址。
- 接下来三条指令将寄存器CR0中的PE位置1。
- 跳到保护模式下的下一条指令的地址，其中段号为装入全局描述符表的kernel代码所在的段号。

# 加载 ELF 格式的 OS 的过程

## 读取磁盘的过程

- 从`0x1F2`IO口输出1，表示读取一个扇区
- 从`0x1F3`至`0x1F6`输出LBA
- 从`0x1F7`发送读取命令
- 等待磁盘返回，根据`0x1F7`的值判断磁盘是否忙

## 加载ELF的过程

从磁盘读取出第一个扇区的数据后，通过魔数判断其是否为ELF头，如果不是的话转向错误处理。然后根据ELF头部记录的代码段的偏移量，载入代码段，然后跳转到代码段起始地址。

# 实现函数调用堆栈跟踪函数

见`kern/debug/kdebug.c`源代码。输出结果：

```
ebp: 7b08, eip: 100a62, args: 10094 0 7b38 100092
    kern/debug/kdebug.c:306: print_stackframe+21
ebp: 7b18, eip: 100d50, args: 0 0 0 7b88
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp: 7b38, eip: 100092, args: 0 7b60 ffff0000 7b64
    kern/init/init.c:48: grade_backtrace2+33
ebp: 7b58, eip: 1000bb, args: 0 ffff0000 7b84 29
    kern/init/init.c:53: grade_backtrace1+38
ebp: 7b78, eip: 1000d9, args: 0 100000 ffff0000 1d
    kern/init/init.c:58: grade_backtrace0+23
ebp: 7b98, eip: 1000fe, args: 1032fc 1032e0 130a 0
    kern/init/init.c:63: grade_backtrace+34
ebp: 7bc8, eip: 100055, args: 0 0 0 10094
    kern/init/init.c:28: kern_init+84
ebp: 7bf8, eip: 7d68, args: c031fcfa c08ed88e 64e4d08e fa7502a8
    <unknow>: -- 0x00007d67 --
```

其中最后一个`<unknow>`为调用`kern_init`的函数栈，为`bootmain.c`中的`bootmain`函数。

# 中断初始化和处理

中断门描述符大小为8个字节，其中高16位和最低16位组成入口地址的offset，第16至31位为入口地址的段号，这些位代表了中断处理的入口地址。

初始化时，向每个中断描述符表中添加相应的中断处理程序的入口地址以及一些其他信息，最后使用`lidt`指令将中断描述符起始地址传给CPU。

```
for( ; i < 256 ; i += 1){
    SETGATE(idt[i],
            i == T_SYSCALL, //is trap ?
            KERNEL_CS, // kernel code segment
            __vectors[i],
            (i == T_SYSCALL ? DPL_USER : DPL_KERNEL));
}
lidt(&idt_pd);
```

在`trap_dispatch`函数中添加对时钟中断的处理代码，编译后运行的效果即为每约1秒钟输出一个`100 ticks`。
