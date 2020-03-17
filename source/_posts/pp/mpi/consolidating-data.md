---
title: 'Consolidating Data - MPI'
tags: 'PP'
---

# Consolidating Data

```cpp
double x[1000];
...
if (my rank == 0)
    for (i = 0; i < 1000; i++)
        MPI_Send(&x[i], 1, MPI DOUBLE, 1, 0, comm);
else /* my rank == 1 */
    for (i = 0; i < 1000; i++)
        MPI_Recv(&x[i], 1, MPI DOUBLE, 0, 0, comm, &status);
```

is much slower than

```cpp
double x[1000];
if (my rank == 0)
    MPI_Send(x, 1000, MPI DOUBLE, 1, 0, comm);
else /* my rank == 1 */
    MPI_Recv(x, 1000, MPI DOUBLE, 0, 0, comm, &status);
```

MPI provides three basic approaches to consolidating data that might otherwise require multiple messages

* the count argument to the various communication functions
* derived datatypes
* MPI_Pack/Unpack

## Derived Datatypes

* A sending process can pack noncontiguous data into contiguous buffer and send the buffered data to a receiving process that can unpack the contiguous buffer and store the data to noncontiguous location
* A derived datatype is an opaque object that specifies
  * A sequence of primitive datatypes
  * A sequence of integer (byte) displacements

MPI Basic Data Types: https://www.mpi-forum.org/docs/mpi-2.1/mpi21-report-bw/node330.htm

### Contiguous

Contiguous datatype constructors create a new datatype by making count copies of existing data type (old_type)

```cpp
int MPI_Type_contiguous(int             count,
                        MPI_Datatype    old_type,
                        MPI_Datatype*   new_type)
```

Send a row of data once.

```cpp
#include <stdio.h>
#include "mpi.h"

#define SIZE 4

int main(int argc, char* argv[]) {
    int numtasks, rank, source = 0, dest, tag = 1, i;
    MPI_Request req;
    float a[SIZE][SIZE] = {1.0, 2.0,  3.0,  4.0,  5.0,  6.0,  7.0,  8.0,
                           9.0, 10.0, 11.0, 12.0, 13.0, 14.0, 15.0, 16.0};
    float b[SIZE];
    MPI_Status stat;
    MPI_Datatype rowtype;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &numtasks);
    MPI_Type_contiguous(SIZE, MPI_FLOAT, &rowtype); /* Homogenous datastructure of size 4 */
    MPI_Type_commit(&rowtype);
    if (numtasks == SIZE) {
        if (rank == 0) {
            for (i = 0; i < numtasks; i++) {
                dest = i;
                MPI_Isend(&a[i][0], 1, rowtype, dest, tag, MPI_COMM_WORLD, &req);
            }
        }
        MPI_Recv(b, SIZE, MPI_FLOAT, source, tag, MPI_COMM_WORLD, &stat);
        printf("rank = %d b = %3.1f %3.1f %3.1f %3.1f\n", rank, b[0], b[1], b[2], b[3]);
    } else {
        printf("Must specify %d processors.Terminating.\n", SIZE);
    }
    MPI_Type_free(&rowtype);
    MPI_Finalize();
}
```

### Vector

Returns a new datatype that represents equally spaced blocks. The spacing between the start of each block is given in units of extent (oldtype).

```cpp
int MPI_Type_vector(int                 count,      /* the number of blocks */
                    int                 blocklen,   /* the number of elements in each block */
                    int                 stride,     /* the number of elements between start of each block of the old_type */
                    MPI_Datatype        old_type,
                    MPI_Datatype*       newtype)
```

![](2020-01-31-22-08-59.png)

Send a coloumn of data once.

