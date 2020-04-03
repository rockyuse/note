---
title: 'Mailbox'
tags: 'OSDI'
---

# Mailbox

Mailboxes facilitate communication between the ARM and the VideoCore

## Introduction

> https://github.com/raspberrypi/firmware/wiki/Mailboxes

* Each mailbox is an 8-deep FIFO of 32-bit words, which can be read (popped)/written (pushed) by the ARM and VC.
* Only mailbox 0's status can trigger interrupts on the ARM, so MB 0 is always for communication **from VC to ARM** and MB 1 is **for ARM to VC**.
* The ARM should never write MB 0 or read MB 1.

### Channel 8 (CPU->GPU)

> https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface

Buffer contents:

* u32: buffer size in bytes (including the header values, the end tag and padding)
* u32: buffer request/response code
    * Request codes:
        * 0x00000000: process request
        * All other values reserved
    * Response codes:
        * 0x80000000: request successful
        * 0x80000001: error parsing request buffer (partial response)
        * All other values reserved
* u8...: sequence of concatenated tags
* u32: 0x0 (end tag)
* u8...: padding

Tag format:

* u32: tag identifier
* u32: value buffer size in bytes
* u32:
    * Request codes:
        * b31 clear: request
        * b30-b0: reserved
    * Response codes:
        * b31 set: response
        * b30-b0: value length in bytes
* u8...: value buffer

## Implementation

> https://github.com/kaiiiz/osdi2020/tree/37da2a722fd7df2abd25ae07106aa3bc6caa4f50

```c
int mbox_call(unsigned int* mbox, unsigned char channel) {
    unsigned int r = (unsigned int)(((unsigned long)mbox) & (~0xF)) | (channel & 0xF);
    // wait until full flag unset
    while (*MBOX_STATUS & MBOX_FULL) {
    }
    // write address of message + channel to mailbox
    *MBOX_WRITE = r;
    // wait until response
    while (1) {
        // wait until empty flag unset
        while (*MBOX_STATUS & MBOX_EMPTY) {
        }
        // is it a response to our msg?
        if (r == *MBOX_READ) {
            // check is response success
            return mbox[1] == MBOX_CODE_BUF_RES_SUCC;
        }
    }
    return 0;
}
```

### Example

> 注意 aligned(16)，因為 mbox_call 在定址時會需要在後方放上 channel number

Get Board Revision:

```c
void mbox_board_revision() {
    unsigned int __attribute__((aligned(16))) mbox[7];
    mbox[0] = 7 * 4;  // buffer size in bytes
    mbox[1] = MBOX_CODE_BUF_REQ;
    // tags begin
    mbox[2] = MBOX_TAG_GET_BOARD_REVISION;  // tag identifier
    mbox[3] = 4;                            // maximum of request and response value buffer's length.
    mbox[4] = MBOX_CODE_TAG_REQ;            // tag code
    mbox[5] = 0;                            // value buffer
    mbox[6] = 0x0;                          // end tag
    // tags end
    mbox_call(mbox, 8);
    uart_printf("Board Revision: %x\n", mbox[5]);
}
```

Get VC Memory:

```c
void mbox_vc_memory() {
    unsigned int __attribute__((aligned(16))) mbox[8];
    mbox[0] = 8 * 4;  // buffer size in bytes
    mbox[1] = MBOX_CODE_BUF_REQ;
    // tags begin
    mbox[2] = MBOX_TAG_GET_VC_MEMORY;  // tag identifier
    mbox[3] = 8;                       // maximum of request and response value buffer's length.
    mbox[4] = MBOX_CODE_TAG_REQ;       // tag code
    mbox[5] = 0;                       // base address
    mbox[6] = 0;                       // size in bytes
    mbox[7] = 0x0;                     // end tag
    // tags end
    mbox_call(mbox, 8);
    uart_printf("VC Core base addr: 0x%x size: 0x%x\n", mbox[5], mbox[6]);
}
```

## Notes

### Mailbox hardware register addresses

```c
#define MMIO_BASE 0x3F000000
#define MBOX_Base (MMIO_BASE + 0xB880)
```

| REGISTER      | ADDRESS            |
| ------------- | ------------------ |
| Read          | `MBOX_Base`        |
| Poll          | `MBOX_Base` + 0x10 |
| Sender        | `MBOX_Base` + 0x14 |
| Status        | `MBOX_Base` + 0x18 |
| Configuration | `MBOX_Base` + 0x1C |
| Write         | `MBOX_Base` + 0x20 |

> Ref: http://magicsmoke.co.za/?p=284