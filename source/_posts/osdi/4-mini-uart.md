---
title: 'Mini UART'
tags: 'OSDI'
---

# Mini UART

> https://github.com/kaiiiz/osdi2020/tree/bfc4ff176fd5051cff465d7bb22e30f3487a60e5

Init:

1. Set `AUXENB` register to enable mini UART. Then mini UART register can be accessed.
2. Set `AUX_MU_CNTL_REG` to 0. Disable transmitter and receiver during configuration.
3. Set `AUX_MU_IER_REG` to 0. Disable interrupt because currently you don’t need interrupt.
4. Set `AUX_MU_LCR_REG` to 3. Set the data size to 8 bit.
5. Set `AUX_MU_MCR_REG` to 0. Don’t need auto flow control.
6. Set `AUX_MU_BAUD` to 270. Set baud rate to 115200Set 
7. AUX_MU_IIR_REG to 6. No FIFO.
8. Set `AUX_MU_CNTL_REG` to 3. Enable the transmitter and receiver.

Read:

1. Check `AUX_MU_LSR_REG`’s data ready field.
2. If set, read from `AUX_MU_IO_REG`

Write:

1. Check `AUX_MU_LSR_REG`’s Transmitter empty field.
2. If set, write to `AUX_MU_IO_REG`

## uart.c

```c
#include "aux.h"
#include "gpio.h"

void uart_init() {
    /* Initialize UART */
    *AUX_ENABLES |= 1;   // Enable mini UART
    *AUX_MU_CNTL = 0;    // Disable TX, RX during configuration
    *AUX_MU_IER = 0;     // Disable interrupt
    *AUX_MU_LCR = 3;     // Set the data size to 8 bit
    *AUX_MU_MCR = 0;     // Don't need auto flow control
    *AUX_MU_BAUD = 270;  // Set baud rate to 115200
    *AUX_MU_IIR = 6;     // No FIFO

    /* Map UART to GPIO Pins */

    // 1. Change GPIO 14, 15 to alternate function
    register unsigned int r = *GPFSEL1;
    r &= ~((7 << 12) | (7 << 15));  // Reset GPIO 14, 15
    r |= (2 << 12) | (2 << 15);     // Set ALT5
    *GPFSEL1 = r;

    // 2. Disable GPIO pull up/down (Because these GPIO pins use alternate functions, not basic input-output)
    // Set control signal to disable
    *GPPUD = 0;
    // Wait 150 cycles
    r = 150;
    while (r--) {
        asm volatile("nop");
    }
    // Clock the control signal into the GPIO pads
    *GPPUDCLK0 = (1 << 14) | (1 << 15);
    // Wait 150 cycles
    r = 150;
    while (r--) {
        asm volatile("nop");
    }
    // Remove the clock
    *GPPUDCLK0 = 0;

    // 3. Enable TX, RX
    *AUX_MU_CNTL = 3;
}

char uart_read() {
    // Check data ready field
    do {
        asm volatile("nop");
    } while (!(*AUX_MU_LSR & 0x01));
    // Read
    char r = (char)(*AUX_MU_IO);
    // Convert carrige return to newline
    return r == '\r' ? '\n' : r;
}

void uart_write(unsigned int c) {
    // Check transmitter idle field
    do {
        asm volatile("nop");
    } while (!(*AUX_MU_LSR & 0x20));
    // Write
    *AUX_MU_IO = c;
}
```

## main.c

```c
#include "uart.h"

int main() {
    uart_init();

    while (1) {
        uart_write(uart_read());
    }
}
```

## Some Notes

### GPIO Documents

#### GPFSELn

> GPIO Function Select Registers

The function select registers are used to define the operation of the general-purpose I/O 
pins.

![](https://i.imgur.com/Mslcbv0.png)

#### GPPUD

> GPIO Pull-up/down Register

The GPIO Pull-up/down Register controls the actuation of the internal pull-up/down control line to ALL the GPIO pins. This register must be used in conjunction with the 2 `GPPUDCLKn` registers.

![](https://i.imgur.com/sEHVnUi.png)

#### GPPUDCLKn

> GPIO Pull-up/down Clock Register

The GPIO Pull-up/down Clock Registers control the actuation of internal pull-downs on the respective GPIO pins.

These registers must be used in conjunction with the `GPPUD` register to effect GPIO Pull-up/down changes. The following sequence of events is required: 

1. Write to `GPPUD` to set the required control signal (i.e. Pull-up or Pull-Down or neither to remove the current Pull-up/down)
2. Wait 150 cycles – this provides the required set-up time for the control signal
3. Write to `GPPUDCLK0/1` to clock the control signal into the GPIO pads you wish to modify – NOTE only the pads which receive a clock will be modified, all others will retain their previous state.
4. Wait 150 cycles – this provides the required hold time for the control signal
5. Write to `GPPUD` to remove the control signal
6. Write to `GPPUDCLK0/1` to remove the clock

![](https://i.imgur.com/KpM9XGr.png)

### AUX Documents

#### AUX_MU_CNTL

The `AUX_MU_CNTL_REG` provides access to some extra useful and nice features not found on a normal 16550 UART

![](https://i.imgur.com/7grZUS1.png)

![](https://i.imgur.com/CdLJnTE.png)

#### AUX_MU_IER

The `AUX_MU_IIR_REG` register shows the interrupt status. It also has two FIFO enable status bits and (when writing) FIFO clear bits. 

![](https://i.imgur.com/jvNQjht.png)

#### AUX_MU_LCR

The `AUX_MU_LCR_REG` register controls the line data format and gives access to the baudrate register

![](https://i.imgur.com/Uv8VN7I.png)

#### AUX_MU_MCR

The `AUX_MU_MCR_REG` register controls the 'modem' signals

![](https://i.imgur.com/PRhfreT.png)

#### AUX_MU_BAUD

The `AUX_MU_BAUD` register allows direct access to the 16-bit wide baudrate counter

![](https://i.imgur.com/4JMb3MJ.png)

$$
\text{baud rate} = \frac{\text{system clock freq}}{8 \times \text{AUX\_MU\_BAUD} + 1}
$$

#### AUX_MU_IIR

The `AUX_MU_IER_REG` register is primary used to enable interrupts 
If the DLAB bit in the line control register is set this register gives access to the MS 8 bits of the baud rate

![](https://i.imgur.com/vqVnBMx.png)

#### AUX_MU_LSR

The `AUX_MU_LSR_REG` register shows the data status

![](https://i.imgur.com/Sxrxvr6.png)

#### AUX_MU_IO

The `AUX_MU_IO_REG` register is primary used to write data to and read data from the UART FIFOs

If the DLAB bit in the line control register is set this register gives access to the LS 8 bits of the baud rate

![](https://i.imgur.com/DfSTI6b.png)

## Reference

[BCM2835 ARM Peripher](https://cs140e.sergio.bz/docs/BCM2837-ARM-Peripherals.pdf)