```cpp
#include <stdio.h>
#include "mpi.h"

#define SIZE 4

int main(int argc, char* argv[]) {
    int numtasks, rank, source = 0, dest, tag = 1, i;
    MPI_Request req;
    float a[SIZE][SIZE] = {1.0, 2.0,  3.0,  4.0,  5.0,  6.0,  7.0,  8.0,
                           9.0, 10.0, 11.0, 12.0, 13.0, 14.0, 15.0, 16.0};
    float b[SIZE];
    MPI_Status stat;
    MPI_Datatype columntype;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &numtasks);
    MPI_Type_vector(SIZE, 1, SIZE, MPI_FLOAT, &columntype);
    MPI_Type_commit(&columntype);
    if (numtasks == SIZE) {
        if (rank == 0) {
            for (i = 0; i < numtasks; i++)
                MPI_Isend(&a[0][i], 1, columntype, i, tag, MPI_COMM_WORLD, &req);
        }
        MPI_Recv(b, SIZE, MPI_FLOAT, source, tag, MPI_COMM_WORLD, &stat);
        printf("rank= %d b= %3.1f %3.1f %3.1f %3.1f\n", rank, b[0], b[1], b[2], b[3]);
    } else {
        printf("Must specify %d processors. Terminating.\n", SIZE);
    }
    MPI_Type_free(&columntype);
    MPI_Finalize();
}
```

### Indexed

* Returns a new datatype that represents count blocks.
* Displacements are expressed in units of extent(oldtype).
* The `count` is
  * the number of blocks 
  * the number of entries in `array_of_displacements` (displacement of each block in units of the oldtype) and `array_of_blocklengths` (number of instances of oldtype in each block)

```cpp
int MPI_Type_indexed(int                count,
                     int*               array_of_blocklengths,
                     int*               array_of_displacements,
                     MPI_Datatype       oldtype,
                     MPI_datatype*      newtype);
```

![](2020-01-31-22-17-12.png)

* 從 0 (displacements[0]) 開始後 2 個 (blocklengths[0])
* 從 4 (displacements[1]) 開始後 4 個 (blocklengths[1])

```cpp
#include <stdio.h>
#include "mpi.h"

#define NELEMENTS 6

int main(int argc, char* argv[]) {
    int numtasks, rank, source = 0, dest, tag = 1, i;
    MPI_Request req;
    float a[16] = {1.0, 2.0,  3.0,  4.0,  5.0,  6.0,  7.0,  8.0,
                   9.0, 10.0, 11.0, 12.0, 13.0, 14.0, 15.0, 16.0};
    float b[NELEMENTS];
    MPI_Status stat;
    MPI_Datatype indextype;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &numtasks);
    int blocklengths[2] = {4, 2};
    int displacements[2] = {5, 12};
    MPI_Type_indexed(2, blocklengths, displacements, MPI_FLOAT, &indextype);
    MPI_Type_commit(&indextype);
    if (rank == 0) {
        for (i = 0; i < numtasks; i++)
            MPI_Isend(a, 1, indextype, i, tag, MPI_COMM_WORLD, &req);
    }
    MPI_Recv(b, NELEMENTS, MPI_FLOAT, source, tag, MPI_COMM_WORLD, &stat);
    printf("rank= %d b= %3.1f %3.1f %3.1f %3.1f %3.1f %3.1f\n", rank, b[0], b[1], b[2], b[3], b[4], b[5]);
    MPI_Type_free(&indextype);
    MPI_Finalize();
}
```

```
rank= 0 b = 6.0 7.0 8.0 9.0 13.0 14.0
rank= 2 b = 6.0 7.0 8.0 9.0 13.0 14.0
rank= 1 b = 6.0 7.0 8.0 9.0 13.0 14.0
rank= 3 b = 6.0 7.0 8.0 9.0 13.0 14.0
```

### Struct

* Returns a new datatype that represents count blocks.
* Displacements are expressed in `bytes`.
* `count` is an integer that specifies the number of blocks (number of entries in arrays).
* `array_of_blocklengths` is the number of elements in each blocks
* `array_of_displacements` specifies the byte displacement of each block.
* `array_of_types` represent type of elements in each block. 

```cpp
int MPI_Type_create_struct(int              count,
                           int*             array_of_blocklengths,
                           MPI_Aint*        array_of_displacements,
                           MPI_Datatype*    array_of_types,
                           MPI_Datatype*    newtype)
```

