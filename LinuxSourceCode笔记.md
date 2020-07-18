[TOC]

*head.s最后，通过ret跳转到main函数（特殊用法，其实有必要么，用call是一样吧，关键是main自己的写法）。*

# main函数1:初始化内核数据结构
初始化内核数据结构，构造进程0，并跳转到用户态


## 内核数据区初始化

### 在1M后的空间分配3M的buffer区
### 在4M后的空间分配2M的ramdisk区(rd_init)
（6M已经分出去了）；
```
# 这里do_rd_request其实用到了request结构，但其此时还没有初始化，所以此调用还不能正常运行
 blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;  // do_rd_request()。
 rd_start = (char *) mem_start;                  // 对于16MB系统该值为4MB。
 rd_length = length;                             // 虚拟盘区域长度值。
```
#### 虚拟内存设备的请求项处理函数
该函数的程序结构与硬盘的do_hd_request()函数类似，参见hd.c，294行。
- 在低级块设备接口函数ll_rw_block()建立起虚拟盘（rd）的请求项并添加到 rd的链表中之后，就会调用该函数对 rd当前请求项进行处理。
- 该函数首先计算当前请求项中指定的起始扇区对应虚拟盘所处内存的起始位置addr 和要求的扇区数对应的字节长度值 len，然后根据请求项中的命令进行操作。
- 若是写命令WRITE，就把请求项所指缓冲区中的数据直接复制到内存位置addr处。
- 若是读操作则反之。
- 数据复制完成后即可直接调用 end_request() 对本次请求项作结束处理。然后跳转到函数开始处再去处理下一个请求项。若已没有请求项则退出。

