---
title: 'Basic Initialization'
tags: 'OSDI'
---

# Basic Initialization

> https://github.com/kaiiiz/osdi2020/tree/f42786b30764454936eb26c1d235c14863b3e7d3

1. Let only one core proceed, and let others enter a busy loop.
2. Initialize the BSS segment.
3. Set the stack pointer to an appropriate position.

## start.s

```asm
.section ".text.boot"

.global _start

_start:
    // get cpu id
    mrs     x1, MPIDR_EL1
    and     x1, x1, #3
    cbz     x1, 2f
    // if cpu_id > 0, stop
1:
    wfe
    b       1b
    // if cpu_id == 0
2:
    // set stack pointer
    ldr     x1, =_start
    mov     sp, x1

    // clear bss
    ldr     x1, =__bss_start
    ldr     x2, =__bss_size
3:  cbz     x2, 4f
    str     xzr, [x1], #8
    sub     x2, x2, #1
    cbnz    x2, 3b

    // jump to main function in C
4:  bl      main
    // halt this core if return
    b       1b
```

> mrs: Move to ARM register from system coprocessor register[^1]


## linker.ld

```
SECTIONS
{
  . = 0x80000;
  .text :
  {
    KEEP(*(.text.boot))
    *(.text)
  }
  .data :
  {
    *(.data)
  }
  .bss ALIGN(16) (NOLOAD) :
  {
    __bss_start = .;

    *(.bss)

    __bss_end = .;
  }
}

__bss_size = (__bss_end - __bss_start) >> 3;
```

> KEEP: Mark sections that should not be eliminated when link-time garbage collection is in use[^2]

> NOLOAD: Mark a section to not be loaded at run time.[^3]

> ALIGN: 對齊到數字的倍數

## Some Notes

### gdb

Switch thread

```
> info threads
  Id   Target Id                    Frame 
  1    Thread 1.1 (CPU#0 [running]) 0x0000000000000000 in ?? ()
* 2    Thread 1.2 (CPU#1 [running]) 0x0000000000000300 in ?? ()
  3    Thread 1.3 (CPU#2 [running]) 0x0000000000000300 in ?? ()
  4    Thread 1.4 (CPU#3 [running]) 0x0000000000000300 in ?? ()

> thread {Id}
```

Print program counter

```
> x/3i $pc
=> 0x300:       mov     x5, #0xd8                       // #216
   0x304:       mrs     x6, mpidr_el1
   0x308:       and     x6, x6, #0x3
```

### Makefile

* [[Linux] 簡單的 Makefile 使用 (% 萬用字元、$@ 特殊符號、.PHONY 假目標)](https://ephrain.net/linux-%E7%B0%A1%E5%96%AE%E7%9A%84-makefile-%E4%BD%BF%E7%94%A8-%E8%90%AC%E7%94%A8%E5%AD%97%E5%85%83%E3%80%81-%E7%89%B9%E6%AE%8A%E7%AC%A6%E8%99%9F%E3%80%81-phony-%E5%81%87%E7%9B%AE%E6%A8%99/)
* [[Linux] Makefile中wildcard notdir patsubst使用方法](https://blog.xuite.net/auster.lai/twblog/520512252-%5BLinux%5D+Makefile%E4%B8%ADwildcard+notdir+patsubst%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95+)

### ARM ASM

#### Local labels

Ref: http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0473c/Caccjfff.html

#### AArch64 identification registers

Ref: [ARM® Cortex®-A53 MPCore Processor Technical Reference Manual](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0500d/DDI0500D_cortex_a53_r0p2_trm.pdf) - 4.2.1

> https://developer.arm.com/docs/ddi0500/j/system-control/aarch64-register-summary/aarch64-identification-registers#CIHDGHEH

#### Registers in AArch64 state

![](https://i.imgur.com/CCj3wsl.png)

* `r0 - r31` 是 32 個通用暫存器
* 使用 `x1 - x30` 會拿到 64-bits ，使用 `w0 - w30` 會拿到 Lower 32-bits
* `x31` 不直接存取
    * 使用 `xzr/wzr` 來存取即是 zero register
    * 使用 `sp/wsp` 來存取即是 stack pointer

Ref: http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0801a/BABBGCAC.html

### Linker Script

* `.gnu.linkonce.XXX`: Mostly filled with zeroes (as it is used by the LKM loader internally)
    * https://www.oreilly.com/library/view/mastering-assembly-programming/9781787287488/0cebebcf-1089-4933-ba0e-9078b154c156.xhtml
* `PROVIDE`: Define a symbol only if it is referenced and is not defined by any object included in the link
    * https://sourceware.org/binutils/docs/ld/PROVIDE.html#PROVIDE


[^1]: http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0489c/CIHGJHHH.html
[^2]: https://sourceware.org/binutils/docs/ld/Input-Section-Keep.html#Input-Section-Keep
[^3]: https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_node/ld_21.html
