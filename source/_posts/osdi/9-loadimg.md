---
title: 'Load Image by UART'
tags: 'OSDI'
---

# Load Image by UART

> https://github.com/kaiiiz/osdi2020/tree/5d40a549b439d11bfd87bc398aa3aa9d81e5fb6e

Create 3rd bootloader

## Relocate

一開場就把自己搬家到 `0x3B3FC000`，以免執行 load image 時被蓋掉

linker.ld

```
 
SECTIONS
{
  . = 0x80000;
  .relocate :
  {
    KEEP(*(.text.relocate))
  }

  . = ALIGN(4096);
  _begin = .;
  .text :
  {
    KEEP(*(.text.boot))
    *(.text)
  }

  . = ALIGN(4096);
  .data :
  {
    *(.data)
  }

  . = ALIGN(4096);
  .bss (NOLOAD) :
  {
    __bss_start = .;

    *(.bss)

    __bss_end = .;
  }
  _end = .;
}

__bss_size = (__bss_end - __bss_start) >> 3;
__boot_loader = 0x3B3FC000;
```

start.s

```asm
.section ".text.relocate"

.global _relocate

_relocate:
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
    ldr     x1, =__boot_loader
    mov     sp, x1

    // clear bss
    ldr     x1, =__bss_start
    ldr     x2, =__bss_size
3:  cbz     x2, 4f
    str     xzr, [x1], #8
    sub     x2, x2, #1
    cbnz    x2, 3b

4:  bl      relocate


.section ".text.boot"

.global _start

_start:
    // jump to main function in C
    bl      main
    // halt this core if return
1:
    wfe
    b       1b
```

relocate.c

```c
extern unsigned char _begin, _end, __boot_loader;

__attribute__((section(".text.relocate"))) void relocate() {
    unsigned long kernel_size = (&_end - &_begin);
    unsigned char *new_bl = (unsigned char *)&__boot_loader;
    unsigned char *bl = (unsigned char *)&_begin;

    while (kernel_size--) {
        *new_bl++ = *bl;
        *bl++ = 0;
    }

    void (*start)(void) = (void *)&__boot_loader;
    start();
}
```

## Load Image

loadimg.c

```c
void loadimg() {
    long long address = address_input();

    if (address == -1) {
        return;
    }

    uart_printf("Send image via UART now!\n");

    // big endian
    int img_size = 0, i;
    for (i = 0; i < 4; i++) {
        img_size <<= 8;
        img_size |= (int)uart_read_raw();
    }

    // big endian
    int img_checksum = 0;
    for (i = 0; i < 4; i++) {
        img_checksum <<= 8;
        img_checksum |= (int)uart_read_raw();
    }

    char *kernel = (char *)address;

    for (i = 0; i < img_size; i++) {
        char b = uart_read_raw();
        *(kernel + i) = b;
        img_checksum -= (int)b;
    }

    if (img_checksum != 0) {
        uart_printf("Failed!");
    }
    else {
        void (*start_os)(void) = (void *)kernel;
        start_os();
    }
}
```

<span>sendimg.py</span>

```py
def checksum(bytecodes):
    # convert bytes to int
    return int(np.array(list(bytecodes), dtype=np.int32).sum())


def main():
    try:
        ser = serial.Serial(args.tty, 115200)
    except:
        print("Serial init failed!")
        exit(1)

    file_path = args.image
    file_size = os.stat(file_path).st_size

    with open(file_path, 'rb') as f:
        bytecodes = f.read()

    file_checksum = checksum(bytecodes)

    ser.write(file_size.to_bytes(4, byteorder="big"))
    ser.write(file_checksum.to_bytes(4, byteorder="big"))

    print(f"Image Size: {file_size}, Checksum: {file_checksum}")

    per_chunk = 128
    chunk_count = file_size // per_chunk
    chunk_count = chunk_count + 1 if file_size % per_chunk else chunk_count

    for i in range(chunk_count):
        sys.stdout.write('\r')
        sys.stdout.write("%d/%d" % (i + 1, chunk_count))
        sys.stdout.flush()
        ser.write(bytecodes[i * per_chunk: (i+1) * per_chunk])
        while not ser.writable():
            pass
```


## Notes

### x86 Boot Flow