虚拟内存设备请求属于块设备请求，详细过程见[块设备访问](#块设备访问)

### 初始化内存管理结构mem_map(mem_init)
物理内存管理初始化。

system数据区的3.75K数组，每个字节标记1个分区，总计15M，其中前5M分区设置为USED，后10M设置为0;


该函数对1MB以上内存区域以页面为单位进行管理前的初始化设置工作。一个页面长度为4KB字节。

该函数把1MB以上所有物理内存划分成一个个页面，并使用一个页面映射字节数组mem_map[] 来管理所有这些页面。对于具有16MB内存容量的机器，该数组共有3840项 ((16MB - 1MB)/4KB)，即可管理3840个物理页面。

每当一个物理内存页面被占用时就把mem_map[]中对应的的字节值增1；若释放一个物理页面，就把对应字节值减1。 若字节值为 0，则表示对应页面空闲；若字节值大于或等于1，则表示对应页面被占用或被不同程序共享占用。

在该版本的Linux内核中，最多能管理16MB的物理内存，大于16MB的内存将弃置不用。

对于具有16MB内存的PC机系统，在没有设置虚拟盘RAMDISK的情况下start_mem通常是4MB，end_mem是16MB。因此此时主内存区范围是4MB—16MB，共有3072个物理页面可供分配。

而范围0-1MB内存空间用于内核系统（其实内核只使用0—640Kb，剩下的部分被部分高速缓冲和设备内存占用）。

参数start_mem是可用作页面分配的主内存区起始地址（已去除RAMDISK所占内存空间）。end_mem是实际物理内存最大地址。而地址范围start_mem到end_mem是主内存区。

```
//linux/include/linux/mm.h

/* 下面定义若需要改动，则需要与head.s等文件中的相关信息一起改变 */
// Linux 0.12内核默认支持的最大内存容量是16MB，可以修改这些定义以适合更多的内存。
#define LOW_MEM 0x100000                     // 机器物理内存低端（1MB）。
extern unsigned long HIGH_MEMORY;            // 存放实际物理内存最高端地址。
#define PAGING_MEMORY (15*1024*1024)
#define PAGING_PAGES (PAGING_MEMORY>>12)
#define MAP_NR(addr) (((addr)-LOW_MEM)>>12)
#define USED 100

//linux/mm/memory.c

// 物理内存映射字节图（1字节代表1页内存）。每个页面对应的字节用于标志页面当前被引用（占用）次数。它最大可以映射15Mb的内存空间。在初始化函数mem_init()中，对于不能用作主内存区页面的位置均都预先被设置成USED（100）。
unsigned char mem_map [ PAGING_PAGES ] = {0,};

void mem_init(long start_mem, long end_mem)
{
        int i;
// 首先将1MB到16MB范围内所有内存页面对应的内存映射字节数组项置为已占用状态，即各项字节值全部设置成USED（100）。PAGING_PAGES被定义为(PAGING_MEMORY>>12)，即1MB以上所有物理内存分页后的内存页面数(15MB/4KB = 3840)。
        HIGH_MEMORY = end_mem;                     // 设置内存最高端（16MB）。
        for (i=0 ; i<PAGING_PAGES ; i++)
                mem_map[i] = USED;
// 然后计算主内存区起始内存 start_mem处页面对应内存映射字节数组中项号i和主内存区页面数。此时 mem_map[] 数组的第i项正对应主内存区中第1个页面。最后将主内存区中页面对应的数组项清零（表示空闲）。对于具有16MB物理内存的系统，mem_map[] 中对应4Mb--16Mb主内存区的项被清零。
        i = MAP_NR(start_mem);                     // 主内存区起始位置处页面号。
        end_mem -= start_mem;
        end_mem >>= 12;                            // 主内存区中的总页面数。
        while (end_mem-->0)
                mem_map[i++]=0;                    // 主内存区页面对应字节值清零。
}

```
###  设置中断调用门（中断向量）(trap_init)

异常（陷阱）中断程序初始化子程序。设置它们的中断调用门（中断向量）。

set_trap_gate()与set_system_gate()都使用了中断描述符表IDT中的陷阱门（Trap Gate），它们之间的主要区别在于前者设置的特权级为0，后者是3。因此断点陷阱中断int3、溢出中断overflow 和边界出错中断 bounds 可以由任何程序调用。 这两个函数均是嵌入式汇编宏程序，参见include/asm/system.h
  
```
void trap_init(void)
{
    int i;

    set_trap_gate(0,&divide_error);     // 设置除操作出错的中断向量值。以下雷同。
    set_trap_gate(1,&debug);
    set_trap_gate(2,&nmi);
    set_system_gate(3,&int3);           /* int3-5 can be called from all */
    set_system_gate(4,&overflow);       /* int3-5 可以被所有程序执行 */
    set_system_gate(5,&bounds);
    set_trap_gate(6,&invalid_op);
    set_trap_gate(7,&device_not_available);
    set_trap_gate(8,&double_fault);
    set_trap_gate(9,&coprocessor_segment_overrun);
    set_trap_gate(10,&invalid_TSS);
    set_trap_gate(11,&segment_not_present);
    set_trap_gate(12,&stack_segment);
    set_trap_gate(13,&general_protection);
    set_trap_gate(14,&page_fault);
    set_trap_gate(15,&reserved);
    set_trap_gate(16,&coprocessor_error);
    set_trap_gate(17,&alignment_check);

// 下面把int17-47的陷阱门先均设置为reserved，以后各硬件初始化时会重新设置自己的陷阱门。
    for (i=18;i<48;i++)
        set_trap_gate(i,&reserved);

// 设置协处理器中断0x2d（45）陷阱门描述符，并允许其产生中断请求。设置并行口中断描述符。
    set_trap_gate(45,&irq13);
    outb_p(inb_p(0x21)&0xfb,0x21);          // 允许8259A主芯片的IRQ2中断请求。
    outb(inb_p(0xA1)&0xdf,0xA1);            // 允许8259A从芯片的IRQ13中断请求。
    set_trap_gate(39,&parallel_interrupt);  // 设置并行口1的中断0x27陷阱门描述符。
}
```
### 初始化块设备(blk_dev_init)
初始化请求数组，将所有请求项置为空闲项(dev = -1)。有32项(NR_REQUEST = 32)。
（request为36 * 32 = 1K128B）

这里初始化的request项，在后续各块设备发起读块请求时被用到。（细节描述后补[块设备请求](#request)，今天我只关心tty，快到了）
```
//linux/kernel/blk_drv/blk.h
#define NR_REQUEST      32

//linux/kernel/blk_drv/ll_rw_blk.c
//请求结构中含有加载nr个扇区数据到内存中去的所有必须的信息。请求项数组队列。共有NR_REQUEST = 32个请求项。
struct request request[NR_REQUEST];

//linux/kernel/blk_drv/ll_rw_blk.c
void blk_dev_init(void)
{
    int i;
    for (i=0 ; i<NR_REQUEST ; i++) {
        request[i].dev = -1;
        request[i].next = NULL;
    }
}

```
### 初始化字符设备(chr_dev_init)
这个有点意外啊
```
//// 字符设备初始化函数。空，为以后扩展做准备。
void chr_dev_init(void)
{
} 
```
### tty终端初始化(tty_init)
tty终端初始化函数。初始化所有终端缓冲队列，初始化串口终端和控制台终端。 
tty细节描述后补[字符设备访问](#字符设备访问)
三种终端类型，其中伪终端又分主从两种（master和slave），总计4种18个终端。
```
//linux/include/linux/tty.h

//伪终端分主从两种（master和slave），总计18个终端。
#define MAX_CONSOLES    8         // 最大虚拟控制台数量。
#define NR_SERIALS      2         // 串行终端数量。
#define NR_PTYS         4         // 伪终端数量。
```

每个tty终端使用3个tty 缓冲队列，它们分别是：
1. 用于缓冲键盘或串行输入的读队列 read_queue
2. 用于缓冲屏幕或串行输出的写队列 write_queue
3. 以及用于保存规范模式字符的辅助缓冲队列secondary。

每个缓冲队列结构可以保存1024个字节

问题：设置显示控制 选择寄存器端口/数据寄存器端口 分别为0x3b4，0x3b5。
    - 初始化控制台tty结构，关联终端缓冲队列con_queues；
    - 初始化串行终端tty结构，关联终端缓冲队列rs_queues；
    - 初始化伪终端使用的tty结构，关联终端缓冲队列mpty_queues和spty_queues；
    - 最后初始化串行中断处理程序和串行接口1和2。

####  初始化所有终端缓冲队列
- tty_queues占 54 *（1024 + 16）

按照终端类型分类，将缓冲队列结构数组tty_queues划分4块
1. 8个虚拟控制台终端占用tty_queues[]数组开头24项（3 X MAX_CONSOLES）（0 -- 23）；
2. 2个串行终端占用随后的6项（3 X NR_SERIALS）（24 -- 29）；
3. 4个主伪终端占用随后的12项（3 X NR_PTYS）（30 -- 41）；
4. 4个从伪终端占用随后的12项（3 X NR_PTYS）（42 -- 53）。

```
//linux/kernel/chr_drv/tty_io.c

// QUEUES是tty终端使用的缓冲队列最大数量。
#define QUEUES  (3*(MAX_CONSOLES+NR_SERIALS+2*NR_PTYS))  // 共54项。

// 下面定义tty终端使用的缓冲队列结构数组tty_queues
static struct tty_queue tty_queues[QUEUES];              // tty缓冲队列数组。

// 下面设定各种类型的tty终端所使用缓冲队列结构在tty_queues[]数组中的起始项位置。
#define con_queues tty_queues  
#define rs_queues ((3*MAX_CONSOLES) + tty_queues) 
#define mpty_queues ((3*(MAX_CONSOLES+NR_SERIALS)) + tty_queues) 
#define spty_queues ((3*(MAX_CONSOLES+NR_SERIALS+NR_PTYS)) + tty_queues) 
```

初始化所有终端的缓冲队列结构，设置初值。对于串行终端的读/写缓冲队列，将它们的data字段设置为串行端口基地址值。串口1是0x3f8，串口2是0x2f8。

问题：串口设备0，1，3，4点data字段设置为0x3f8，0x2f8的串行端口基地址。这个基地址啥意思。
==phoenix:2020.2.1凌晨：所有与串口的交互，是通过几个端口完成的，这个基地址，起始是端口号开始的端口。对端口操作就是对它，以及以它为偏移的几个端口操作的==
```
for (i=0 ; i < QUEUES ; i++)
        tty_queues[i] = (struct tty_queue) {0,0,0,0,""};
rs_queues[0] = (struct tty_queue) {0x3f8,0,0,0,""};
rs_queues[1] = (struct tty_queue) {0x3f8,0,0,0,""};
rs_queues[3] = (struct tty_queue) {0x2f8,0,0,0,""};
rs_queues[4] = (struct tty_queue) {0x2f8,0,0,0,""};
```
####  初始化所有终端tty结构
- tty_table 256 * 64。

按照终端类型分类，将tty终端表结构数组tty_table划分4块，
1. 8个虚拟控制台终端可用tty_table[]数组开头64项（0 -- 63），其实只用了8项；
2. 2个串行终端使用随后的64项（64 -- 65），其实只用了2项;
3. 4个主伪终端使用从128开始的项，最多64项（128 -- 191），其实只用了4项;
4. 4个从伪终端使用从192开始的项，最多64项（192 -- 255），其实只用了4项。

==串行终端段这里看起来空闲了62项，其实感觉其才是正常的，另外3类终端，应该是各自占有8/4/4个table项才是合理的吧==

```
//linux/kernel/chr_drv/tty_io.c

下面定义tty终端使用的tty终端表结构数组tty_table。
struct tty_struct tty_table[256];                        // tty表结构数组。

//下面设定各种类型tty终端所使用的tty结构在tty_table[]数组中的起始项位置。
#define con_table tty_table                // 定义控制台终端tty表符号常数。
#define rs_table (64+tty_table)            // 串行终端tty表。
#define mpty_table (128+tty_table)         // 主伪终端tty表。
#define spty_table (192+tty_table)         // 从伪终端tty表。
int fg_console = 0;              // 当前前台控制台号（范围0--7）。

```
tty_init中初始化所有256项tty结构
```
// 然后先初步设置所有终端的tty结构。其中特殊字符数组c_cc[]设置的初值定义在include/linux/tty.h文件中。
        for (i=0 ; i<256 ; i++) {
                tty_table[i] =  (struct tty_struct) {
                        {0, 0, 0, 0, 0, INIT_C_CC},
                        0, 0, 0, NULL, NULL, NULL, NULL
                };
        }
```
####  初始化控制台终端
接着初始化控制台终端（console.c，834行）。

初始化控制台终端tty结构(con_table，其实是tty_table的前64项)，其中读缓冲、写缓冲和辅助缓冲队列结构，分别指向tty缓冲队列结构数组(con_queues,其实是tty_queues的前24项) 中的相应结构项。

这里con_table和vc_cons有对应关系，个数就是NR_CONSOLES。
==这里看到con_table至少有56项浪费了。==（2020.01.31：是的，丫的tty_table不仅仅到con_table浪费了56项，rs_talbe也浪费了62项，后面两组也浪费了62项）
```
for (i = 0 ; i<NR_CONSOLES ; i++) {
    con_table[i] = (struct tty_struct) {
        {ICRNL,       /* change incoming CR to NL */   /* CR转NL */
        OPOST|ONLCR,  /* change outgoing NL to CRNL */ /*NL转CRNL*/
        0,                                    // 控制模式标志集。
        IXON | ISIG | ICANON | ECHO | ECHOCTL | ECHOKE, // 本地标志集。
        0,            /* console termio */    // 线路规程，0 -- TTY。
        INIT_C_CC},                           // 控制字符数组c_cc[]。
        0,            /* initial pgrp */      // 所属初始进程组pgrp。
        0,            /* initial session */   // 初始会话组session。
        0,            /* initial stopped */   // 初始停止标志stopped。
        con_write,                            // 控制台写函数。
        con_queues+0+i*3,con_queues+1+i*3,con_queues+2+i*3
    };
}
```

注意：此初始化过程应该在[控制台初始化(con_init)](#con_init)之后.
 <span id="con_init"></span>
#####  控制台初始化(con_init)
```
    con_init();
```
重要：显式的处理

根据显示器参数，设置显存起始地址和结束地址以及一屏的行数和列数；并计算出显存总共支持的屏数(虚拟控制台数NR_CONSOLES)。

初始化虚拟控制台信息vc_cons[MAX_CONSOLES]，主要设置显存地址，显式格式等信息。
注意：此处初始化的vc_cons个数，逻辑上讲是NR_CONSOLES个，而不是MAX_CONSOLES个，不过他们刚好相等。
```
static struct {
	unsigned short  vc_video_erase_char;    // 擦除字符属性及字符（0x0720）
	unsigned char   vc_attr;                // 字符属性。
	unsigned char   vc_def_attr;            // 默认字符属性。
	int             vc_bold_attr;           // 粗体字符属性。
	unsigned long   vc_ques;                // 问号字符。
	unsigned long   vc_state;               // 处理转义或控制序列的当前状态。
	unsigned long   vc_restate;             // 处理转义或控制序列的下一状态。
	unsigned long   vc_checkin;
	unsigned long   vc_origin;              /* Used for EGA/VGA fast scroll */
	unsigned long   vc_scr_end;             /* Used for EGA/VGA fast scroll */
	unsigned long   vc_pos;                 // 当前光标对应的显示内存位置。
	unsigned long   vc_x,vc_y;              // 当前光标列、行值。
	unsigned long   vc_top,vc_bottom;       // 滚动时顶行行号；底行行号。
	unsigned long   vc_npar,vc_par[NPAR];   // 转义序列参数个数和参数数组。
	unsigned long   vc_video_mem_start;     /* Start of video RAM           */
	unsigned long   vc_video_mem_end;       /* End of video RAM (sort of)   */
	unsigned int    vc_saved_x;             // 保存的光标列号。
	unsigned int    vc_saved_y;             // 保存的光标行号。
	unsigned int    vc_iscolor;             // 彩色显示标志。
	char *          vc_translate;           // 使用的字符集。
} vc_cons [MAX_CONSOLES];
```
con_init这个子程序初始化控制台中断，其他什么都不做。如果你想让屏幕干净的话，就使用 适当的转义字符序列调用tty_write()函数。

把con_init()放在这里，是因为我们需要根据显示卡类型和显示内存容量来确定系统中虚拟控制台的数量NR_CONSOLES。该值被用于随后的控制台tty 结构初始化循环中。

<span id="键盘中断"></span>
#####  键盘中断
设置键盘中断陷阱门描述符，打开键盘中断屏蔽位，重置键盘中断。
这里键盘中断尤其重要：

- 当键盘控制器接收到用户的一个按键操作时，就会向中断控制器发出一个键盘中断请求信号IRQ1。当CPU响应该请求时就会执行键盘中断处理程序。该中断处理程序会从键盘控制器相应端口（0x60）读入按键扫描码，并调用对应的扫描码子程序进行处理。
- 首先从端口0x60读取当前按键的扫描码。然后判断该扫描码是否是0xe0 或 0xe1，
    - 如果是的话就立刻对键盘控制器作出应答，并向中断控制器发送中断结束（EOI）信号，以允许键盘控制器能继续产生中断信号，从而让我们来接收后续的字符。
    - 如果接收到的扫描码不是这两个特殊扫描码，我们就根据扫描码值调用按键跳转表 key_table 中相应按键处理子程序，把扫描码对应的字符放入读字符缓冲队列read_q中。然后，在对键盘控制器作出应答并发送EOI信号之后，调用函数do_tty_interrupt()（实际上是调用 copy_to_cooked()）把read_q中的字符经过处理后放到secondary辅助队列中。

注意：
- 键盘中断程序使用了table_list,这里默认用的0号控制台。
```
/*
 * these are the tables used by the machine code handlers.
 * you can implement virtual consoles.
 */
/*
 * 下面是汇编程序中使用的缓冲队列结构地址表。通过修改这个表，
 * 你可以实现虚拟控制台。
 */
// tty读写缓冲队列结构地址表。供rs_io.s程序使用，用于取得读写缓冲队列结构的地址。
struct tty_queue * table_list[]={
	con_queues + 0, con_queues + 1,       // 前台控制台读、写队列结构地址。
	rs_queues + 0, rs_queues + 1,         // 串行终端1读、写队列结构地址。
	rs_queues + 3, rs_queues + 4          // 串行终端2读、写队列结构地址。
        };
```
- 键盘中断程序使用了tty_table，此时，它还没有完全初始化
- 此时(键盘中断打开时)con_queues还没有初始化，它在后面初始化的，因为con_init执行完才知道显存够启动多少个控制台（每个控制台至少需要显示一屏字符的显存）

####  初始化串口终端

接着初始化串行终端的tty结构各字段。

初始化串行终端tty结构(rs_table，其实是tty_table的第2个64项)，其中读缓冲、写缓冲和辅助缓冲队列结构，分别指向tty缓冲队列结构数组(rs_queues,其实是tty_queues的前25~30项) 中的相应结构项。rs_table个数就是NR_SERIALS(2)，这里看到rs_table至少有62项浪费了。

```
for (i = 0 ; i<NR_SERIALS ; i++) {
    rs_table[i] = (struct tty_struct) {
        {0, /* no translation */   // 输入模式标志集。0，无须转换。
        0,  /* no translation */   // 输出模式标志集。0，无须转换。
        B2400 | CS8,        // 控制模式标志集。2400bps，8位数据位。
        0,                         // 本地模式标志集。
        0,                         // 线路规程，0 -- TTY。
        INIT_C_CC},                // 控制字符数组。
        0,                         // 所属初始进程组。
        0,                         // 初始会话组。
        0,                         // 初始停止标志。
        rs_write,                  // 串口终端写函数。
        rs_queues+0+i*3,rs_queues+1+i*3,rs_queues+2+i*3  // 三个队列。
    };
}
```
####  初始化虚拟终端
 接着初始化伪终端使用的tty结构。伪终端是配对使用的，即一个主（master）伪终端配有一个从（slave）伪终端。因此对它们都要进行初始化设置。在循环中，我们首先初始化每个主伪终端的tty结构，然后再初始化其对应的从伪终端的tty结构。

初始化串行终端tty结构(mpty_table和spty_table，其实是tty_table的第3，4个64项)，其中读缓冲、写缓冲和辅助缓冲队列结构，分别指向tty缓冲队列结构数组(mpty_queues和spty_queues,其实是tty_queues的前31~54项) 中的相应结构项。mpty_table和spty_table个数就是NR_PTYS(4)，这里看到mpty_table和spty_table各有60项浪费了。

```
for (i = 0 ; i<NR_PTYS ; i++) {
    mpty_table[i] = (struct tty_struct) {
        {0, /* no translation */   // 输入模式标志集。0，无须转换。
        0,  /* no translation */   // 输出模式标志集。0，无须转换。
        B9600 | CS8,       // 控制模式标志集。9600bps，8位数据位。
        0,                         // 本地模式标志集。
        0,                         // 线路规程，0 -- TTY。
        INIT_C_CC},                // 控制字符数组。
        0,                         // 所属初始进程组。
        0,                         // 所属初始会话组。
        0,                         // 初始停止标志。
        mpty_write,                // 主伪终端写函数。
        mpty_queues+0+i*3,mpty_queues+1+i*3,mpty_queues+2+i*3
    };
    spty_table[i] = (struct tty_struct) {
        {0, /* no translation */   // 输入模式标志集。0，无须转换。
        0,  /* no translation */   // 输出模式标志集。0，无须转换。
        B9600 | CS8,       // 控制模式标志集。9600bps，8位数据位。
        IXON | ISIG | ICANON,      // 本地模式标志集。
        0,                         // 线路规程，0 -- TTY。
        INIT_C_CC},                // 控制字符数组。
        0,                         // 所属初始进程组。
        0,                         // 所属初始会话组。
        0,                         // 初始停止标志。
        spty_write,                // 从伪终端写函数。
        spty_queues+0+i*3,spty_queues+1+i*3,spty_queues+2+i*3
    };
}
```
####  串行中断处理程序
最后初始化串行中断处理程序和串行接口1和2（serial.c，37行）

下面两句用于设置两个串行口的中断门描述符。rs1_interrupt是串口1的中断处理过程指针。
串口1使用的中断是int 0x24，串口2的是int 0x23。参见表2-2和system.h文件。


```
rs_init();

```

```
void rs_init(void)
{
        set_intr_gate(0x24,rs1_interrupt);   // 设置串行口1的中断门向量(IRQ4信号)。
        set_intr_gate(0x23,rs2_interrupt);   // 设置串行口2的中断门向量(IRQ3信号)。
        init(tty_table[64].read_q->data);    // 初始化串行口1(.data是端口基地址)。
        init(tty_table[65].read_q->data);    // 初始化串行口2。
        outb(inb_p(0x21)&0xE7,0x21);         // 允许主8259A响应IRQ3、IRQ4中断请求。
}
```
这里串口中断很重要：
1. 借用table_list结构的内容，定位串口相关信息和缓存位置；
    1. 端口1中断，则读table_list.rs_queues + 0;
    2. 端口2中断，则都table_list.rs_queues + 3
2. 根据rs_queues中信息，2个串口的基地址`rs_addr`是（端口地址，不是内存地址）分别为0x3f8和0x2f8。中断围绕串口端口工作

    1. 0x3f8(0x2f8) rs_addr + 0: 接收缓冲寄存器
    2. 0x3fa(0x2fa) rs_addr + 1: 中断允许寄存器IER(如：发送保持寄存器空中断（位1）)
    2. 0x3fa(0x2fa) rs_addr + 2: 中断标识寄存器
    3. 0x3fd(0x2fd) rs_addr + 5: 线路状态寄存器
    3. 0x3fe(0x2fe) rs_addr + 6: modem状态寄存器
```
rs_queues[0] = (struct tty_queue) {0x3f8,0,0,0,""};
rs_queues[1] = (struct tty_queue) {0x3f8,0,0,0,""};
rs_queues[3] = (struct tty_queue) {0x2f8,0,0,0,""};
rs_queues[4] = (struct tty_queue) {0x2f8,0,0,0,""};
```
3. 当中断标识寄存器为读时，将接收缓冲寄存器中读处，并写入对应的读tty_queue的buf；并调用do_tty_interrupt将读缓冲区中字符经过转换，写入对应的辅助tty_queue。这里do_tty_interrupt是一个关键操作，其在键盘终端里也涉及到了。(键盘终端有写么？)
4. 当中断标识寄存器为写时，从对应的tty_queue的buf，读1个字符，并写入接收缓冲寄存器

### 从CMOS读取时钟，存入内存；
### 内核调度程序的初始化子程序；
    - 任务槽占 4 *６４　字节；
    - 初始任务占 4K 字节；
    - 内核栈占4K字节

 <span id="buffer_init"></span>
### 缓冲区初始化(buffer_init)
- 缓冲区的起始位置是由编译时的连接程序ld生成，用于表明内核代码的末端，即指明内核模块末端位置；
- 缓冲区结束位置则在main中定义过，这里是16M内存的4M处。
- 其在起始位置开始建立buffer_head，并且在尾部取BLOCK_SIZE和b_data关联(循环取，直到内存分完)。
```
#define BLOCK_SIZE 1024                         // 数据块长度（字节值）。

struct buffer_head {
char * b_data;                  /* pointer to data block (1024 bytes) */ //指针。
unsigned long b_blocknr;        /* block number */       // 块号。
unsigned short b_dev;           /* device (0 = free) */  // 数据源的设备号。
unsigned char b_uptodate;       // 更新标志：表示数据是否已更新。
unsigned char b_dirt;           /* 0-clean,1-dirty */ //修改标志:0未修改,1已修改.
unsigned char b_count;          /* users using this block */   // 使用的用户数。
unsigned char b_lock;           /* 0 - ok, 1 -locked */  // 缓冲区是否被锁定。
struct task_struct * b_wait;    // 指向等待该缓冲区解锁的任务。
struct buffer_head * b_prev;    // hash队列上前一块（这四个指针用于缓冲区的管理）。
struct buffer_head * b_next;    // hash队列上下一块。
struct buffer_head * b_prev_free;   // 空闲表上前一块。
struct buffer_head * b_next_free;   // 空闲表上下一块。
};
```
- 缓冲区按照 <buffer_head  + buf[BLOCK_SIZE]> 分块，并且构建了环链表。并将buffer_head首位连接成环
```
// 下面h是最后分配的buffer_head
free_list = start_buffer;           // 让空闲链表头指向头一个缓冲块。
free_list->b_prev_free = h;  // 链表头的b_prev_free指向前一项（即最后一项）。
h->b_next_free = free_list;         // h的下一项指针指向第一项，形成一个环链。
```

将起始位置保存在free_list；并将已分配的块加入哈希表hash_table(这部分初始化阶段初始化为NULL)。
```
extern int end;
struct buffer_head * start_buffer = (struct buffer_head *) &end;
struct buffer_head * hash_table[NR_HASH];        // NR_HASH = 307项。
static struct buffer_head * free_list;           // 空闲缓冲块链表头指针。
```


```
// 最后初始化hash表（哈希表、散列表），置表中所有指针为NULL。
for (i=0;i<NR_HASH;i++)
        hash_table[i]=NULL;
```
### 硬盘系统初始化
设置硬盘中断描述符，并允许硬盘控制器发送中断请求信号。
- 设置硬盘设备的请求项处理函数指针为do_hd_request()
- 然后设置硬盘中断门描述符。hd_interrupt（kernel/sys_call.s，第235行）是其中断处理过程地址。 硬盘中断号为int 0x2E（46），对应8259A芯片的中断请求信号IRQ13。
- 接着复位接联的主8259A int2的屏蔽位，允许从片发出中断请求信号。
- 再复位硬盘的中断请求屏蔽位（在从片上），允许硬盘控制器发送中断请求信号。中断描述符表IDT内中断门描述符设置宏set_intr_gate()
```
// 在include/asm/system.h中实现。
void hd_init(void)
{
blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;     // do_hd_request()。
set_intr_gate(0x2E,&hd_interrupt);  // 设置中断门中处理函数指针。
outb_p(inb_p(0x21)&0xfb,0x21);      // 复位接联的主8259A int2的屏蔽位。
outb(inb_p(0xA1)&0xbf,0xA1);        // 复位硬盘中断请求屏蔽位（在从片上）。
}
```
 <span id="硬盘设备的请求项处理函数"></span>
#### 硬盘设备的请求项处理函数(do_hd_request)
硬盘设备请求属于块设备请求，详细过程见[块设备访问](#块设备访问)
- 该函数根据设备当前请求项中的设备号和起始扇区号信息首先计算得到对应硬盘上的柱面号、当前磁道中扇区号、磁头号数据，然后再根据请求项中的命令（READ/WRITE）对硬盘发送相应 读/写命令。 若控制器复位标志或硬盘重新校正标志已被置位，那么首先会去执行复位或重新校正操作。
- 若请求项此时是块设备的第1个（原来设备空闲），则块设备当前请求项指针会直接指向该请求项（参见ll_rw_blk.c，28行），并会立刻调用本函数执行读写操作。否则在一个读写操作完成而引发的硬盘中断过程中，若还有请求项需要处理，则也会在硬盘中断过程中调用本函数。
// 参见kernel/sys_call.s，235行。

1. 函数首先检测请求项的合法性。若请求队列中已没有请求项则退出（参见blk.h，127行）。
2. 然后取设备号中的子设备号（见列表后对硬盘设备号的说明）以及设备当前请求项中的起始扇区号。子设备号即对应硬盘上各分区。如果子设备号不存在或者起始扇区大于该分区扇区数-2(因为一次要求读写一块数据（2个扇区，即1024字节）)，则结束该请求项，并跳转到标号repeat处（定义在INIT_REQUEST开始处）。
3. 然后通过加上子设备号对应分区的起始扇区号，就把需要读写的块对应到整个硬盘的绝对扇区号block上。
4. 而子设备号被5整除即可得到对应的硬盘号。
5. 然后根据求得的绝对扇区号block和硬盘号dev，我们就可以计算出对应硬盘中的磁道中扇区号（sec）、所在柱面号（cyl） 和磁头号（head）。 下面嵌入的汇编代码即用来根据硬盘信息结构中的每磁道扇区数和硬盘磁头数来计算这些数据。计算方法为：
		7. 310--311行代码初始时eax是扇区号block，edx中置0。divl指令把edx:eax组成的扇区号除以每磁道扇区数（hd_info[dev].sect），所得整数商值在eax中，余数在edx中。其中eax中是到指定位置的对应总磁道数（所有磁头面），edx中是当前磁道上的扇区号。
		8. 312--313行代码初始时eax是计算出的对应总磁道数，edx中置0。divl指令把edx:eax的对应总磁道数除以硬盘总磁头数（hd_info[dev].head），在eax中得到的整除值是柱面号（cyl），edx中得到的余数就是对应得当前磁头号（head）(感觉这里是当前柱面的磁道号)。
```
__asm__("divl %4":"=a" (block),"=d" (sec):"0" (block),"1" (0), "r" (hd_info[dev].sect));
__asm__("divl %4":"=a" (cyl),"=d" (head):"0" (block),"1" (0), "r" (hd_info[dev].head));
sec++;                               // 对计算所得当前磁道扇区号进行调整。
nsect = CURRENT->nr_sectors;         // 欲读/写的扇区数。
```
6. 此时我们得到了欲读写的硬盘起始扇区block所对应的硬盘上柱面号（cyl）、在当前磁道上的扇区号（sec）、磁头号（head）以及欲读写的总扇区数（nsect）。 
7. 接着我们可以根据这些信息向硬盘控制器发送I/O操作信息了。 但在发送之前我们还需要先看看是否有复位控制器状态和重新校正硬盘的标志。通常在复位操作之后都需要重新校正硬盘磁头位置。若这些标志已被置位，则说明前面的硬盘操作可能出现了一些问题，或者现在是系统第一次硬盘读写操作等情况。 于是我们就需要重新复位硬盘或控制器并重新校正硬盘。如果此时复位标志reset是置位的，则需要执行复位操作。复位硬盘和控制器，并置硬盘需要重新校正标志，返回。reset_hd()将首先向硬盘控制器发送复位（重新校正）命令，然后发送硬盘控制器命令“建立驱动器参数”。(这里面他妈有个循环调用，我们还是不管它了吧，因为reset默认是0)
8. 如果是硬盘读，则直接返回，真正的读操作应该是在中断里完成；
9. 如果是硬盘写，则返回后试探`HD_STATUS`一万次，如果其`DRQ_STAT`置位，则开始写往`HD_DATA`端口写一扇区数据。见[硬盘访问](#硬盘访问)

#### 硬盘中断(hd_interrupt)
当请求的硬盘操作完成或出错就会发出此中断信号。(参见kernel/blk_drv/hd.c)。
- 首先向8259A中断控制从芯片发送结束硬件中断指令(EOI)
- 然后取变量do_hd中的函数指针放入edx寄存器中，并置do_hd为NULL，接着判断edx函数指针是否为空。
	- 如果为空，则给edx赋值指向unexpected_hd_interrupt()，用于显示出错信息。
	- 随后向8259A主芯片送EOI指令，并调用edx中指针指向的函数: read_intr()、write_intr()或unexpected_hd_interrupt()。

可见：硬盘中断函数的实现，本身只实现了堆栈切换(中断本身)和硬盘状态修改(EOI)，核心操作是do_hd，其是[硬盘设备的请求项处理函数](#硬盘设备的请求项处理函数)在调用hd_out时设置(该设置和硬件无关，纯软件功能)
### 软驱初始化
设置硬盘中断描述符，并允许硬盘控制器发送中断请求信号。
- 设置软盘块设备请求项的处理函数do_fd_request()
- 设置软盘中断门（int 0x26，对应硬件中断请求信号IRQ6）
- 然后取消对该中断信号的屏蔽，以允许软盘控制器FDC发送中断请求信号。

中断描述符表IDT中陷阱门描述符设置宏set_trap_gate()定义在头文件
```
// include/asm/system.h中。
void floppy_init(void)
{
// 设置软盘中断门描述符。floppy_interrupt（kernel/sys_call.s，267行）是其中断处理
// 过程。中断号为int 0x26（38），对应8259A芯片中断请求信号IRQ6。
	blk_size[MAJOR_NR] = floppy_sizes;
	blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;  // = do_fd_request()。
	set_trap_gate(0x26,&floppy_interrupt);          // 设置陷阱门描述符。
	outb(inb_p(0x21)&~0x40,0x21);                   // 复位软盘中断请求屏蔽位。
}
```
 <span id="软盘设备的请求项处理函数"></span>
#### 软盘设备的请求项处理函数(do_fd_request)
该函数是软盘驱动程序中最主要的函数。主要作用是：
- 处理有复位标志或重新校正标志置位情况
- 利用请求项中的设备号计算取得请求项指定软驱的参数块
- 利用内核定时器启动软盘读/写操作
- 定时器超时则调用[floppy_on_interrupt](#floppy_on_interrupt)

所以，软盘设备请求，是计算好软盘读写的参数(全局变量保存)后设置一个定时器，定时器超时则调用指定函数。
phoenix：2020.02.05 深夜 这里有个问题，do_fd_request没有和软盘做任何交互，软盘怎么知道启动马达旋转。

//参见floppy.c
1. 首先利用blk.h文件中的INIT_REQUEST宏来检测请求项的合法性，如果已没有请求项则退出（参见blk.h,127）。
2. 然后利用请求项中的设备号取得请求项指定软驱的参数块。这个参数块将在下面用于设置软盘操作使用的全局变量参数块（参见112 - 122行）。
3. 请求项设备号中的软盘类型 (MINOR(CURRENT->dev)>>2) 被用作磁盘类型数组floppy_type[]的索引值来取得指定软驱的参数块。
4. 如果当前驱动器号current_drive不是请求项中指定的驱动器号，则置标志seek，表示在执行读/写操作之前需要先让驱动器执行寻道处理。然后把当前驱动器号设置为请求项中指定的驱动器号。
5. 设置读写起始扇区block。因为每次读写是以块为单位（1块为2个扇区），所以起始扇区需要起码比磁盘总扇区数小2个扇区。否则说明这个请求项参数无效，结束该次软盘请求项去执行下一个请求项。(~~和硬盘相比，软盘没有分区和多柱面多磁头~~下一行就打脸了)
6. 再求对应在磁道上的扇区号、磁头号、磁道号、搜寻磁道号（对于软驱读不同格式的盘）
7. 再看看是否还需要首先执行寻道操作。如果寻道号与当前磁头所在磁道号不同，则需要进行寻道操作，于是置需要寻道标志seek。
8. 最后我们设置执行的软盘命令command
9. 设置好所有全局变量值之后，我们可以开始执行请求项操作了。该操作利用定时器来启动。因为为了能对软驱进行读写操作，需要首先启动驱动器马达并达到正常运转速度。而这需要一定的时间。因此这里利用 ticks_to_floppy_on() 来计算启动延时时间，然后使用该延时设定一个定时器。当时间到时就调用函数floppy_on_interrupt()。

 <span id="floppy_on_interrupt"></span>
##### floppy_on_interrupt
在执行一个请求项要求的操作之前，为了等待指定软驱马达旋转起来到达正常的工作转速，do_fd_request()函数为准备好的当前请求项添加了一个延时定时器。本函数即是该定时器到期时调用的函数。
- 首先检查数字输出寄存器(DOR)，使其选择当前指定的驱动器。
如果当前驱动器号与数字输出寄存器DOR中的不同，则需要重新设置DOR为当前驱动器。在向数字输出寄存器输出当前DOR以后，使用定时器延迟2个滴答时间，以让命令得到执 行。
- 然后 调用执行软盘读写传输函数transfer()。
```
static void floppy_on_interrupt(void)           // floppy_on() interrupt。
{
	selected = 1;        // 置已选定当前驱动器标志。
	if (current_drive != (current_DOR & 3)) {
	    current_DOR &= 0xFC;
	    current_DOR |= current_drive;
	    // 向数字输出寄存器输出当前DOR。
	    outb(current_DOR,FD_DOR); 
	    // 添加定时器并执行传输函数。     
	    add_timer(2,&transfer);        
	} else
	    transfer();        // 执行软盘读写传输函数。
}
```

##### transfer
读写数据传输函数。
该函数是在传输操作的所有信息都正确设置好后被调用的（即软驱马达已开启并且已选择了正确的软盘（软驱）。
- 首先检查当前驱动器参数是否就是指定驱动器的参数。若不是就发送设置驱动器参数命令及相应参数（参数1：高4位步进速率，低四位磁头卸载时间；参数2：磁头加载时间）。
```
if (cur_spec1 != floppy->spec1) {  // 检测当前参数。
	cur_spec1 = floppy->spec1;
	output_byte(FD_SPECIFY);   // 发送设置磁盘参数命令
	output_byte(cur_spec1);   /* hut etc 发送参数*/
	output_byte(6);  /* Head load time =6ms, DMA */
}
```
- 然后判断当前数据传输速率是否与指定驱动器的一致，若不是就发送指定软驱的速率值到数据传输速率控制寄存器(FD_DCR)。
```
if (cur_rate != floppy->rate)    // 检测当前速率。
	outb_p(cur_rate = floppy->rate,FD_DCR);
```
- 若上面任何一个output_byte()操作执行出错，则复位标志reset就会被置位。因此这里我们需要检测一下reset标志。若reset真的被置位了，就立刻去执行do_fd_request()中的复位处理代码。
```
if (reset) {
	do_fd_request();
	return;
}
```
- 如果此时寻道标志为零（即不需要寻道），则设置DMA并向软盘控制器发送相应操作命令和参数后返回。[setup_rw_floppy](#setup_rw_floppy)
```
if (!seek) {
	setup_rw_floppy();         // 发送命令参数块。
	return;
}
```
- 否则就执行寻道处理，于是首先置软盘中断处理调用函数为寻道中断函数[seek_interrupt](#seek_interrupt)。如果起始磁道号不等于零则发送磁头寻道命令和参数。所使用的参数即是第112--121行上设置的全局变量值。如果起始磁道号seek_track为0，则执行重新校正命令让磁头归零位。
```
do_floppy = seek_interrupt;    // 寻道中断调用的C函数。
if (seek_track) {              // 起始磁道号。
	output_byte(FD_SEEK);      // 发送磁头寻道命令。
	// 发送参数：磁头号+当前软驱号。
	output_byte(head<<2 | current_drive);
	output_byte(seek_track);   // 发送参数：磁道号。
} else {
    // 发送重新校正命令（磁头归零）。
	output_byte(FD_RECALIBRATE); 
	// 发送参数：磁头号+当前软驱号。 
	output_byte(head<<2 | current_drive);
}
```
- 同样地，若上面任何一个output_byte()操作执行出错，则复位标志reset就会被置位。若reset真的被置位了，就立刻去执行do_fd_request()中的复位处理代码。

 <span id="setup_rw_floppy"></span>
##### setup_rw_floppy
设置DMA通道2并向软盘控制器输出命令和参数（输出1字节命令 + 0~7字节参数）。若reset标志没有置位，那么在该函数退出并且软盘控制器执行完相应读/写操作后就会产生一个软盘中断请求，并开始执行软盘中断处理程序。

所以：setup_rw_floppy首先设置DMA通道参数，然后预设[软盘中断回调函数](#floppy_interrupt)的回调函数[软盘读写操作中断调用函数](#rw_interrupt)，最后向软盘发送命令后退出(异常场景就不关注了)。


 <span id="seek_interrupt"></span>
##### seek_interrupt
寻道处理结束后中断过程中调用的C函数。其实此处并不关心实现。但有个重点是：软盘所有操作，都是先设置外设端口参数，然后退出等待中断。中断被调用后处理后续操作。

也就是说，这个时期的外设(至少软盘是吧)，是通过多次：
1. 设置当前需要外设执行某操作后的回调；
2. 从cpu设置参数到外设端口触发外设执行某操作(cpu返回)
3. 外设完成操作后触发中断
4. 中断处理函数调用第1步设置的函数

在一次块设备请求过程中，如本例的do_fd_request，可以多次设置软盘执行操作，并等待操作完成后中断中回调后续处理，然后重新调用do_fd_request重新开始执行，**直到真正完成了块设备请求的操作后，执行end_request来结束当前request**

 <span id="floppy_interrupt"></span>
#### 软盘中断(floppy_interrupt)
其处理过程与上面对硬盘的处理基本一样。（kernel/blk_drv/floppy.c）。
- 首先向8259A中断控制器主芯片发送EOI指令
- 然后取变量do_floppy中的函数指针放入eax寄存器中，并置do_floppy为NULL，接着判断eax函数指针是否为空。
	- 如为空，则给eax赋值指向unexpected_floppy_interrupt ()，用于显示出错信息。
	- 随后调用eax指向的函数: rw_interrupt,seek_interrupt,recal_interrupt,reset_interrupt或unexpected_floppy_interrupt。



<span id="rw_interrupt"></span>
##### rw_interrupt
软盘读写操作中断调用函数。
该函数在软驱控制器操作结束后引发的中断处理过程中被调用。函数首先读取操作结果状态信息，据此判断操作是否出现问题并作相应处理。 如果读/写操作成功，那么若请求项是读操作并且其缓冲区在内存1MB以上位置，则需要把数据从软盘临时缓冲区复制到请求项的缓冲区。

注意：如果当前请求项的缓冲区位于1MB地址以上，则说明此次软盘读操作的内容还放在临时缓冲区内，需要复制到当前请求项的缓冲区中（因为DMA只能在1MB地址范围寻址）。这里tmp_floppy_area的用途明确了。
注意2：其实，这里缓冲区位于1MB地址以上，还说明本次说读操作；因为写操作在[setup_rw_floppy](#setup_rw_floppy)第一步，设置DMA参数时，如果是写且缓冲区其实大于1M，则拷贝到tmp_floppy_area
```
if (addr >= 0x100000) {
  addr = (long) tmp_floppy_area;
  if (command == FD_WRITE)
   copy_buffer(CURRENT->buffer,tmp_floppy_area);
}
```
重要：最后，执行当前请求项结束处理：唤醒等待该请求项的进行，唤醒等待空闲请求项的进程（若有的话），从软驱设备请求项链表中删除本请求项。再继续执行其他软盘请求项操作。



### 开启中断
所有初始化工作都做完了，于是开启中断；

## 进程0进入用户态 

13. 移到用户模式下执行
    - 这里将LDT中数据段段选择符入栈（SS，在initTassk中定义的LDT表中，从0开始的640K区间，但是段属性变成0x07）；调用前的SP入栈（iret成功跳转后这段栈内容丢失了，因为SP跳回之前的位置）；一个有趣的现象（initTask在正式进入用户态后，它的堆栈用的是之前的堆栈，而它再次切换回内核态时，其堆栈反而是initTask页）																
    - efalgs入栈；LDT中代码段选择符入栈（CS，在initTassk中定义的LDT表中，从0开始的640K区间，但是段属性变成0x07）；iret后的指令地址入栈（EIP，用户态从此开始执行）																
    - 通过iret中断返回指令，跳转；此时堆栈中刚刚压入的信息弹回各寄存器，栈顶指回此宏函数执行前的位置，且ESP跳到指定位置；但是当前代码运行在用户态；																
    - 将LDT的数据段段选择符赋值给ds，es，fs，gs，用户态切换完成；		
 14. fork()
- 其实就是 int 0x80：eax=2；
    - int本身压栈了ss/esp/eflags/cs/eip；system_call又压栈ds/es/fs/eax/edx/ecx/ebx,且将0x10段加载ds/ex,0x17加载fs；
        - call调用2号软中断（fork），EIP压栈；
            - call调用find_empty_process（C函数）；进程槽获取一个空进程号；
            - 压栈gs/esi/edi/ebp/eax；然后call调用copy_process（C函数）；
                - 申请一页内存存储task信息，将current内容复赋值到新内存页，并修改各字段（内核栈SS0指向0x10段，SP0指向新页页尾，eip指向返回地址，其他寄存器和当前进程一致）
                - C调用copy_mem复制页表；设置基址为64M起始；
                    - 将新基址填入新任务信息结构ldt[1/2]；
                    - 拷贝页表；申请一张新页面为页表，在页目录表中跨64M的页目录项指向此页表，并从0页目录项指向的0页表（0页表的0表项指向0页面，该页面其实存的是页目录），拷贝0页表的160项到新页表（64M开始的640K线性地址和0M开始的640K地址指向相同的640K物理内存）；
                    - 把新旧表项改成只读（==这里其实只让新表的表项是只读，旧表只有1M以上地址只读。内核态的时候其实不会使用高位段？==）；
                    - 将1M以上物理页设置引用计数加1；
                - 把新任务ldt和tss地址填入GDT对应的选择符中；
                - 把新任务的状态改为TASK_RUNNING，然后返回；																
            - esp增加20字节（跳过上一步压栈内容）然后ret
        - eax结果入栈，进入ret_from_sys_call
    - 由于是task0，所以不做信号处理等操作；eax出栈；ebx/ecx/edx出栈，跳过4字节（丢弃了原eax，eax要保存系统调用结果），fs/es/dsc出栈；iret返回		

- fork结果不等于0，进入pause死循环	


## 小结(遗留问题)
此节有2个要点，进程0描述信息的初始化，从系统态切换用户态：

## 进程0描述符的初始化

进程0描述符的初始化如下，我们观察到一个现象，进程描述符的tss字段，保存了进程用户态的寄存器和堆栈信息，而仅保存了系统态的堆栈信息，而没有保存CS段寄存器内容。这里就比较有意思了，难道系统态和用户态共用了CS,EIP？

- 进程0描述符基本信息的初始化
进程0的进程描述结构初始化仅初始化了少数字段
1. 时间计数和优先级默认为15；
2. 父进程指针指向自己；
3. 重点是后续的ldt和tss初始化

```
//linux/include/linux/sched.h
0                           //long state                    任务的运行状态（-1不可运行，0可运行(就绪)，>0已停止）。
15                          //long counter                  任务运行时间计数(递减)（滴答数），运行时间片。
15                          //long priority                 优先数。任务开始运行时counter=priority，越大运行越长。

0                           //long signal                   信号位图，每个比特位代表一种信号，信号值=位偏移值+1。
{{},}                       //struct sigaction sigaction[32]   信号执行属性结构，对应信号将要执行的操作和标志信息。
0                           //long blocked                  进程信号屏蔽码（对应信号位图）。
                            //-------------------
0                           //int exit_code                 任务执行停止的退出码，其父进程会取。
0                           //unsigned long start_code      代码段地址。
0                           //unsigned long end_code        代码长度（字节数）。
0                           //unsigned long end_data        代码长度 + 数据长度（字节数）。
0                           //unsigned long brk             总长度（字节数）。
0                           //unsigned long start_stack     堆栈段地址。

0                           //long pid                      进程标识号(进程号)。
0                           //long pgrp                     进程组号。
0                           //long session                  会话号。
0                           //long leader                   会话首领。

{NOGROUP,}                  //int groups[NGROUPS]           进程所属组号。一个进程可属于多个组。

&init_task.task             //task_struct *p_pptr           指向父进程的指针。
0                           //task_struct *p_cptr           指向最新子进程的指针。
0                           //task_struct *p_ysptr          指向比自己后创建的相邻进程的指针。
0                           //task_struct *p_osptr          指向比自己早创建的相邻进程的指针。

0                           //unsigned short uid            用户标识号（用户id）。
0                           //unsigned short euid           有效用户id。
0                           //unsigned short suid           保存的用户id。
0                           //unsigned short gid            组标识号（组id）。
0                           //unsigned short egid           有效组id。
0                           //unsigned short sgid           保存的组id。

0                           //long timeout                  内核定时超时值。
0                           //long alarm                    报警定时值（滴答数）。
0                           //long utime                    用户态运行时间（滴答数）。
0                           //long stime                    系统态运行时间（滴答数）。
0                           //long cutime                   子进程用户态运行时间。
0                           //long cstime                   子进程系统态运行时间。
0                           //long start_time               进程开始运行时刻。

{0x7fffffff, 0x7fffffff} *6 //struct rlimit rlim[RLIM_NLIMITS]  进程资源使用统计数组。

0                           //unsigned int flags;           各进程的标志，在下面第149行开始定义（还未使用）。
0                           //unsigned short used_math      标志：是否使用了协处理器。
                            //------------------------
-1                          //int tty                       进程使用tty终端的子设备号。-1表示没有使用。
0022                        //unsigned short umask          文件创建属性屏蔽位。
NULL                        //struct m_inode * pwd          当前工作目录i节点结构指针。
NULL                        //struct m_inode * root         根目录i节点结构指针。
NULL                        //struct m_inode * executable   执行文件i节点结构指针。
NULL                        //struct m_inode * library      被加载库文件i节点结构指针。
0                           //unsigned long close_on_exec   执行时关闭文件句柄位图标志。（参见include/fcntl.h）

{NULL,}                     //struct file * filp[NR_OPEN]   文件结构指针表，最多32项。表项号即是文件描述符的值。

如下：                       //struct desc_struct ldt[3]     局部描述符表。0-空，1-代码段cs，2-数据和堆栈段ds&ss。
如下：                       //struct tss_struct tss         进程的任务状态段信息结构。
                            //======================================
```
- 进程0描述符局部描述符表的初始化
1. 0 不适用，初始化为0；
2. 1 初始化为0~640k的代码段；
3. 2 初始化为0~640k的数据段；

```
//linux/include/linux/sched.h
//局部描述符表
                 {0,0}, \
 /* ldt */       {0x9f,0xc0fa00}, \  // 代码长640K，基址0x0，G=1，D=1，DPL=3，P=1 TYPE=0xa
                 {0x9f,0xc0f200}, \  // 数据长640K，基址0x0，G=1，D=1，DPL=3，P=1 TYPE=0x2
```
- 进程0描述符tss任务状态段信息的初始化
1. 内核栈栈顶指向init_task的结尾后一个字节；
2. 内核栈段段选择子指向GDT[2]（内核数据段起始地址，其实是0）;
3. cr3寄存器指向页目录起始地址（起始就是内存起始地址0）；
4. 所有段寄存器都指向局部描述符表LDT[2]
5. 局部段选择表ldt的GDT选择子指向进程0的局部段选择表（此时其实GDT中此位置还没有有效值，在sched_init中将上一步的初始值地址回填进GDT的）

```   
//linux/include/linux/sched.h
// tss 任务状态段信息结构。
0                             //long    back_link;      /* 16 high bits zero */
PAGE_SIZE+(long)&init_task    //long    esp0;
0x10                          //long    ss0;            /* 16 high bits zero */
0                             //long    esp1;
0                             //long    ss1;            /* 16 high bits zero */
0                             //long    esp2;
0                             //long    ss2;            /* 16 high bits zero */

&pg_dir                       //long    cr3;

0                             //long    eip;
0                             //long    eflags;
0,0,0,0                       //long    eax,ecx,edx,ebx;
0                             //long    esp;
0                             //long    ebp;
0                             //long    esi;
0                             //long    edi;
0                             //long    es;             /* 16 high bits zero */

0x17                          //long    cs;             /* 16 high bits zero */
0x17                          //long    ss;             /* 16 high bits zero */
0x17                          //long    ds;             /* 16 high bits zero */
0x17                          //long    fs;             /* 16 high bits zero */
0x17                          //long    gs;             /* 16 high bits zero */

_LDT(0)                       //long    ldt;            /* 16 high bits zero */
0x80000000                    //long    trace_bitmap;   /* bits: trace 0, bitmap 16-31 */
{}                            //struct i387_struct i387;
```

## 系统态切换用户态	
从系统态切换用户态，完成如下几步操作，这几个操作其实就是手动构造了一次用户态转系统态的入栈内容，然后通过iret执行中断返回指令，就切换到用户态了。其实用户态的准备工作在之前已经完成了，如ldt和tss的构造和选择子加载。
1. 堆栈段选择符(SS)入栈；
2. 堆栈指针(ESP)入栈；
3. 标志寄存器(eflags)入栈；
4. 代码段选择符(CS)入栈；
5. 本执行指令序列返回指令(iret)的后一条指令偏移(eip)入栈;
6. 通过iret执行中断返回指令，切换到用户态；
7. 在用户态下立即手动局部描述符2(LDT[2])的选择子载入附加的4个数据段寄存器

```
//linux/include/asm/system.h
// 移动到用户模式运行。
// 该函数利用iret指令实现从内核模式移动到初始任务0中去执行。
#define move_to_user_mode() \
__asm__ ("movl %%esp,%%eax\n\t" \      // 保存堆栈指针esp到eax寄存器中。
        "pushl $0x17\n\t" \            // 首先将堆栈段选择符(SS)入栈。
        "pushl %%eax\n\t" \            // 然后将保存的堆栈指针值(esp)入栈。
        "pushfl\n\t" \                 // 将标志寄存器(eflags)内容入栈。
        "pushl $0x0f\n\t" \            // 将Task0代码段选择符(cs)入栈。
        "pushl $1f\n\t" \              // 将下面标号1的偏移地址(eip)入栈。
        "iret\n" \                     // 执行中断返回指令，则会跳转到下面标号1处。
        "1:\tmovl $0x17,%%eax\n\t" \   // 此时开始执行任务0，
        "movw %%ax,%%ds\n\t" \         // 初始化段寄存器指向本局部表的数据段。
        "movw %%ax,%%es\n\t" \
        "movw %%ax,%%fs\n\t" \
        "movw %%ax,%%gs" \
        :::"ax")
```
 <span id="块设备访问"></span>
## 块设备访问
块设备的访问，是由几个数据结构和中断相互配合完成的。
有2个重点是：
1. 无论读写，发起的业务进程在把读写请求写入[块设备请求](#request)队列后就退出进入休眠态。等待读写完成后再被唤醒。读写的数据预设放在[缓冲区](#buffer_head)
2. 忘了(又想起来了，一个设备的请求是串行的，即只能一个请求一个请求处理。同一个请求可能分多段多次中断执行)

### 块设备访问数据结构

#### 块设备数组blk_dev
块设备数组。该数组使用主设备号作为索引。实际内容将在各块设备驱动程序初始化时填入。
```
#define NR_BLK_DEV      7          // 块设备类型数量。

struct blk_dev_struct blk_dev[NR_BLK_DEV] = {
  { NULL, NULL },  /* no_dev */   // 0 - 无设备
  { NULL, NULL },  /* dev mem */  // 1 - 内存
  { NULL, NULL },  /* dev fd */   // 2 - 软驱设备
  { NULL, NULL },  /* dev hd */   // 3 - 硬盘设备
  { NULL, NULL },  /* dev ttyx */ // 4 - ttyx设备
  { NULL, NULL },  /* dev tty */  // 5 - tty设备
  { NULL, NULL }   /* dev lp */   // 6 - lp打印机
};
```
块设备数组是针对每种块设备分配一个单元，块设备结构如下:
```
struct blk_dev_struct {
  // 请求处理函数指针
  void (*request_fn)(void);       
  // 当前处理的请求结构
  struct request * current_request;
};
```
这里注意到一个关键结构，[块设备请求](#request)

 <span id="request"></span>
#### 块设备请求request

```
#define NR_REQUEST      32

// 请求项数组队列。共有NR_REQUEST = 32个请求项。
struct request request[NR_REQUEST];
```

```
/*
 * OK，下面是request结构的一个扩展形式，因而当实现以后，我们
 * 就可以在分页请求中使用同样的 request 结构。 在分页处理中，
 * 'bh'是NULL，而'waiting'则用于等待读/写的完成。
 */
// 下面是请求队列中项的结构。其中如果字段dev = -1，则表示队列中该项没有被使用。
// 字段cmd可取常量 READ（0）或 WRITE（1）（定义在include/linux/fs.h中）。
// 其中，内核并没有用到waiting指针，起而代之地内核使用了缓冲块的等待队列。因为
// 等待一个缓冲块与等待请求项完成是对等的。
struct request {
	int dev;                       /* -1 if no request */  // 发请求的设备号。
	int cmd;                       /* READ or WRITE */  // READ或WRITE命令。
	int errors;                     //操作时产生的错误次数。
	unsigned long sector;           // 起始扇区。(1块=2扇区)
	unsigned long nr_sectors;       // 读/写扇区数。
	char * buffer;                  // 数据缓冲区。
	struct task_struct * waiting;   // 任务等待请求完成操作的地方（队列）。
	struct buffer_head * bh;        // 缓冲区头指针(include/linux/fs.h,68)。
	struct request * next;          // 指向下一请求项。
};
```

这里注意到一个关键结构，[缓冲区](#buffer_head)  和 [任务结构](#task_struct)
 <span id="buffer_head"></span>
#### 缓冲区及buffer_head
这里看到了前面[缓冲区初始化](#buffer_init)时分配的缓冲区及其环链管理队列buffer_head。显然，其在request被使用时分配。

 <span id="硬盘访问"></span>
### 硬盘访问

硬盘访问有个有趣的过程，当blk_dev的对应硬盘设备的处理函数`blk_dev[MAJOR_NR].do_hd_request`被调用时:
1. 其根据当前请求`blk_dev[MAJOR_NR].current_request`分析其请求内容(`struct request`)。计算出其请求的磁盘物理位置(硬盘，柱面，磁道，扇区)
2. 然后向硬盘控制器发送命令，这里的重点是`hd_out`,该命令初始化了硬盘控制器的一系列参数，并且为硬盘中断挂接了一个回调函数的指针(~~即硬盘操作是，先告诉硬盘有个操作要执行，然后硬盘准备好后触发一个中断，中断应该只是回调函数~~错误的理解，见[磁盘读写和中断的关系](#磁盘读写和中断的关系))
3.  如果是硬盘读，则直接返回，真正的读操作应该是在中断里完成；
4.  如果是硬盘写，则返回后试探`HD_STATUS`一万次，如果其`DRQ_STAT`置位，则开始写往`HD_DATA`端口写一扇区数据

#### read_intr
读操作中断调用函数。
该函数将在硬盘读命令结束时引发的硬盘中断过程中被调用。在读命令执行后会产生硬盘中断信号，并执行硬盘中断处理程序，此时在硬盘中断处理程序中调用的C函数指针do_hd已经指向read_intr()，因此会在一次读扇区操作完成（或出错）后就会执行该函数。
1. 从数据寄存器端口把1个扇区的数据读到请求项的缓冲区中
2. 并且递减请求项所需读取的扇区数值。
	3. 若递减后不等于 0，表示本项请求还有数据没取完，于是再次置中断调用C函数指针do_hd为read_intr()并直接返回，等待硬盘在读出另1个扇区数据后发出中断并再次调用本函数。(再次置do_hd指针指向read_intr()是因为硬盘中断处理程序每次调用do_hd时都会将该函数指针置空。)
	4. 本次请求项的全部扇区数据已经读完，则调用end_request()函数去处理请求项结束事宜。 最后再次调用 do_hd_request()，去处理其他硬盘请求项。执行其他硬盘请求操作。

#### write_intr
写扇区中断调用函数。
该函数将在硬盘写命令结束时引发的硬盘中断过程中被调用。函数功能与read_intr()类似。
在写命令执行后会产生硬盘中断信号，并执行硬盘中断处理程序，此时在硬盘中断处理程序中调用的C函数指针do_hd已经指向write_intr()，因此会在一次写扇区操作完成（或出错）后就会执行该函数。
1. 读硬盘状态寄存器判断状态，如果其非完成状态，则处理异常，并再次调用do_hd_request后返回(怎么感觉又是个循环调用)
2. 硬盘状态正常，说明本次写一扇区操作成功，因此将欲写扇区数减1。
	3. 若其不为0，则说明还有扇区要写，于是把当前请求起始扇区号 +1，并调整请求项数据缓冲区指针指向下一块欲写的数据。然后再重置硬盘中断处理程序中调用的C函数指针do_hd（指向本函数）。接着向控制器数据端口写入512字节数据，然后函数返回去等待控制器把这些数据写入硬盘后产生的中断。
	4. 若本次请求项的全部扇区数据已经写完，则调用end_request()函数去处理请求项结束事宜。最后再次调用 do_hd_request()，去处理其他硬盘请求项。执行其他硬盘请求操作。

<span id="磁盘读写和中断的关系"></span>
由上所以，磁盘写请求发送后，并不是等中断完成写，而是等磁盘状态达到预期后开始写，中断只是判断是否需要继续写；而读则不同，读要等到中断来才在中断处理程序中读。为什么硬盘有不同的操作呢。硬盘准备好以后发中断不是更合理统一么。
phoenix：2020.2.3日凌晨

 <span id="字符设备访问"></span>
## 字符设备访问
字符设备最典型的就是键盘和串口了。一个有意思的地方在于:
键盘输入的字符，只有0xe0或者0xe1时候才触发[do_tty_interrupt](#do_tty_interrupt)；而串口设备在读串口时直接调用该函数。

### 字符设备访问数据结构
 
 <span id="table_list"></span>
####tty读写缓冲队列结构地址表table_list
tty读写缓冲队列结构地址表。供rs_io.s程序使用，用于取得读写缓冲队列结构的地址。
```
int fg_console = 0;              // 当前前台控制台号（范围0--7）。

struct tty_queue * table_list[]={
        con_queues + 0, con_queues + 1,       // 前台控制台读、写队列结构地址。
        rs_queues + 0, rs_queues + 1,         // 串行终端1读、写队列结构地址。
        rs_queues + 3, rs_queues + 4          // 串行终端2读、写队列结构地址。
        };
```
注意：因为默认控制台是0，所以以上table_list的时候，前台控制台读写队列地址给的是0号控制台的读写队列，如果切换控制台，table_list前2个字段也需要修改。

 <span id="tty_table"></span>
#### tty结构tty_table
按照终端类型分类，将tty终端表结构数组tty_table划分4块，
1. 8个虚拟控制台终端可用tty_table[]数组开头64项（0 -- 63），其实只用了8项；
2. 2个串行终端使用随后的64项（64 -- 65），其实只用了2项;
3. 4个主伪终端使用从128开始的项，最多64项（128 -- 191），其实只用了4项;
4. 4个从伪终端使用从192开始的项，最多64项（192 -- 255），其实只用了4项。

```
struct tty_struct tty_table[256];                        // tty表结构数组。

#define con_table tty_table                // 定义控制台终端tty表符号常数。
#define rs_table (64+tty_table)            // 串行终端tty表。
#define mpty_table (128+tty_table)         // 主伪终端tty表。
#define spty_table (192+tty_table)         // 从伪终端tty表。
```
tty数据结构。
```
struct tty_struct {
	struct termios termios;                   // 终端io属性和控制字符数据结构。
	int pgrp;                                 // 所属进程组。
	int session;                              // 会话号。
	int stopped;                              // 停止标志。
	void (*write)(struct tty_struct * tty);   // tty写函数指针。
	struct tty_queue *read_q;                 // tty读队列。
	struct tty_queue *write_q;                // tty写队列。
	struct tty_queue *secondary;              // tty辅助队列(存放规范模式字符序列)，
};     
```

#### tty字符缓冲队列数据结构tty_queue 
```
#define MAX_CONSOLES    8         // 最大虚拟控制台数量
#define NR_SERIALS      2         // 串行终端数量
#define NR_PTYS         4         // 伪终端数量

#define QUEUES  (3*(MAX_CONSOLES+NR_SERIALS+2*NR_PTYS))  // 共54项。

static struct tty_queue tty_queues[QUEUES];              // tty缓冲队列数组。
```

tty字符缓冲队列数据结构。用于tty_struc结构中的读、写和辅助（规范）缓冲队列。
```
struct tty_queue {
	unsigned long data;              // 队列缓冲区中含有字符行数值（不是当前字符数）。
	                                 // 对于串口终端，则存放串行端口地址。
	unsigned long head;              // 缓冲区中数据头指针。
	unsigned long tail;              // 缓冲区中数据尾指针。
	struct task_struct * proc_list;  // 等待本队列的进程列表。
	char buf[TTY_BUF_SIZE];          // 队列的缓冲区。
};
```

### 键盘访问
字符设备最显著的代表就是键盘设备，如上[table_list](#table_list)定义，其第一个字段就被[键盘中断](#键盘中断)所使用。

注意：键盘没有写，但是有回显。所以在读键盘的时候，回将需要回显的字符加入写队列。

键盘中断程序简单来说，功能比较简单。
1. 收到键盘中断；
2. 键盘中断程序扫描键盘控制器，读取扫描码
3. 根据扫描码值，设置控制状态，或者转换成字符；
4. 将键盘字符写入
	5. 首先从缓冲队列地址表table_list（tty_io.c，99行）取控制台的读缓冲队列read_q地址。
	6. 然后把al寄存器中的字符复制到读队列头指针处并把头指针前移1字节位置。若头指针移出读缓冲区的末端，就让其回绕到缓冲区开始处。 然后再看看此时缓冲队列是否已满，即比较一下队列头指针是否与尾指针相等（相等表示满）。 如果已满，就把ebx:eax中可能还有的其余字符全部抛弃掉。如果缓冲区还未满，就把ebx:eax中数据联合右移8个比特（即把ah值移到al、blàah、bhàbl），然后重复上面对al的处理过程。直到所有字符都处理完后，就保存当前头指针值，再检查一下是否有进程等待着读队列，如果有救唤醒之。

注意：~~键盘中断并没有使用[tty_table](#tty_table)结构~~应该说它扫描到普通字符的时候，它始终将字符写入[table_list](#table_list)的第一项指向的[tty_queue](#tty_queue)(当前控制台的读队列)

注意，在上面第3步，当发现扫描到0xe1或者0xe0时，会调用do_tty_interrupt

### 串口访问
这里串口中断很重要：
1. 借用table_list结构的内容，定位串口相关信息和缓存位置；
    1. 端口1中断，读table_list.rs_queues + 0，写table_list.rs_queues + 1
    2. 端口2中断，读table_list.rs_queues + 3，写table_list.rs_queues + 4
2. 根据rs_queues中信息，2个串口的基地址`rs_addr`是（端口地址，不是内存地址）分别为0x3f8和0x2f8。中断围绕串口端口工作

    1. 0x3f8(0x2f8) rs_addr + 0: 接收缓冲寄存器
    2. 0x3fa(0x2fa) rs_addr + 1: 中断允许寄存器IER(如：发送保持寄存器空中断（位1）)
    2. 0x3fa(0x2fa) rs_addr + 2: 中断标识寄存器
    3. 0x3fd(0x2fd) rs_addr + 5: 线路状态寄存器
    3. 0x3fe(0x2fe) rs_addr + 6: modem状态寄存器
```
rs_queues[0] = (struct tty_queue) {0x3f8,0,0,0,""};
rs_queues[1] = (struct tty_queue) {0x3f8,0,0,0,""};
rs_queues[3] = (struct tty_queue) {0x2f8,0,0,0,""};
rs_queues[4] = (struct tty_queue) {0x2f8,0,0,0,""};
```
3. 当中断标识寄存器为读时，将接收缓冲寄存器中读处，并写入对应的读tty_queue的buf；并调用do_tty_interrupt将读缓冲区中字符经过转换，写入对应的辅助tty_queue。这里do_tty_interrupt是一个关键操作，其在键盘终端里也涉及到了。(键盘终端有写么？)
4. 当中断标识寄存器为写时，从对应的tty_queue的buf，取1个字符，并写入接收缓冲寄存器

#### read_char
由于UART芯片接收到字符而引起这次中断。对接收缓冲寄存器执行读操作可复位该中断源。
这个子程序将接收到的字符放到读缓冲队列read_q头指针（head）处，并且让该指针前移一个字符位置。若head指针已经到达缓冲区末端，则让其折返到缓冲区开始处。最后调用C函数do_tty_interrupt()（也即copy_to_cooked()），把读入的字符经过处理放入规范模式缓冲队列（辅助缓冲队列secondary）中.



#### write_char


<span id="do_tty_interrupt"></span>
### 字符规范模式处理do_tty_interrupt
此函数在键盘中断和串口中断中都被调用
1. 收到键盘0xe0或者0xe1；
2. 从串口读字符时

do_tty_interrupt其实是调用的copy_to_cooked方法，其功能如下：
1. 循环提取读队列中字符，并处理；
2. 如果字符是换行或者回车，则检查是否需要转换，如果是则转换；
3. 字符是否需要转换为小写字符，如果是则转换；
4. 如果此tty取规范模式标志置位：
	5. 如果该字符是键盘终止控制字符KILL(^U)，如下处理后continue；
		6. 将tty辅助队列头指针后退1字节(从tty->secondary删除一个有效字符)
		7. `如果回显标志置位则往写队列写127(擦除控制字符ERASE（^H）),并调tty->write把写队列中的所有字符输出到终端屏幕上(控制台是con_write，串口是rs_write)`
	6. 如果该字符是删除控制字符ERASE（^H），如下处理后continue；
		7. 将tty辅助队列头指针后退1字节(从tty->secondary删除一个有效字符)
		8. `如果回显标志置位则往写队列写127(擦除控制字符ERASE（^H）),并调tty->write把写队列中的所有字符输出到终端屏幕上(控制台是con_write，串口是rs_write)`
6. 如果设置了IXON标志，则使终端停止/开始输出控制字符起作用，比如：
	7.  对于控制台来说，控制台将由于发现stopped=1而会立刻暂停在屏幕上显示新字符
	8. 对于伪终端也是由于设置了终端stopped标志而会暂停写操作
	9. 对于串行终端，也应该在发送终端过程中根据终端stopped标志暂停发送，但本版未实现
10. 若输入模式标志集中ISIG标志置位，表示终端键盘可以产生信号，则在收到控制字符INTR、QUIT、SUSP 或 DSUSP 时，需要为进程产生相应的信号 
	11. 如果该字符是键盘中断符（^C），则向当前进程之进程组中所有进程发送键盘中断信号SIGINT，并继续处理下一字符
	12. 如果该字符是退出符（^\），则向当前进程之进程组中所有进程发送键盘退出信号SIGQUIT，并继续处理下一字符
	13. 如果字符是暂停符（^Z），则向当前进程发送暂停信号SIGTSTP
14. 如果该字符是换行符NL（10），或者是文件结束符EOF（4，^D），表示一行字符已处理完，则把辅助缓冲队列中当前含有字符行数值secondary.data增1。如果在函数tty_read()中取走一行字符，该值即会被减1，参见315行。
15. 如果本地模式标志集中回显标志ECHO在置位状态
	16. 如果字符是换行符NL（10），则将换行符NL（10）和回车符CR（13）放入tty写队列缓冲区中；
	17. 如果字符是控制字符（值<32）并且回显控制字符标志ECHOCTL置位，则将字符'^'和字符 c+64 放入tty写队列中（也即会显示^C、^H等)；
	18. `否则将该字符直接放入tty写缓冲队列中。最后调用该tty写操作函数`。
19. 每一次循环末将处理过的字符放入辅助队列中

最后在退出循环体后唤醒等待该辅助缓冲队列的进程（如果有的话）

####  con_write
控制台写函数
- 从终端对应的tty写缓冲队列中取字符，针对每个字符进行分析。
- 若是控制字符或转义或控制序列，则进行光标定位、字符删除等的控制处理；
- 对于普通字符就直接在光标处显示。

####  rs_write
串行数据发送输出
- 该函数实际上只是开启`发送保持寄存器`已空中断标志。
- 此后当`发送保持寄存器`空时，UART就会产生中断请求。
- 而在该串行中断处理过程中，程序会取出写队列尾指针处的字符，并输出到发送保持寄存器中。
- 一旦UART把该字符发送了出去，发送保持寄存器又会变空而引发中断请求。
- 于是只要写队列中还有字符，系统就会重复这个处理过程，把字符一个一个地发送出去。
- 当写队列中所有字符都发送了出去，写队列变空了，中断处理程序就会把中断允许寄存器中的`发送保持寄存器`中断允许标志复位掉，从而再次禁止发送保持寄存器空引发中断请求。此次“循环”发送操作也随之结束。


## 吐个槽
从03年在大分租的房子里看《AI32》到后来的《自己动手写操作系统》，再到《linux内核》，转转了15年，才终于看明白一些体系结构的东西，知道CPU支持的基本功能和操作系统在其上的设计原理。

有些东西到现在才知道，如：
1. 页目录其实所有进程共用的，所谓的进程独立页表，只是用户态的代码映射用，内核代码还是在其自己的位置，且进程切换到内核态后，段选择符使用的GDT[1],GDT[2];所以tss仅一个CS,EIP寄存器信息，而有2个堆栈信息；
2. 用户态切换到内核态时，CPU主动往内核态堆栈压栈了用户态时的5个寄存器信息(SS,sp,eflags,CS,eip);但是反方向时只需要iret就够，无需压栈；
3. 进程0点用户态堆栈是进入用户态之前的系统态（其实应该叫初始态）堆栈；而第一次进入用户态时，它的系统态堆栈是一个空栈，即下次进入系统态，堆栈一切从0开始(一个没有过往的状态)；
4. 进程发生切换时，jmp一个tss选择子；此操作cpu会把当前寄存器信息写入当前tss描述符内！！！


如果15年前能看到这里，也许一切都会不同~
我要去买包烟~




