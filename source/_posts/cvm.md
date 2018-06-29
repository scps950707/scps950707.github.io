---
title: Compact Virtual Machine (CVM)
date: 2018-02-01 00:59:23
tags: NCTU
---
# Compact Virtual Machine (CVM)

## Introduction
The Java 2 Platform, Micro Edition (J2ME) is aimed at the market for consumer and embedded electronic devices: cellular telephones, two-way pagers, Personal Digital Assistants (PDAs), set-top boxes, and other small devices.
1. Connected Limited Device Configuration(CLDC):cellular phones, pagers, PDAs
2. Connected Device Configuration(CDC):set-top boxes, and Internet TVs

Mobile Information Device Profile (MIDP) runs on top of the CLDC which includes the KVM virtual machine. Several profiles run on top of CDC such as foundation, Personal andPersonal Basis. In this article will focus on CDC's JVM:Compact Virtual Machine(CVM).

CVM's capabilities are similar to the JVMs that support the Java 2 Platform, Standard Edition (J2SE). The main differences lie in the memory and display capabilities of the devices targeted.

* Is powered by a 32-bit processor
* Makes available to the Java runtime at least 2 MB of memory, including both RAM and flash memory or ROM
* Requires the full functionality of the Java Virtual Machine Specification
* Has connectivity to some kind of network, often wireless
* May have a user interface with some degree of sophistication

![](http://www.oracle.com/ocom/groups/public/@otn/documents/digitalasset/158002.gif)


## Porting
To order to fit my research's need, I made some modifications.
1. Ignore java class file version check
2. Compile with 32-bit mode in x86_64 system

[repo](https://github.com/scps950707/phoneme)

## Build

### x86_32

Makefile is in `cdc/build/linux-x86-generic`

On 64-bit system, only 32-bit mode is allowed, so I modified `ASM_ARCH_FLAGS`,`CC_ARCH_FLAGS`and`LINK_ARCH_FLAGS` by adding `-m32` option which already committed in repo above

Build with
```shell
make CVM_TARGET_TOOLS_PREFIX=/usr/bin/ J2ME_CLASSLIB=foundation
```

### ARM

There is an issue needed to concern when running CVM on ARM devices.

{%pdf  https://www.dropbox.com/s/p5o9yr6h9p5t8uk/0125.pdf?raw=1 %}

Build with
```shell
make CVM_TARGET_TOOLS_PREFIX=/usr/bin/arm-linux-gnueabi- J2ME_CLASSLIB=foundation USE_AAPCS=true CC_ARCH_FLAGS='-marm'
```

### Run Test
```shell
bin/cvm -cp testclasses.zip Test
```

### Reference
- [Liunx for zedboard resources build by my self](https://www.dropbox.com/s/c2i9rhs0hasytr1/zedlinux.zip?dl=0)
- [J2ME for Home Appliances and Consumer Electronic Devices](http://www.oracle.com/technetwork/systems/cdc-155908.html)
- [CDC Build System Guide](https://docs.oracle.com/javame/config/cdc/cdc-opt-impl/1.1.2/build.pdf)
- [CDC Porting Guide](https://docs.oracle.com/javame/config/cdc/cdc-opt-impl/cdc_porting_guide.pdf)
- [CDC Runtime Guide](https://docs.oracle.com/javame/config/cdc/cdc-opt-impl/1.1.2/runtime.pdf)
- [How to Port phoneME Advanced Software](http://docs.huihoo.com/openmoko/TS-6304.pdf)
- [JAVA ON HANDHELD DEVICES - COMPARING J2ME CDC TO JAVA 1.1 AND JAVA 2](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.23.297&rep=rep1&type=pdf)
- [CVM Stacks and Code Execution](http://blog.csdn.net/bazookier/article/details/4687280)
