---
title: 'UART'
tags: 'OSDI'
---

# UART

> https://github.com/kaiiiz/osdi2020/tree/a6e0d2fcac37ec468957b4967e85c1fe1163b3b5

rpi3 has 2 UARTs, mini UART and PL011 UART.

## MMIO

rpi3 access peripheral registers by memory mapped io (MMIO).

There is a VideoCore/ARM MMU sit between ARM CPU and peripheral bus. This MMU maps ARM’s physical address `0x3f000000` to `0x7e000000`.

> The register’s memory address in reference is bus address, you should translate into physical address.

## GPIO

rpi3 has several GPIO lines for basic input-output such as light on an LED use button as input. Besides, some of the GPIO lines provide alternate functions such as UART and SPI. Before using UART, you should **configure GPIO pin to the corresponding mode**.

![GPIO](https://i.imgur.com/8KOmt8c.png)

GPIO 14, 15 can be both used for mini UART and PL011 UART. However, mini UART should set **ALT5** and PL011 UART should set **ALT0**.

You need to configure `GPFSELn` register to change alternate function.

Next, you need to configure **pull up/down register** to disable GPIO pull up/down. It’s because these GPIO pins use alternate functions, not basic input-output. Please refer to the description of `GPPUD` and `GPPUDCLKn` registers for a detailed setup.

## Headers

### mmio.h

```c
#ifndef MMIO_H
#define MMIO_H
#define MMIO_BASE       0x3F000000
#endif
```

### gpio.h

Ref: [BCM2835 ARM Peripher](https://cs140e.sergio.bz/docs/BCM2837-ARM-Peripherals.pdf) - 6.1

```c
#include "mmio.h"

#define GPIO_BASE       (MMIO_BASE + 0x200000)

#define GPFSEL0         ((volatile unsigned int*)(GPIO_BASE + 0x00))
#define GPFSEL1         ((volatile unsigned int*)(GPIO_BASE + 0x04))
#define GPFSEL2         ((volatile unsigned int*)(GPIO_BASE + 0x08))
#define GPFSEL3         ((volatile unsigned int*)(GPIO_BASE + 0x0C))
#define GPFSEL4         ((volatile unsigned int*)(GPIO_BASE + 0x10))
#define GPFSEL5         ((volatile unsigned int*)(GPIO_BASE + 0x14))
// 0x18 Reserved
#define GPSET0          ((volatile unsigned int*)(GPIO_BASE + 0x1C))
#define GPSET1          ((volatile unsigned int*)(GPIO_BASE + 0x20))
// 0x24 Reserved
#define GPCLR0          ((volatile unsigned int*)(GPIO_BASE + 0x28))
#define GPCLR1          ((volatile unsigned int*)(GPIO_BASE + 0x2C))
// 0x30 Reserved
#define GPLEV0          ((volatile unsigned int*)(GPIO_BASE + 0x34))
#define GPLEV1          ((volatile unsigned int*)(GPIO_BASE + 0x38))
// 0x3C Reserved
#define GPEDS0          ((volatile unsigned int*)(GPIO_BASE + 0x40))
#define GPEDS1          ((volatile unsigned int*)(GPIO_BASE + 0x44))
// 0x48 Reserved
#define GPREN0          ((volatile unsigned int*)(GPIO_BASE + 0x4C))
#define GPREN1          ((volatile unsigned int*)(GPIO_BASE + 0x50))
// 0x54 Reserved
#define GPFEN0          ((volatile unsigned int*)(GPIO_BASE + 0x58))
#define GPFEN1          ((volatile unsigned int*)(GPIO_BASE + 0x5C))
// 0x60 Reserved
#define GPHEN0          ((volatile unsigned int*)(GPIO_BASE + 0x64))
#define GPHEN1          ((volatile unsigned int*)(GPIO_BASE + 0x68))
// 0x6C Reserved
#define GPLEN0          ((volatile unsigned int*)(GPIO_BASE + 0x70))
#define GPLEN1          ((volatile unsigned int*)(GPIO_BASE + 0x74))
// 0x78 Reserved
#define GPAREN0         ((volatile unsigned int*)(GPIO_BASE + 0x7C))
#define GPAREN1         ((volatile unsigned int*)(GPIO_BASE + 0x80))
// 0x84 Reserved
#define GPAFEN0         ((volatile unsigned int*)(GPIO_BASE + 0x88))
#define GPAFEN1         ((volatile unsigned int*)(GPIO_BASE + 0x8C))
// 0x90 Reserved
#define GPPUD           ((volatile unsigned int*)(GPIO_BASE + 0x94))
#define GPPUDCLK0       ((volatile unsigned int*)(GPIO_BASE + 0x98))
#define GPPUDCLK1       ((volatile unsigned int*)(GPIO_BASE + 0x9C))
```

### aux.h

Ref: [BCM2835 ARM Peripher](https://cs140e.sergio.bz/docs/BCM2837-ARM-Peripherals.pdf) - 2.1

```c
#include "mmio.h"

#define AUX_BASE        (MMIO_BASE + 0x215000)

#define AUX_IRQ         ((volatile unsigned int*)(AUX_BASE + 0x00))
#define AUX_ENABLES     ((volatile unsigned int*)(AUX_BASE + 0x04))
#define AUX_MU_IO       ((volatile unsigned int*)(AUX_BASE + 0x40))
#define AUX_MU_IER      ((volatile unsigned int*)(AUX_BASE + 0x44))
#define AUX_MU_IIR      ((volatile unsigned int*)(AUX_BASE + 0x48))
#define AUX_MU_LCR      ((volatile unsigned int*)(AUX_BASE + 0x4C))
#define AUX_MU_MCR      ((volatile unsigned int*)(AUX_BASE + 0x50))
#define AUX_MU_LSR      ((volatile unsigned int*)(AUX_BASE + 0x54))
#define AUX_MU_MSR      ((volatile unsigned int*)(AUX_BASE + 0x58))
#define AUX_MU_SCRATCH  ((volatile unsigned int*)(AUX_BASE + 0x5C))
#define AUX_MU_CNTL     ((volatile unsigned int*)(AUX_BASE + 0x60))
#define AUX_MU_STAT     ((volatile unsigned int*)(AUX_BASE + 0x64))
#define AUX_MU_BAUD     ((volatile unsigned int*)(AUX_BASE + 0x68))
#define AUX_SPI0_CNTL0  ((volatile unsigned int*)(AUX_BASE + 0x80))
#define AUX_SPI0_CNTL1  ((volatile unsigned int*)(AUX_BASE + 0x84))
#define AUX_SPI0_STAT   ((volatile unsigned int*)(AUX_BASE + 0x88))
#define AUX_SPI0_IO     ((volatile unsigned int*)(AUX_BASE + 0x90))
#define AUX_SPI0_PEEK   ((volatile unsigned int*)(AUX_BASE + 0x94))
#define AUX_SPI1_CNTL0  ((volatile unsigned int*)(AUX_BASE + 0xC0))
#define AUX_SPI1_CNTL1  ((volatile unsigned int*)(AUX_BASE + 0xC4))
#define AUX_SPI1_STAT   ((volatile unsigned int*)(AUX_BASE + 0xC8))
#define AUX_SPI1_IO     ((volatile unsigned int*)(AUX_BASE + 0xD0))
#define AUX_SPI1_PEEK   ((volatile unsigned int*)(AUX_BASE + 0xD4))
```

