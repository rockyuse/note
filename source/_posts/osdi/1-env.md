---
title: 'Environment'
tags: OSDI
---

# Environment

Development Environment:

* Arch Linux
* Raspberry Pi 3B+

## Hardware Spec

RAM: 1GB LPDDR2 SDRAM
CACHE: L1:32KB, L2:512KB

![GPIO](https://i.imgur.com/8KOmt8c.png)


## Toolchain

```bash
sudo pacman -S aarch64-linux-gnu-binutils aarch64-linux-gnu-gcc aarch64-linux-gnu-gdb
```

## QEMU

```bash
sudo pacman -S qemu qemu-arch-extra
```

## Code

### main.s

```ASM
.section ".text"
_start:
  wfe
  b _start
```

> wfe: Wait for event, enter to low-power standby state

### linker.ld

Linker script[^1][^2]

```
SECTIONS
{
  . = 0x80000;
  .text :
  {
      *(.text)
  }
}
```

> 把 input files 的 `.text` 放到 output files 的 `.text` section，並將 output files 的 `.text` section 放在 `0x80000` 後

### Makefile

```
TOOLCHAIN_PREFIX = aarch64-linux-gnu-
CC = $(TOOLCHAIN_PREFIX)gcc
LD = $(TOOLCHAIN_PREFIX)ld
OBJCPY = $(TOOLCHAIN_PREFIX)objcopy

CFLAGS = -Wall

.PHONY: all clean run

all: kernel8.img clean

main.o: main.s
    $(CC) $(CFLAGS) -c main.s -o main.o

kernel8.img: main.o
    $(LD) -T linker.ld -o kernel8.elf main.o
    $(OBJCPY) -O binary kernel8.elf kernel8.img

clean:
    rm -f kernel8.elf *.o

run:
    qemu-system-aarch64 -M raspi3 -kernel kernel8.img -display none -d in_asm
```

## Build Image

```bash
make
```

## Run

```bash
make run
```

## Debugging

[peda-arm](https://github.com/alset0326/peda-arm)


[^1]: [Linker Script初探 - GNU Linker Ld手冊略讀](https://wen00072.github.io/blog/2014/03/14/study-on-the-linker-script/)
[^2]: [from Source to Binary: How GNU Toolchain Works](https://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works/43)