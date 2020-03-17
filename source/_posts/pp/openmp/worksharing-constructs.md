---
title: 'Worksharing Constructs - OpenMP'
tags: 'PP'
---

# Worksharing Constructs

A worksharing construct distributes the execution of the corresponding region among the members of the team that encounters it. Threads execute portions of the region in the context of the implicit tasks that each one is executing.

## Loop Construct

![](2020-01-28-22-51-36.png)

Sequential:

```cpp
for (i = 0; i < N; i++) {
    a[i] = a[i] + b[i];
}
```

若用 Parallel 就會變得很冗長：

```cpp
#pragma omp parallel
{
  int id, Nthrds, istart, iend;
  id = omp_get_thread_num();
  Nthrds = omp_get_num_threads();
  istart = id * N / Nthrds;
  iend = (id + 1) * N / Nthrds;
  if (id == Nthrds - 1) iend = N;
  for (i = istart; i < iend; i++) {
    a[i] = a[i] + b[i];
  }
}
```

用 loop construct：

```cpp
#pragma omp parallel
{
#pragma omp for
  for (i = 0; i < N; i++) {
    a[i] = a[i] + b[i];
  }
}
```

或是

```cpp
#pragma omp parallel for
  for (i = 0; i < N; i++) {
    a[i] = a[i] + b[i];
  }
```

> 預設變數 `i` 為 private

也可以使用 `collapse` 來表達 Nested Loop，會展開成 N * M 的單層 for loop

```cpp
#pragma omp parallel for collapse(2)
for (int i = 0; i < N; i++) {
  for (int j = 0; j < M; j++) {
    ...
  }
}
```

### Schedule

* `schedule(static[,chunk])`
* `schedule(dynamic[,chunk])`
    * 有些 Iteration 做比較久，在 Runtime 才知道
    * 一次先拿固定量的 chunk，做完再拿下一份
    * 缺點：在 Runtime 才決定要拿哪些 Task 會花時間、chunk 切太小會影響效能
* `schedule(guided[,chunk])`
    * 為了解決 dynamic 的缺點
    * Thread 一次拿剩下的 Iteration 的一半來做
    * 直到剩下的 Iteration 數量小於 chunk，就直接做完
* `schedule(runtime)`
    * `$ export OMP_SCHEDULE="static,1"`
* `schedule(auto)`

Default: `schedule(static, total_iterations/thread_count)`

For example:

```
schedule(static, 1):
Thread 0: 0, 3, 6, 9
Thread 1: 1, 4, 7, 10
Thread 2: 2, 5, 8, 11

schedule(static, 2):
Thread 0: 0, 1, 6, 7
Thread 1: 2, 3, 8, 9
Thread 2: 4, 5, 10, 11
```

### Reduction

![](2020-01-28-22-54-13.png)

```cpp
double ave = 0.0, A[MAX];
int i;
for (i = 0; i < MAX; i++) {
  ave += A[i];
}
ave = ave / MAX;
```

透過 `reduction (op : list)` 來處理平行化後 thread 間的 dependency (`ave`)

```cpp
double ave = 0.0, A[MAX];
int i;
#pragma omp parallel for reduction(+ : ave)
for (i = 0; i < MAX; i++) {
  ave += A[i];
}
ave = ave / MAX;
```

> list variable 在 thread 中初始化時會依照 op 來決定初始值

### Caveats

* `parallel for` 不會檢查 dependency
* 沒有辦法平行化 `while` 或 `do-while`
* for loops 要是 [canonical form](https://www.openmp.org/spec-html/5.0/openmpsu40.html)
    * index 要是 int or pointer type
    * start, end, and incr 要 compatible
    * start, end, and incr 在 loop body 中不會被改變

## Master/Single Construct

只有 Master Thread 會執行：

```cpp
#pragma omp parallel
{
  do_many_things();
#pragma omp master
  { exchange_boundaries(); }
#pragma omp barrier
  do_many_other_things();
}
```

只有單一個 Thread 會執行：

```cpp
#pragma omp parallel
{
  do_many_things();
#pragma omp single
  { exchange_boundaries(); }
  do_many_other_things();
}
```

> `single` 後有一個 barrier (可以用 `nowait` 移除 barrier)

## Sections/Section Construct

不同 Thread 執行不同 section

```cpp
#pragma omp parallel sections
{
#pragma omp section
  double a = alice();
#pragma omp section
  double b = bob();
#pragma omp section
  double c = cy();
}
```

> `sections` 後有一個 barrier (可以用 `nowait` 移除 barrier)

## Task Construct

![](2020-01-28-23-10-34.png)

Handle irregular patterns and recursive function calls

```cpp
#pragma omp parallel
{
#pragma omp single
  {
    node *p = head; // block 1
    while (p) {
#pragma omp task firstprivate(p) // single thread create a task with local value of pointer p
      process(p); // block 2
      p = p->next; // block 3
    }
  }
}
```

### Barriers

Thread barriers: Applies to all tasks generated in the current parallel region up to the barrier

```cpp
#pragma omp parallel
{
#pragma omp task
  foo(); // multiple foo() tasks created
#pragma omp barrier // All foo tasks guaranteed to be completed here

#pragma omp single
  {
#pragma omp task // one bar() task created
    bar();
  } // bar task guaranteed to be complete here
}
```

Task barriers: Applies only to tasks generated in the current task (i.e. wait until all tasks defined in the current task have completed)

```cpp
#pragma omp task  // Task 1
{
  ...
#pragma omp task  // Task 2
  {
    do_work1();
  }
#pragma omp task  // Task 3
  {
    ...
#pragma omp task  // Task 4
    {
      do_work2();
    }
#pragma omp taskwait // Task 3 wait for the completion of Task 4.
                     // It does not wait for the completion of Task 2 or Task 1
    ...
  }
}
```

### if Clause

expression 為 true 則建立一個 task

```cpp
#pragma omp task if (expression)
```

### final Clause

expression 為 true 則不要再繼續建立 task

```cpp
#pragma omp task final (expression)
```

### untied Clause

Specifies that the task is never tied to the thread that started its execution. Any thread in the team can resume the task region after a suspension. (thread switch) 

```cpp
#pragma omp task untied
```

### mergeable Clause

A merged task is a task whose data environment is the same as that of its generating task region, which allows the implementation to reuse the environment from another task

```cpp
#pragma omp task mergeable
```
