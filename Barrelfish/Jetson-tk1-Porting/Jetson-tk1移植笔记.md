# Barrelfish在Jetson TK1上的移植笔记

## Day1

### 一 armv7/offest.h注解

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

### 二 ARM-A15 Boot过程

