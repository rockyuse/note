---
title: 'OpenMP'
tags: 'PP'
---

# OpenMP

Pthread 等 thread library 對於 Programmer 要考慮的事情太多了(initialization, synchronization, thread creation, condition variables...）。所以有沒有一種工具是可以在 import 後直接幫我們自動找出程式中可以平行化的部分並優化它？

## Introduction

* OpenMP = Open specification for Multi-Processing
* 一些 API for multi-threaded shared-memory programs
* "light" syntax
* Multi-threading, shared address model

## What is OpenMP?

High-level API

* Preprocessor (compiler) directives (~80%)
    * Create Thread
    * Worksharing
    * Synchronization
* Library Calls (~19%)
    * 設定 Thread attributes
* Environment Variables (~1%)
    * 控制 Runtime-behavior


OpenMP Basic Defs: Solution Stack[^1]

![](https://i.imgur.com/y77Xw7Y.png)

## Execution Model

Fork–join model[^2]: Mater Thread 分成多個 Thread 同步執行

![](https://i.imgur.com/W69zya9.png)

## Syntax

```cpp
#pragma omp construct [clause [clause]…]
```

For example:

```cpp
#include <iostream>
#include "omp.h"

int thread_count = 3;
int main() {

#pragma omp parallel
  {
    int ID = omp_get_thread_num();
    printf("Hello from thread %d of %d\n", ID, thread_count);
  }

}
```

`#pragma omp parallel` 會建立預設的 Thread 數量 

```shell
$ g++ test.cpp -o test -fopenmp && ./test
Hello from thread 1 of 3
Hello from thread 0 of 3
Hello from thread 7 of 3
Hello from thread 2 of 3
Hello from thread 6 of 3
Hello from thread 3 of 3
Hello from thread 5 of 3
Hello from thread 4 of 3
```

### Creating Threads

可以在 Runtime request 指定數量的 Thread：

```cpp
omp_set_num_threads(4);
#pragma omp parallel
  {
    foobar();
  }
```

或是寫在 Parallel Region 中：

```cpp
#pragma omp parallel num_threads(4)
  {
    foobar();
  }
```

Compiler 會生成：

```cpp
void thunk() { foobar(); }

pthread_t tid[4];
for (int i = 1; i < 4; ++i) {
    pthread_create(&tid[i], 0, thunk, 0);
}
thunk();
for (int i = 1; i < 4; ++i) {
    pthread_join(tid[i], NULL);
}
```


[^1]: [A “Hands-on” Introduction to OpenMP P21](https://www.openmp.org/wp-content/uploads/Intro_To_OpenMP_Mattson.pdf)
[^2]: [Fork–join model - Wikipedia](https://en.wikipedia.org/wiki/Fork%E2%80%93join_model)