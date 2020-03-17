---
title: 'Synchronization - OpenMP'
tags: 'PP'
---

# Synchronization

* High level synchronization
    * critical
    * atomic
    * barrier
    * ordered
* Low level synchronization
    * flush
    * locks

## critical

Mutual exclusion: 同時只有一個 Thread 可以進入 Critical

```cpp
#pragma omp parallel
{
  for (i = id; i < niters; i + nthrds) {
    B = big_job(i);
#pragma omp critical
    res += consume(B);
  }
}
```

> In OpenMP, all unnamed critical sections are considered identical
> Naming: `#pragma omp critical(name)`

## atomic

Mutual exclusion but only applies to the update of a memory location (`res` in example)

* `x binop= expr`
* `x++`
* `++x`
* `x--`
* `--x`

```cpp
#pragma omp parallel
{
  for (i = id; i < niters; i + nthrds) {
    B = big_job(i);
#pragma omp atomic
    res += B;
  }
}
```

## barrier

Thread 會 wait 在 barrier 上

```
#pragma omp barrier
```

## ordered

`#pragma omp ordered` 後的會 sequential 執行

```cpp
#pragma omp parallel for ordered
  for (int i = 0; i < 5; ++i) {
    printf("%d\n", i);
#pragma omp ordered
    printf("%d\n", 10+i);
  }
}
```

```
$ g++ test.cpp -o test -fopenmp && ./test
0
10
1
11
3
4
2
12
13
14
```