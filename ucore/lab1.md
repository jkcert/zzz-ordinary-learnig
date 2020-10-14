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
