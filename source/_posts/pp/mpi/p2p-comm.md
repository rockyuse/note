---
title: 'Point-to-Point Communication - MPI'
tags: 'PP'
---

# Point-to-Point Communication

The message envelope consists of the following information:

* The rank of the receiver
* The rank of the sender
* A tag
  * Distinguish messages from a single process
* A communicator

## MPI_Send

Blocking send

```cpp
int MPI_Send(void*          message,
             int            count,
             MPI_Datatype   datatype,
             int            dest,
             int            tag,
             MPIComm        comm)
```

## MPI_Recv

Blocking receive

```cpp
int MPI_Recv(void*          message,
             int            count,
             MPI_Datatype   datatype,
             int            source,
             int            tag,
             MPI_Comm       comm,
             MPI_Status*    status)
```

## MPI Data Types

https://www.mpi-forum.org/docs/mpi-2.1/mpi21-report-bw/node330.htm

## MPI_Status object

```cpp
MPI_Status status;
```

## Example

```cpp
#include <stdio.h>
#include <string.h>
#include "mpi.h"

int main(int argc, char* argv[]) {
    int my_rank;       /* rank of process */
    int p;             /* number of processes */
    int source;        /* rank of sender */
    int dest;          /* rank of receiver */
    int tag = 0;       /* tag for messages */
    char message[100]; /* storage for message */
    MPI_Status status; /* return status for receive */

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &p);
    if (my_rank != 0) {
        sprintf(message, "Greetings from process %d!", my_rank);
        dest = 0;
        /* Use strlen+1 so that \0 gets transmitted */
        MPI_Send(message, strlen(message) + 1, MPI_CHAR, dest, tag, MPI_COMM_WORLD);
    } else {
        for (source = 1; source < p; source++) {
            MPI_Recv(message, 100, MPI_CHAR, source, tag, MPI_COMM_WORLD, &status);
            printf("%s\n", message);
        }
        printf("Greetings from process %d!\n", my_rank);
    }
    MPI_Finalize();
}
```

## Communication Mode

* Buffered
* Ready
* Standard
* Synchronous

Each of these communication modes has both blocking and non-blocking primitives:

* blocking
  * The send call blocks until the send block can be reclaimed
  * The receive function blocks until the buffer has successfully obtained the contents of the message
* non-blocking
  * Allow the possible overlap of communication with computation
  * Communication is usually done in 2 phases
    * the posting phase
    * the test for completion phase

### Basic concepts

![](2020-01-31-18-03-51.png)

### Blocking

**Synchronization Overhead**
the time spent waiting for an event to occur on another task

**System Overhead**
the time spent when copying the message data from sender’s message buffer to network and from network to the receiver’s message buffer.

#### Synchronous Send

![](2020-01-31-18-11-28.png)

1. `MPI_Ssend()`: "ready to send" message is sent from the sending task to receiving task.
2. `MPI_Recv()`: "ready to receive" message is sent, followed by the transfer of data

Synchronization Overhead:

* Sender 要等待 Receiver 回覆 handshake 後才開始傳輸
* Receiver 要等待 Sender 回覆 handshake 後才開始接收

System Overhead:

* copying from sender & receiver buffers to the network.

#### Ready Send

![](2020-01-31-18-20-15.png)

* `MPI_Rsend()`: Sender 收到 "ready to receive" 再將 user-supplied buffer 送往 Receiver
* If "ready to receive" message hasn't arrived, the ready mode send will incur an error and exit.

#### Buffered Send

![](2020-01-31-18-28-31.png)

1. `MPI_Bsend()`  先將數據複製到 user-supplied buffer，複製完馬上 return
2. Sender 收到 "ready to receive" 再將 user-supplied buffer 送往 Receiver

Synchronization overhead:

* Sender 不用等待 Receiver 回應
* receiving process can still be incurred, because if the receive is executed before the send, the process must wait before it can return to the execution sequence

System Overhead:

* Replicated copies of the buffer

#### Standard Send

![](2020-01-31-19-53-13.png)

When the data size is smaller than a threshold value:

* `MPI_Send()` Sender 將數據複製到 Receiver 的 sysbuf
* `MPI_Recv()` Receiver 將數據從 sysbuf 複製出來

The decreased synchronization overhead is usually at the cost of increased system overhead due to the extra copy of buffers

#### Buffered Standard Send

When the message size is greater than a threshold

![](2020-01-31-19-59-50.png)

* The behavior is same as for the synchronous mode
* Small messages benefit from the decreased chance of synchronization overhead
* Large messages results in increased cost of copying to the buffer and system overhead

### Non-blocking

![](2020-01-31-20-04-25.png)

Sender:

* `MPI_Isend()` posts a non-blocking **standard send** when the message buffer contents are ready to be transmitted
* The control returns immediately without waiting for the copy to the remote system buffer to complete
* `MPI_Wait` is called just before the sending task needs to overwrite the message buffer
* Programmer is responsible for checking the status of the message to know whether data to be sent has been copied out of the send buffer

Receiver:

* `MPI_Irecv()` issues a non-blocking receive as soon as a message buffer is ready to hold the message
* The non-blocking receive returns without waiting for the message to arrive
* The receiving task calls `MPI_Wait` when it needs to use the incoming message data

test without waiting using MPI_TEST:

* `MPI_TEST(request, flag, status)`

## Deadlock

A situation where the dependencies between processors are cyclic

> MPI does not have timeouts

```cpp
if (rank == 0) {
    err = MPI_Send(sendbuf, count, datatype, 1, tag, comm);
    err = MPI_Recv(recvbuf, count, datatype, 1, tag, comm, &status);
}
else {
    err = MPI_Send(sendbuf, count, datatype, 0, tag, comm);
    err = MPI_Recv(recvbuf, count, datatype, 0, tag, comm, &status);
}
```

* If the message sizes are small enough, this should work because of systems buffers
* If the messages are too large, or system buffering is not used, this will hang

Solution:

```cpp
if (rank == 0) {
    err = MPI_Send(sendbuf, count, datatype, 1, tag, comm);
    err = MPI_Recv(recvbuf, count, datatype, 1, tag, comm, &status);
}
else {
    err = MPI_Recv(recvbuf, count, datatype, 0, tag, comm, &status);
    err = MPI_Send(sendbuf, count, datatype, 0, tag, comm);
}
```

```cpp
if (rank == 0) {
    err = MPI_Isend(sendbuf, count, datatype, 1, tag, comm, &req);
    err = MPI_Irecv(recvbuf, count, datatype, 1, tag, comm);
    err = MPI_Wait(req, &status);
}
else {
    err = MPI_Isend(sendbuf, count, datatype, 0, tag, comm, &req);
    err = MPI_Irecv(recvbuf, count, datatype, 0, tag, comm);
    err = MPI_Wait(req, &status);
}
```