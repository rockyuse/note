---
title: 'PL011 UART'
tags: 'OSDI'
---

# PL011 UART

> https://github.com/kaiiiz/osdi2020/tree/c9818d3900a40139116e75d353a5a883a1c5f095

## UART Init

1. 調整 UART0 Clk (clock-id: https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface#clocks)
2. Map UART 到 GPIO Pin
3. 設定 Baud rate

> Caveat: GPIO 14, 15: PL011 UART should set ALT0

```c
void uart_init() {
    *UART0_CR = 0;  // turn off UART0

    /* Configure UART0 Clock Frequency */
    unsigned int __attribute__((aligned(16))) mbox[9];
    mbox[0] = 9 * 4;
    mbox[1] = MBOX_CODE_BUF_REQ;
    // tags begin
    mbox[2] = MBOX_TAG_SET_CLOCK_RATE;
    mbox[3] = 12;
    mbox[4] = MBOX_CODE_TAG_REQ;
    mbox[5] = 2;        // UART clock
    mbox[6] = 4000000;  // 4MHz
    mbox[7] = 0;        // clear turbo
    mbox[8] = 0x0;      // end tag
    // tags end
    mbox_call(mbox, 8);

    /* Map UART to GPIO Pins */
    // 1. Change GPIO 14, 15 to alternate function
    register unsigned int r = *GPFSEL1;
    r &= ~((7 << 12) | (7 << 15));  // Reset GPIO 14, 15
    r |= (4 << 12) | (4 << 15);     // Set ALT0
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

    /* Configure UART0 */
    *UART0_IBRD = 0x2;        // Set 115200 Baud
    *UART0_FBRD = 0xB;        // Set 115200 Baud
    *UART0_LCRH = 0b11 << 5;  // Set word length to 8-bits
    *UART0_ICR = 0x7FF;       // Clear Interrupts

    /* Enable UART */
    *UART0_CR = 0x301;
}
```

## UART Read

```c
char uart_read() {
    // Check data ready field
    do {
        asm volatile("nop");
    } while (*UART0_FR & 0x10);
    // Read
    char r = (char)(*UART0_DR);
    // Convert carrige return to newline
    return r == '\r' ? '\n' : r;
}
```

## UART Write

```c
void uart_write(unsigned int c) {
    // Check transmitter idle field
    do {
        asm volatile("nop");
    } while (*UART0_FR & 0x20);
    // Write
    *UART0_DR = c;
}
```

## Notes

### UARTIBRD

The UARTIBRD register is the integer part of the baud rate divisor value. All the bits
are cleared to 0 on reset.

> Baud rate divisor BAUDDIV = (FUARTCLK/ {16 * Baud rate})

> Ref: [PrimeCell® UART (PL011)](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0183f/DDI0183.pdf) - 3.3.5

### UARTFBRD

The UARTFBRD register is the fractional part of the baud rate divisor value. All the bits
are cleared to 0 on reset.

> Ref: [PrimeCell® UART (PL011)](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0183f/DDI0183.pdf) - 3.3.6

### Typical baud rates and integer and fractional divisors

![](https://i.imgur.com/UuKSpcu.png)

### UARTLCR_H

The UARTLCR_H register is the line control register. This register accesses bits 29 to
22 of the UART bit rate and line control register, UARTLCR. All the bits are cleared to 0 when reset.

> Ref: [PrimeCell® UART (PL011)](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0183f/DDI0183.pdf) - 3.3.7

### UARTICR

The UARTICR register is the interrupt clear register and is write-only. On a write of 1,
the corresponding interrupt is cleared. A write of 0 has no effect.

> Ref: [PrimeCell® UART (PL011)](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0183f/DDI0183.pdf) - 3.3.13

### UARTFR

The UARTFR register is the flag register. After reset TXFF, RXFF, and BUSY are 0,
and TXFE and RXFE are 1.

> Ref: [PrimeCell® UART (PL011)](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0183f/DDI0183.pdf) - 3.3.3

### UARTDR

The UARTDR register is the data register.

> Ref: [PrimeCell® UART (PL011)](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0183f/DDI0183.pdf) - 3.3.1


