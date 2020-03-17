---
title: 'Additional Topics - MPI'
tags: 'PP'
---

# Additional Topics

## Jumpshot

https://www.mcs.anl.gov/research/projects/perfvis/software/viewers/index.htm

## Profiling

### MPI_Wtime

Returns time in seconds elapsed on the calling processor.

```cpp
double MPI_Wtime()
```

## Wildcard Arguments

* MPI_ANY_SOURCE
* MPI_ANY_TAG
* Only a receiver can use a wildcard argument.
* Senders must specify a process ran and a nonnegative tag

## Input/Ouput

* Although the MPI standard doesn't specify which processes have access to which I/O devices, virtually **all** MPI implementations allow all the processes in MPI_COMM_WORLD **full access to stdout and stderr**
  * so most MPI implementations allow all processes to execute `printf` and `fprintf(stderr, ...)`
* Unlike output, most MPI implementations only allow process 0 in MPI_COMM_WORLD access to stdin
  * This makes sense: If multiple processes have access to stdin, which process should get which parts of the input data?

## Collective vs. Point-to-point

* All the processes in the communicator must call the same collective function
* The arguments passed by each process to an MPI collective communication must be "compatible"
* Collective communications don't use tags

## Other Topics

* Topologies
  * map a communicator onto, say, a 3D Cartesian processor grid
* Rich set of I/O functions
  * individual, collective, blocking and non-blocking
* One-sided communication
  * puts and gets with various synchronization schemes
* Task creation and destruction
  * change number of tasks during a run