1. Waiting power supply to settle to nominal state
2. Fetch BIOS from ROM

BIOS:

1. Power-On Self-Test
2. Load boot configuration (e.g. boot order)
3. Load the Stage 1 bootloader

Stage 1 Bootloader:

1. Read MBR on disk
2. Setup a stack
3. Load the Stage 2 bootloader

Stage 2 bootloader:

1. Read configuration file to startup a boot selection menu (e.g. GRUB)
2. Load kernel

> Ref: [x86 Initial Boot Sequence](https://www.diag.uniroma1.it/~pellegrini/didattica/2017/aosv/1.Initial-Boot-Sequence.pdf)

### GRUB

BIOS:

* 依設定順序找尋軟碟機、光碟槽、隨身碟或硬碟中是否有 boot sector
* 執行 BIOS interrupt, INT 13H，這個 interrupt 定位所有可開機裝置的 boot sectors，接著把 boot sector 載入 RAM
* 把控制權交給已經載入 RAM 的 boot sector，boot sector 也就是 1st stage boot loader (MBR 階段)

GRUB2 Stage 1 boot loader:

* **boot.img**
* 必需要符合 MBR 的大小，MBR 為 512 byte，位於磁碟中的第一個 sector，內容包含 boot loader 程式碼及 partition table
    * 前 446 bytes 為 stage 1 boot loader，包含可執行程式碼及錯誤訊文字
    * 後 64 bytes 是磁區分割表(partition table)，分別儲存著四個磁區中，每個磁區的資料，也就是每個磁區的資料為 16 bytes
    * 最後兩個 bytes 是 magic number (0xAA55)，用於驗証 MBR 的合法性
* 因為 stage 1 boot loader 很小，所以它不聰明，認不得檔案系統，因此 stage 1 boot loader 的主要任務是載入 stage 1.5 boot loader
* GRUB2 stage 1.5 通常位於磁碟的 boot record 和第一個磁區之間的位置，再把 GRUB2 stage 1.5 載入 memory 後，會把控制權交給 GRUB2 stage 1.5

GRUB2 Stage 1.5 boot loader:

* **core.img**
* 包含一些常見檔案系統的 driver，像是 EXT, FAT, NTFS，這表示 GRUB2 stage 2 可以位於標準的 EXT 檔案系統內
* 明確地說，GRUB2 stage 2 的檔案位於 /boot/grub2

> 通常來說，磁碟上的第一個磁區會位於第 63 個 sector，MBR 為第 0 個 sector，也就是說，留下了 62 個 512 bytes 的空間供 GRUB2 stage 1.5 使用

> 注意 /boot 資料夾一定要位在 GRUB2 stage 1.5 可以認得的檔案系統內，GRUB2 stage 1.5 的主要任務就是載入可以取得 GRUB2 stage 2 檔案的檔案系統 driver，及其它需要的 driver

GRUB2 Stage 2 boot loader:

* 大多是 runtime kernel modules，位於 /boot/grub2/i386-pc/ 資料夾內，被有需求時，再載入 kernel
* 最主要的任務就是載入 Linux kernel 到 RAM 中，然後把控制權交給 kernel
* kernel 和相關的檔案都位於 /boot/ 資料夾內，kernel 的名稱以 vmlinuz 為開頭，例如：/boot/vmlinuz-4.15.0-74-generic。

Kernel:

* GRUB 把 kernel 載入 RAM中
* initramfs 也會同時被載入到 RAM 中，它含有在 kernel 啟動時所必要的驅動程式
* 在 kernel 被載入到 RAM 並開始執行後，它的第一步是先自我解壓縮，等解壓縮完成之後，接著載入 Init(systemd)，然後把控制權交給 systemd

> Ref: [桌機 Linux 的開機流程 (BIOS +GRUB2)](https://silverwind1982.pixnet.net/blog/post/358594004-linux-%E7%9A%84%E9%96%8B%E6%A9%9F%E6%B5%81%E7%A8%8B-%28bios%29)

### Loading speed

> Calculate how long will it take for loading a 10MB kernel image by UART if baud rate is 115200.

$$
(\frac{1}{115200} \times 10) \times 10 \times 1024 \times 1024 = 910.22 \text{(s)}
$$