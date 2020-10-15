# 练习一 #
## 1.2 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？ ##
查看sign.c代码
```
buf[510] = 0x55;
buf[511] = 0xAA;
FILE *ofp = fopen(argv[2], "wb+");
size = fwrite(buf, 1, 512, ofp);
if (size != 512) {
    fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
    return -1;
}
```
可以看出，要求磁盘主引导扇区的大小为512字节，并且最后两个字节为0x55AA

# 练习五 #
## 实现函数调用栈堆跟踪函数 ##
一个简单的栈模型如下：  
```
ss:[ebp-8]   ;变量2  
ss:[ebp-4]   ;变量1  
ss:[ebp]       ;栈针  
ss:[ebp+4]  ;返回地址  
ss:[ebp+8]   ;第一个参数  
```
代码如下：
```
291 void
292 print_stackframe(void) {
293      /* LAB1 YOUR CODE : STEP 1 */
294      /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
295       * (2) call read_eip() to get the value of eip. the type is (uint32_t);
296       * (3) from 0 .. STACKFRAME_DEPTH
297       *    (3.1) printf value of ebp, eip
298       *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [    0..4]
299       *    (3.3) cprintf("\n");
300       *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line numbe    r, etc.
301       *    (3.5) popup a calling stackframe
302       *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
303       *                   the calling funciton's ebp = ss:[ebp]
304       */
305     uint32_t ebp = read_ebp();
306     uint32_t eip = read_eip();
307     for(int i = 0; i < STACKFRAME_DEPTH && ebp != 0; ++ i){
308         cprintf("ebp:0x%08x eip:0x%08x ", ebp, eip);
309         uint32_t *tmp = (uint32_t *)ebp + 2;
310         cprintf("args: 0x%08x 0x%08x 0x%08x 0x%08x", *tmp, *(tmp + 1), *(tmp + 2), *(tmp + 3));
311         cprintf("\n");
312         print_debuginfo(eip - 1);
313         ebp = *((uint32_t *)ebp);
314         eip = *((uint32_t *)ebp + 1);
315     }
316 }
```
注意加ebp = 0时即没有函数调用，需要加入判断

# 练习六 #
## 初始化中断描述符表IDT ##
中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，  
两者联合便是中断处理程序的入口地址。如下图  
![IDT](https://img-blog.csdn.net/20170318102822515?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzE0ODExODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  

需要用到SETGATE宏，定义如下：
```
/* *
 * Set up a normal interrupt/trap gate descriptor
 *   - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate
 *   - sel: Code segment selector for interrupt/trap handler
 *   - off: Offset in code segment for interrupt/trap handler
 *   - dpl: Descriptor Privilege Level - the privilege level required
 *          for software to invoke this interrupt/trap gate explicitly
 *          using an int instruction.
 * */
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
大致处理流程如下：  
1. CPU在执行完当前程序的每一条指令后，都会去确认在执行刚才的指令过程中中断控制器（如：8259A）是否发送中断请求过来，如果有那么CPU就会在相应的时钟脉冲到来时从总线上读取中断请求对应的中断向量；
2. CPU根据得到的中断向量（以此为索引）到IDT中找到该向量对应的中断描述符，中断描述符里保存着中断服务例程的段选择子；
3. CPU使用IDT查到的中断服务例程的段选择子从GDT中取得相应的段描述符，段描述符里保存了中断服务例程的段基址和属性信息，此时CPU就得到了中断服务例程的起始地址，并跳转到该地址；
4. CPU会根据CPL和中断服务例程的段描述符的DPL信息确认是否发生了特权级的转换。比如当前程序正运行在用户态，而中断程序是运行在内核态的，则意味着发生了特权级的转换，这时CPU会从当前程序的TSS信息（该信息在内存中的起始地址存在TR寄存器中）里取得该程序的内核栈地址，即包括内核态的ss和esp的值，并立即将系统当前使用的栈切换成新的内核栈。这个栈就是即将运行的中断服务程序要使用的栈。紧接着就将当前程序使用的用户态的ss和esp压到新的内核栈中保存起来；
5. CPU需要开始保存当前被打断的程序的现场（即一些寄存器的值），以便于将来恢复被打断的程序继续执行。这需要利用内核栈来保存相关现场信息，即依次压入当前被打断程序使用的eflags，cs，eip，errorCode（如果是有错误码的异常）信息；
6. CPU利用中断服务例程的段描述符将其第一条指令的地址加载到cs和eip寄存器中，开始执行中断服务例程。这意味着先前的程序被暂停执行，中断服务程序正式开始工作。
```
 34 /* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
 35 void
 36 idt_init(void) {
 37      /* LAB1 YOUR CODE : STEP 2 */
 38      /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
 39       *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
 40       *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
 41       *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
 42       *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which wi    ll be used later.
 43       * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
 44       *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup     each item of IDT
 45       * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'li    dt' instruction.
 46       *     You don't know the meaning of this instruction? just google it! and check the libs/x    86.h to know more.
 47       *     Notice: the argument of lidt is idt_pd. try to find it!
 48       */
 49     extern uintptr_t __vectors[];
 50     int i;
 51     for(i = 0; i < sizeof(idt) / sizeof(struct gatedesc); ++ i) {
 52         SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
 53     }
 54     lidt(&idt_pd);
 55 }
```
