---
title: 'Simple Shell'
tags: 'OSDI'
---

# Simple Shell

> https://github.com/kaiiiz/osdi2020/tree/cda1a48b90bda0ac98a9d5cd3a711ae94c14bd63

After setting up UART, you should implement a simple shell to let rpi3 interact with the host computer. The shell should be able to execute the following commands.

* help
* hello
* timestamp
* reboot

## main.c

```c
#include "shell.h"

#define CMD_LEN 128

enum shell_status {
    Read,
    Parse
};

int main() {
    shell_init();

    enum shell_status status = Read;
    while (1) {
        char cmd[CMD_LEN];
        switch (status) {
            case Read:
                shell_input(cmd);
                status = Parse;
                break;

            case Parse:
                shell_controller(cmd);
                status = Read;
                break;
        }
    }
}
```

## Notes

### CNTFRQ_EL0

This register is provided so that software can discover the frequency of the system counter. It must be programmed with this value as part of system initialization. The value of the register is not interpreted by hardware.

> https://developer.arm.com/docs/ddi0595/c/aarch64-system-registers/cntfrq_el0

### CNTPCT_EL0

Holds the 64-bit physical count value.

> https://developer.arm.com/docs/ddi0595/b/aarch64-system-registers/cntpct_el0

### GNU Inline Assembly

```
asm volatile ("mrs %0, cntfrq_el0" : "=r"(f));
```

[關於GNU Inline Assembly](http://wen00072.github.io/blog/2015/12/10/about-inline-asm/)

### PMPASSWORD, PM_RSTC, PM_WDOG

https://elinux.org/BCM2835_registers#PM