```cpp
#include <stdio.h>
#include "mpi.h"

#define NELEM 25

typedef struct {
    float x, y, z;
    float velocity;
    int n, type;
} Particle;

int main(int argc, char* argv[]) {
    int numtasks, rank, source = 0, dest, tag = 1, i;
    Particle p[NELEM], particles[NELEM];
    /* MPI_Aint type used to be consistent with syntax of */
    /* MPI_Type_get_extent routine */
    MPI_Aint lb, float_extent;
    MPI_Status stat;
    /* Float, Float, Float, Float, Int, Int */
    int blockcounts[2] = {4, 2};
    MPI_Aint offsets[2] = {0, 4 * float_extent};
    MPI_Datatype particletype, oldtypes[2] = {MPI_FLOAT, MPI_INT};
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &numtasks);
    MPI_Type_get_extent(MPI_FLOAT, &lb, &float_extent);
    MPI_Type_create_struct(2, blockcounts, offsets, oldtypes, &particletype);
    MPI_Type_commit(&particletype);
    if (rank == 0) {
        for (i = 0; i < NELEM; i++) {
            particles[i].x = i * 1.0;
            particles[i].y = i * -1.0;
            particles[i].z = i * 1.0;
            particles[i].velocity = 0.25;
            particles[i].n = i;
            particles[i].type = i % 2;
        }
        for (i = 0; i < numtasks; i++)
            MPI_Send(particles, NELEM, particletype, i, tag, MPI_COMM_WORLD);
    }
    MPI_Recv(p, NELEM, particletype, source, tag, MPI_COMM_WORLD, &stat);
    printf("rank= %d %3.2f %3.2f %3.2f %3.2f %d %d\n", rank, p[3].x, p[3].y, p[3].z, p[3].velocity, p[3].n, p[3].type);
    MPI_Type_free(&particletype);
    MPI_Finalize();
}
```

## MPI_Pack/MPI_Unpack

* We explicitly pack non-contiguous data into a contiguous buffer for transmission, then unpack it at the other end.
* When sending/receiving packed messages, must use `MPI_PACKED` datatype in send/receive calls.

### Packing

```cpp
int MPI_Pack(void*          inbuf,
             int            incount, 
             MPI_Datatype   datatype,
             void*          outbuf,
             int            outsize,
             int*           position,
             MPI_Comm       comm)
```

* will pack the information specified by `inbuf` and `incount` into the buffer space provided by `outbuf` and `outsize`
* the current packing call will pack this data starting at offset `position` in the `outbuf`
* `position` is incremented by the size of the packed message

### Unpacking

```cpp
int MPI_Unpack(void*        inbuf,
               int          insize,
               int*         position,
               void*        outbuf,
               int          outcount, 
               MPI_Datatype datatype,
               MPI_Comm     comm)
```

* unpacks message to `outbuf`
* updates the `position` argument so it can be used in a subsequent call to `MPI_Unpack`

### Example

Sender:

```cpp
int i, position = 0;
char c[100], buffer[110];
// pack
MPI_Pack(&i, 1, MPI_INT, buffer, 110, &position, MPI_COMM_WORLD);
MPI_Pack(c, 100, MPI_CHAR, buffer, 110, &position, MPI_COMM_WORLD);
// send
MPI_Send(buffer, position, MPI_PACKED, 1, 0, MPI_COMM_WORLD);
...
```

Receiver:

```cpp
int i, position = 0;
char c[100], buffer[110];
MPI_Recv(buffer, 110, MPI_PACKED, 0, 0, MPI_COMM_WORLD, &status);
// and unpack
MPI_Unpack(buffer, 110, &position, &i, 1, MPI_INT, MPI_COMM_WORLD);
MPI_Unpack(buffer, 110, &position, c, 100, MPI_CHAR, MPI_COMM_WORLD);
...
```

## Comparision

* Pack/Unpack is quicker/easier to program.
* Derived datatypes are more flexible in allowing complex derived datatypes to be defined.
* Pack/Unpack has higher overhead.
* Derived datatypes are better if the datatype is regularly reused