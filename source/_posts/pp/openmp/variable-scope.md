---
title: 'Variable Scope - OpenMP'
tags: 'PP'
---

# Variable Scope

* shared
* private
* firstprivate
* lastprivate
* default

> All data clauses apply to parallel constructs and worksharing constructs except `shared` only applies to parallel constructs

## private

`private(var)` 把 var copy 到 local thread 中，預設沒有  initialized (OpenMP 2.5)

```cpp
void wrong() {
  int tmp = 0;
#pragma omp parallel for private(tmp)
  for (int j = 0; j < 1000; ++j) tmp += j;
  printf("%d\n", tmp); // output 0
}
```

## firstprivate

拿前面定義的當作初始值

```cpp
void useless() {
  int tmp = 0;
#pragma omp parallel for firstprivate(tmp)
  for (int j = 0; j < 1000; ++j) tmp += j;
  printf("%d\n", tmp); // output 0
}
```

## lastprivate

最後的 iteration 算完的值會被 copy 出來

```cpp
void closer() {
  int tmp = 0;
#pragma omp parallel for firstprivate(tmp) lastprivate(tmp)
  for (int j = 0; j < 1000; ++j) tmp += j; // each thread initialize tmp to 0
  printf("%d\n", tmp); // last sequential iteration
}
```

## default

> 預設為 `default(shared)`

default(none): 自己明確定義到底是 private 還是 shared
