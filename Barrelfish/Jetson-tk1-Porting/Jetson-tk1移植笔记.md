# Barrelfish在Jetson-tk1上的移植笔记

## Day1

### armv7/offest.h注解

文件位于kernel/arch/armv7/offest.h，该文件主要定义了与内核启动相关的地址（偏移）。

关键注解如下：

```c
/*
 * 一些一般的命名规则:
 *    *_SIZE ：以字节为单位给出
 *    *_SPACE_* ：物理或者虚拟的地址空间
 *    *_PHYS ：物理地址相关
 *    *_LIMIT ：地址限制，物理或者虚拟
 *    *_OFFSET ：偏移
 */
```

### Barrelfish构建系统

目前Barrelfish操作系统支持X86，X86_64，ARMv7，ARMv8等处理器架构，在每个处理器架构上对不同的硬件平台进行了适配。就ARMv7而言，根据硬件平台的不同，Barrelfish实现了PandaboardES，VExpressEMM-A15，VExpressEMM-A15-A7，VExpressEMM-A9，Zynq7000等版本。

与通常的大型软件一样，Barrelfish采用Makefile进行编译控制，与其他软件不同的是，Barrelfish采用单一并且结构简单的Makefile。为了实现Makefile的自动生成，Barrelfish用Haskell语言实现了Hake构建工具。Hake根据代码树中的所有 Hakefile 生成一个单一的Makefile，通常一个Hakefile对应一组Make规则。

我们在这里主要关心的是如何为我们的Jetson-tk1开发板新建一个分支。在代码树中我们可以看到platforms文件夹下有个Hakefile文件，这个文件具体定义了一个平台（platform）的构建规则。通常一个具体平台需要定义如下的内容：

1. Modules：该平台需要加载的模块，包括CPU_driver等。
2. platform Rules：平台构建规则。
3. BootImage Rules：启动镜像生成规则。
4. menu.list Rule：Menu List生成规则。
5. Boot Rules：平台启动规则（模拟器运行或生成可启动介质）。

仿照PandaboardES的各项定义，可以添加Jetson-tk1的分支，然后修改代码树中相关的Hakefile，完成Jetson-tk1的各个Module的编译规则定义。例如 kernel 文件夹中Hakefile定义个各个Cpu_driver的编译规则。

### ARM-A15 Boot过程

Barrelfish在ARM-A15架构上的Bootstrap过程。

Bootstrap有别与Bootloader，但又有相似之处，与Bootloader作用相似主要用于引导进入内核代码，作为Bootloader与内核之间的纽带，但Bootstrap代码一般与内核相关性强，属于内核代码的一部分。

例如，Linux的内核启动流程（来源于网络资料）：

1. 加电
2. Bootloader引导加载内核镜像
3. Bootstrap代码（head.o）
4. 内核vmlinux（head.o）
5. 内核start_kernel

关键的数据结构：

PIC_REGITER：在Makefile中，GCC编译选项中设定为R9。

got_base：ELF文件中.got段的基址，got全称为gobal offest table。

boot_records：该数据结构定义在armv7/boot_driver.c文件中，其类型为armv7_boot_record 结构体（kernel/include/arch/armv7/boot_protocol.h），用于启动APP核。

Bootstrap的主要逻辑如下：

## Day2

### Boot Driver关键数据结构

数据结构1：arm_core_data，一个新kernel启动是需要的数据。

