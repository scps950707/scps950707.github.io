---
title: OSDI-Booting-process
date: 2018-03-02 17:29:48
tags:
- OSDI
- NCTU
---
# Booting process
###### tags: `OSDI` `NCTU`
## Bootloader OverView

### First loader
- Save in permanent memory such as ROM/NOR flash
CPU會指定固定的位址到ROM取得取得data(開機要做的instruction)
- Check/initialize/test the hardware
- May provide basic services to other programs
- May provide shells to end‐users for basic operations
- Load/jump to the second loader (based on predefined procedures/configurations)
- Ex:u-boot for ARM, BIOS for x86
### Second loader
- As soon as first bootloader can access the second loader (on any device such as CD-ROM or hard disk), load it into memory, and can jump to the starting instruction
- Loading OS into memory
- Prepared by OS vendors or other third party vendors such as GRUB LILO
### Third loader
- A program executes without OS services
- Initial and prepare OS services 像是vector table等等
### Fourth loader
- OS is now ready and can provide services
- Any process calls OS services to fork/execute other processes

## Booting process overview

### Power On

不是這門課關注的範圍詳情見[^1][^2]


### Booting from permanent memory

#### ARM Example

- Hardware Example:
FPU fire address到system bus 到Flash取得bootloader
![](https://i.imgur.com/QaI54tE.png =550x350)

- Procedure
![](https://i.imgur.com/w35gYgJ.png =520x420)
First Stage boot loader:設定/檢查硬體並且將CPU主導權交給second stage
Second Stage boot loader:提供shell做一些設定,BIOS通常包含在此

- First stage example

[^7][^8][^9]

```cpp=
.globl _start
/* Vector table 通常用來處理interrupt */
_start:
    b reset
    b undefined_instruction
    b software_interrupt
    b prefetch_abort
    b data_abort
    b not_used
    b irq
    b fiq
    
...

/* the actual reset code */
reset:
    /* First, mask **ALL** interrupts */
    ldr r0, IC_BASE
    mov r1, #0x00
    str r1, [r0, #ICMR]
    
    /* switch CPU to correct speed */
    ldr r0, PWR_BASE
    LDR r1, cpuspeed
    str r1, [r0, #PPCR]

    /* setup memory */
    bl memsetup
    /* init LED */
    bl ledinit
    
    /* check if this is a wake-up from sleep */
    
    ldr r0, RST_BASE // Reset status register
    ldr r1, [r0, #RCSR]
    and r1, r1, #0x0f
    teq r1, #0x08
    bne normal_boot /* no, continue booting */
    
    /* yes, a wake-up. clear RCSR by writing a 1 (see 9.6.2.1 from [1]) */
    mov r1, #0x08
    str r1, [r0, #RCSR]

    /* get value from the PSPR and jump to it */
    ldr r0, PWR_BASE
    ldr r1, [r0, #PCSR] // Power manager scratch pad register
    mov pc, r1

normal_boot:
    /* enable I-cache */
    mrc p15, 0, r1, c1, c0, 0     @ read control reg
    orr r1, r1, #0x1000           @ set Icache
    mcr p15, 0, r1, c1, c0, 0     @ write it back
    
    /* check the first 1MB in increments of 4k */
    mov r7, #0x1000
    mov r6, r7, lsl #8
    ldr r5, MEM_START
    /*清空1MB的RAM把second stage要用到的boot loader load到此處的RAM
     因為需要較多的mem Read/Write,在ROM下效率不佳*/
    
mem_test_loop:
    mov r0, r5
    bl testram
    teq r0, #1
    beq badram
    
    add r5, r5, r7
    subs r6, r6, r7
    bne mem_test_loop
    
    /* the first megabyte is OK, so let's clear it */
    mov r0, #((1024 * 1024) / (8 * 4)) // 1MB in steps of 32 bytes
    ldr r1, MEM_START
...
clear_loop:
    stmia r1!, {r2-r9}
    subs r0, r0, #(8 * 4)
    bne clear_loop
    
    /* relocate the second stage loader */
    add r2, r0, #(128 * 1024) // blob is 128kB
    add r0, r0, #0x400 // skip first 1024 bytes
    ldr r1, MEM_START
    add r1, r1, #0x400 // skip over here as well
...
copy_loop:
    ldmia r0!, {r3-r10}
    stmia r1!, {r3-r10}
    cmp r0, r2
    ble copy_loop
    /* set up the stack pointer */
    ldr r0, MEM_START
    add r1, r0, #(1024*1024)
    sub sp, r1, #0x04
    
    /* blob is copied to ram, so jump to it */
    add r0, r0, #0x400
    mov pc, r0
```

Second stage:
```cpp=
int main(void)
{
~
	led_on();
~
	SerialInit(baud9k6);
	TimerInit();
~
	SerialOutputString(PACKAGE " version " VERSION
	"Copyright (C) 1999 2000 2001 "
~
	get_memory_map();
~
	SerialOutputString("Running from ");
	if(RunningFromInternal())
		SerialOutputString("internal");
	else
		SerialOutputString("external");
...
/* wait 10 seconds before starting autoboot */
SerialOutputString("Autoboot in progress, press any key...");
	for(i = 0; i < 10; i++) {
	SerialOutputByte('.');
	retval = SerialInputBlock(commandline, 1, 1);
~
	if(retval == 0) {
~
	boot_linux(commandline);
	}
~
	for(;;) {
~
	if(numRead > 0) {
	if(MyStrNCmp(commandline, "boot", 4) == 0) {
		boot_linux(commandline + 4);
	} else if(MyStrNCmp(commandline, "clock", 5) == 0) {
		SetClock(commandline + 5);
	} else if(MyStrNCmp(commandline, "download ", 9) == 0) {
~ return 0;
} /* main */
```

#### x86 Example

- Hardware
![](https://upload.wikimedia.org/wikipedia/commons/thumb/b/bd/Motherboard_diagram.svg/450px-Motherboard_diagram.svg.png =300x462)
- First stage
    1. CPU一開機就會去抓FFFFFFF0h(ROM addr)的BIOS
    1. 將384KB的BIOS code從ROM搬到RAM並從0xFFFF0開始執行
    1. Power supply sends POWER GOOD to CPU
    1. CPU resets
    1. Run System BIOS code at 0x0000~0xFFFF with respect to ROM address(But BIOS is already in RAM now)
    1. Jump to a real BIOS start address(通常不會隔太遠)
    1. POST(power on self test)，其中會檢查例如:是不是重開機，判斷的flag放在CMOS裡
    1. Beep if there is an error
    1. Read CMOS data/settings
    1. 發出INT 19h去找開機硬碟，這檢查動作包含重硬碟load 1 sector(512bytes)到0x7c00並檢查最後兩個byte是不是0xAA55
    1. 主控權交給0x7c00這邊會準備load MBR

[^10][^11][^12]

![](https://i.imgur.com/Mdiik0m.png =430x367)

- Second stage
    1. 呼叫interupt 13 service啟動"hard disk drive service"
    1. 到硬碟指定位置讀取MBR(Master Boot Record)
![](https://i.imgur.com/RBgbHaR.png)

![](https://i.imgur.com/5n9XKFd.png)


### Booting from any devices

- 透過MBR再將GRUB或是LILO等OS loader載入memory並執行

![](https://i.imgur.com/f9Sw8YS.png =300x300)

- KernelImage剖析
![](https://upload.wikimedia.org/wikipedia/commons/9/97/Anatomy-of-bzimage.png)

通常kernel image會再經過壓縮成為一個自解檔，此目的在於時間與空間的取捨。通常在嵌入式系統CPU較慢所以載入OS時偏好原始的kernel image

### Loading OS

- Procedure
![](https://i.imgur.com/mmsbEpG.png)

Linux在開機的時候為了速度會使用Ramdisk將一部份driver載入並且存取,做完完整設定後再重新Remount真正的root filesystem[^3][^4][^5][^6]

- Example
![](https://i.imgur.com/L0v2cF7.png)
step 6 從floppy disk load MBR到0x7C00並複製一塊到0x90000並在那邊執行

- Mount root disk
![](https://i.imgur.com/qL11EfN.jpg)


- Kernel Image Structure
![](https://i.imgur.com/VYwFN42.png)


- Kernel init procedure
![](https://i.imgur.com/JMLPxdq.png)

### Loading process

![](https://i.imgur.com/eB1XyYz.png)

### Booting speedup
* Remove waiting time
* Removing unnecessary initialization routines
* Uncompressed kernel
* DMA Copy Of Kernel On Startup
* Fast Kernel Decompression
* Kernel XIP(execution in place kernel run in ROM)


[^1]:[How computer power supplies work – KitGuru Guide](https://www.kitguru.net/components/power-supplies/ironlaw/how-computer-power-supplies-work-kitguru-guide/)
[^2]:[Everything You Need to Know About The Motherboard Voltage Regulator Circuit](http://www.hardwaresecrets.com/everything-you-need-to-know-about-the-motherboard-voltage-regulator-circuit/)
[^3]:[Jserv's blog: 探索 Linux bootloader 的佳作](http://blog.linux.org.tw/~jserv/archives/001840.html)
[^4]:[Jserv's blog: 深入理解 Linux 2.6 的 initramfs 機制](http://blog.linux.org.tw/~jserv/archives/001954.html)
[^5]:[嵌入式系统 Boot Loader 技术内幕](https://www.ibm.com/developerworks/cn/linux/l-btloader/)
[^6]:[initrd和initramfs的區別](https://read01.com/zh-tw/QzJka.html)
[^7]:[source code](https://github.com/xcvista/vivi/blob/master/arch/s3c2410/head.S)
[^8]:[S3C2410 vivi閱讀筆記](http://blog.csdn.net/myspor/article/details/6316291)
[^9]:[linux kernel driver for s3c2410](https://github.com/torvalds/linux/blob/master/drivers/mtd/nand/s3c2410.c)
[^10]:[BIOS](https://en.wikipedia.org/wiki/BIOS)
[^11]:[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
[^12]:[INT 19H Bootstrap Loader](http://webpages.charter.net/danrollins/techhelp/0243.HTM)
