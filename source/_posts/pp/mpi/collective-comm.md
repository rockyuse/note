---
title: 'Collective Communication - MPI'
tags: 'PP'
---

# Collective Communication

## Synchronization

### MPI_Barrier

![](2020-01-31-20-56-16.png)

```cpp
int MPI_Barrier(MPI_Comm comm)
```

## Communication

### MPI_Bcast

![](2020-01-31-20-55-41.png)

A collective communication call where a single process sends the same data contained in the message to every process in the communicator.

```cpp
int MPI_Bcast(void*            message,
              int              count,
              MPI_Datatype     datatype,
              int              root,
              MPI_Comm         comm)
```

### MPI_Scatter

![](2020-01-31-20-56-50.png)

`MPI_Scatter` splits the data referenced by the `sendbuf` on the process with rank root into p segments each of which consists of `send_count` elements of type `send_type`.

```cpp
int MPI_Scatter(void*          sendbuf,
                int            send_count,
                MPI_Datatype   send_type,
                void*          recvbuf,
                int            recv_count,
                MPI_Datatype   recv_type,
                int            root,
                MPI_Comm       comm)
```

### MPI_Gather

![](2020-01-31-20-57-10.png)

`MPI_Gather` collects the data referenced by `sendbuf` from each process in the communicator `comm`, and stores the data in process rank order on the process with rank `root` in the location referenced by `recvbuf`.

```cpp
int MPI_Gather(void*           sendbuf,
               int             send_count,
               MPI_Datatype    send_type,
               void*           recvbuf,
               int             recv_count,
               MPI_Datatype    recv_type,
               int             root,
               MPI_Comm        comm)
```

### MPI_Allgather

![](2020-01-31-20-57-33.png)

`MPI_Allgather` gathers the content from the send buffer (`sendbuf`) on each process.

```cpp
int MPI_Allgather(void*         sendbuf, 
                  int           send_count, 
                  MPI_Datatype  sendtype, 
                  void*         recvbuf, 
                  int           recvcount, 
                  MPI_Datatype  recvtype, 
                  MPI_Comm      comm)
```

## Reduction

### MPI_Reduce

![](2020-01-31-20-59-05.png)

```cpp
int MPI_Reduce(void*            operand,
               void*            result,
               int              count,
               MPI_Datatype     datatype,
               MPI_Op           operator,
               int              root,
               MPI_Comm         comm)
```

### MPI Binary Operations

| Operation Name | Meaning                      |
| -------------- | ---------------------------- |
| MPI_MAX        | Maximum                      |
| MPI_MIN        | Minimum                      |
| MPI_SUM        | Sum                          |
| MPI_PROD       | Product                      |
| MPI_LAND       | Logical And                  |
| MPI_BAND       | Bitwise And                  |
| MPI_LOR        | Logical Or                   |
| MPI_BOR        | Bitwise Or                   |
| MPI_LXOR       | Logical XOR                  |
| MPI_BXOR       | Bitwise XOR                  |
| MPI_MAXLOC     | Maximum and location of max. |
| MPI_MINLOC     | Maximum and location of min. |

### MPI_Allreduce

![](2020-01-31-21-01-39.png)

```cpp
int MPI_Allreduce(void*         sendbuf,
                  void*         recvbuf,
                  int           count,
                  MPI_Datatype  datatype,
                  MPI_Op        op,
                  MPI_Comm      comm)
```