| 属性                | 类型                       | 注解                            |
| :---------------- | :----------------------- | :---------------------------- |
| multiboot_header  | lvaddr_t                 | multiboot header的物理（虚拟）地址     |
| kernel_l1_low     | lpaddr_t                 | 低地址空间页表基址                     |
| kernel_l2_high    | lpaddr_t                 | 高地址空间页表基址                     |
| kernel_l2_vec     | lpaddr_t                 | 向量表映射空间基址（异常等）                |
| entry_point       | lvaddr_t                 | CPU driver入口地址                |
| stack_base        | lpaddr_t                 | CPU driver堆栈基址                |
| stack_top         | lpaddr_t                 | CPU driver堆栈栈顶                |
| kernel_module     | struct multiboot_modinfo | 内核模块信息                        |
| kernel_elf        | struct multiboot_elf     | ELF头部信息                       |
| kcb               | lvaddr_t                 | 预分配的内核控制块                     |
| cmdline_buff      | char *                   | 内核命令行参数，可能与BSP核有区别            |
| cmdline           | lvaddr_t                 | 指向前一项或者multiboot镜像            |
| urpc_frame_base   | uint32_t                 | 预分配monitor信道frame基址           |
| urpc_frame_size   | uint32_t                 | 预分配monitor信道frame大小           |
| chan_id           | uint32_t                 | 预分配monitor信道ID                |
| monitor_module    | struct multiboot_modinfo | monitor模块信息                   |
| memory_base_start | uint32_t                 | 该内核会被分配的地址空间起始                |
| memory_bytes      | uint32_t                 | 内核别分配空间字节数                    |
| src_core_id       | uint32_t                 | 启动本内核的内核ID                    |
| dst_core_id       | uint32_t                 | 本内核ID，由启动内核设置                 |
| src_arch_id       | uint8_t                  | 启动本内核的内核架构                    |
| global            | lvaddr_t                 | global锁地址？                    |
| target_bootrecs   | lvaddr_t                 | boot driver邮箱地址（boot records） |
| build_id          | struct gnu_build_id      | GNU build ID                  |
| kernel_load_base  | lvaddr_t                 | Image不可重定向部分的加载地址             |
| got_base          | lvaddr_t                 | global offest table的虚拟地址      |

数据结构2：multiboot_info，bootlaoder传送个OS的Multiboot相关信息。关键属性如下：

1. `flags`：表明multiboot_info的属性，`uint32_t`。
2. `mem_lower`：内存空间下界。

数据结构3：

### Boot Drvier修改

添加文件：

1. devices/tegrak1/jetson_semaphore.dev
2. devices/tegrak1/jetson_uart.dev
3. devices/tegrak1/jetson_flow_controller.dev
4. devices/tegrak1/jetson_exception_vetors.dev
5. kernel/arch/armv7/plat_tegrak1_boot.c
6. include/jetsontk1_map.h

修改文件：

1. devices/Hakefile


## Day3

### 串口驱动



### 文件修改

## Day4

### 调试小技巧

串口未成功启动之前，可以直接向UART的TBR寄存器中写字符，可用于测试制定位置代码是否运行。样式代码如下:

```c
//汇编文件中
ldr r5, =0x70006300 //UARTD的TBR寄存器地址，UARTD由Uboot初始化
mov r6, #65 //字符‘A’
str r6, [r5] //输出到控制台
b	. //死循环，halt
//C文件中
__asm volatile(
	"ldr	r5,	=0x70006300\n\t"
	"mov	r6, #65\n\t"
	"str	r6, [r5]\n\t"
	"b	.");
//或者
char *temp = (char *) 0x70006300
*temp = 'C';
```

### NS16550串口调试

U-boot初始化串口过程如下：

1. 检查LSR的tsr_e为1，表示TX，RX缓冲为空
2. 置LCR的dlab位为1，清除波特率除数高位和低位，而后恢复dlab为0
3. MCR置dtr，rts位为1
4. ​



### Jetson-tk1 波特率除数计算

计算公式：UART时钟频率 / 波特率除数 = 16 × 波特率

其中UART时钟频率为408000000 Hz，波特率为115200

### 待解决问题

1. 波特率（少写一个零）
2. spinlock
3. CPU中设备映射


### Day5

### CPU中设备映射问题的解决

最后确定问题源于Cache刷新存在bug，目前暂时改用老版的Cache刷新代码。

### 波特率

408Mhz，少写一个零。

### spinlock

???

### Day6

