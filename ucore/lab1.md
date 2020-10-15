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

# 练习5 #
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
