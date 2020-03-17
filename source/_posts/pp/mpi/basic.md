---
title: 'MPI'
tags: 'PP'
---

# MPI

**Multithreading vs Message Passing**

We should just abandon threading? [^1]

![](2020-01-31-16-51-24.png)

**Shared Memory vs Distribured Memory**

![Shared Memory System](2020-01-31-16-54-32.png)

![Distribured Memory System](2020-01-31-16-56-57.png)

## Introduction

http://www.mpi-forum.org/

* The message passing interface (MPI) is a standard library
* Hardware-portable, multi-language communication library
* MPI implementations
  *  MPICH
  *  Open MPI

## Basics

```cpp
#include "mpi.h"
#include <stdio.h>

int main(int argc, char *argv[])
{
    MPI_Init(&argc, &argv);
    printf("Hello, World!\n");
    MPI_Finalize();
    return 0;
}
```

### MPI_init

MPI_init() must be called before any other MPI functions can be called and it should be called only once.

```cpp
int MPI_Init(int *argc, char ***argv)
```

### MPI_Finalize

All MPI processes must call this routine before exiting.

```cpp
int MPI_Finalize()
```

### Compile

Library version: User tells compiler where header file and library are 

```shell
$ gcc -Iheaderdir -Llibdir mpicode.c –lmpich
```

Wrapper version: hides the details from the user

```shell
$ mpicc -o executable mpicode.c
```

### Execute

```shell
$ mpiexec -n 8 -host node1,node2 /hello
```

```shell
$ mpiexec -n 8 -hostfile <hostfile> ./hello
```

## Communicators

![https://computing.llnl.gov/tutorials/mpi/](2020-01-31-17-17-49.png)

* Communicator is an internal object
* MPI Programs are made up of **communicating processes**
  * Each process has its own address space containing its own attributes such as rank, size (and argc, argv, etc.)
  * MPI provides functions to interact with it
* Default communicator is `MPI_COMM_WORLD`:
  * All processes are its members
  * It has a size (the number of processes)
  * Each process has a unique rank within it
  * One can think of it as an ordered list of processes
* Additional communicator(s) can co-exist
* A process can belong to more than one communicator

```cpp
#include "mpi.h"
#include <stdio.h>

int main(int argc, char *argv[])
{
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    printf("Hello, World! from %d of %d\n", rank, size);
    MPI_Finalize();
    return 0;
}
```

### MPI_Comm_size

Determines the size of the group associated with a communicator.

```cpp
int MPI_Comm_size (MPI_Comm comm, int *size)
```

### MPI_Comm_rank

Returns the rank of the calling process in the group underlying the comm.

```cpp
int MPI_Comm_rank (MPI_Comm comm, int *rank)
```


[^1]: Tim Mattson, “Recent developments in parallel programming: the good, the bad, and the ugly”, Euro-Par 2013
