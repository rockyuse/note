---
title: 'Advanced - Pthread'
tags:
- PP
---

# Advanced

## Barriers

### Busy-waiting

Use a shared counter protected by the mutex

```cpp
pthread_mutex_lock(&barrier mutex);
counter++;
pthread_mutex_unlock(&barrier mutex)
while (counter < thread count);
```

### Semaphores

Use two semaphores

* `count_sem = 1` protects the counter
* `barrier_sem = 0` blocks threads that have entered the barrier

```cpp
/* Shared variables */
int counter; /* Initialize to 0 */ 
sem_t count_sem; /* Initialize to 1 */
sem_t barrier_sem; /* Initialize to 0 */ 
...
Thread_work(...) { 
    ...
    /* Barrier */
    sem_wait(&count_sem);
    if (counter == thread_count−1) { 
        counter = 0;
        sem_post(&count_sem);
        for (j = 0; j < thread_count−1; j++) 
            sem_post(&barrier_sem); 
    } else {
        counter++;
        sem_post(&count_sem); 
        sem_wait(&barrier_sem); 
    } 
    ... 
}
```

### Condition Variables

a data object that allows a thread to suspend execution until a certain event or condition occurs

* When the event or condition occurs, another thread can signal the thread to "wake up"
* A condition variable is **always** associated with a mutex
* `pthread_cond_t`

#### pseudocode

```
lock mutex;
if condition has occurred
    signal thread(s);
else {
    unlock the mutex and block;
    /* when thread is unblocked, mutex is relocked */
}
unlock mutex;
```

#### pthread_cond_init

```cpp
int pthread_cond_init(pthread_cond_t *restrict              cond,
                      const pthread_condattr_t *restrict    attr);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_cond_destroy.3p.html

#### pthread_cond_signal

Unblock one of the blocked threads

```cpp
int pthread_cond_signal(pthread_cond_t *cond);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_cond_signal.3p.html

#### pthread_cond_broadcast

Unblock all of the blocked threads

```cpp
int pthread_cond_broadcast(pthread_cond_t *cond);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_cond_broadcast.3p.html

#### pthread_cond_wait

* Unlock `mutex`
* Block thread until another thread's call to `pthread_cond_signal` or `pthread_cond_broadcast`
* When the thread is unblocked, it reacquires the mutex

```cpp
int pthread_cond_wait(pthread_cond_t *restrict      cond,
                      pthread_mutex_t *restrict     mutex);
```

Upon successful completion, a value of zero shall be returned;

Ref: http://man7.org/linux/man-pages/man3/pthread_cond_timedwait.3p.html

#### pthread_cond_destroy

```cpp
int pthread_cond_destroy(pthread_cond_t *cond);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_cond_destroy.3p.html

#### Example

```cpp
/* Shared */
int counter = 0;
pthread_mutex_t mutex;
pthread_cond_t cond_var;
...
void* Thread_work(...) { 
    ... 
    /* Barrier */
    pthread_mutex_lock(&mutex);
    counter++;
    if (counter == thread_count - 1) { 
        counter = 0; 
        pthread_cond_broadcast(&cond_var); 
    } else {
        while (pthread_cond_wait(&cond_var, &mutex) != 0);
    } 
    pthread_mutex_unlock(&mutex); 
    ... 
}
```

while: 當 `pthread_cond_wait` 失敗後重新 wait

## Read-Write Locks

A read-write lock is somewhat like a mutex except that it provides two lock functions

* Locks the read-write lock for *reading*
* Locks the read-write lock for *writing*

Multiple threads can **simultaneously** obtain the lock by calling the *read-lock* function, while **only one thread** can obtain the lock by calling the *write-lock* function

* 當有任何 thread 持有 reading lock，其他想要獲得 writing lock 的 thread 會被 block 住
* 當某個 thread 持有 writing lock，其他想要獲得 reading lock 的 thread 會被 block 住

### pthread_rwlock_init

```cpp
int pthread_rwlock_init(pthread_rwlock_t *restrict              rwlock,
                        const pthread_rwlockattr_t *restrict    attr);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_rwlock_destroy.3p.html

### pthread_rwlock_rdlock

```cpp
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_rwlock_rdlock.3p.html

### pthread_rwlock_wrlock

```cpp
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_rwlock_wrlock.3p.html

### pthread_rwlock_unlock

```cpp
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_rwlock_unlock.3p.html

### pthread_rwlock_destroy

```cpp
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_rwlock_destroy.3p.html

## Issue

Slow may due to:

* Write miss
* Read miss
* Cache Coherence

### Cache Coherence

![](2020-01-28-16-00-55.png)

cache 在某個 process 寫入後應要使其他 process 也看到新的值

How to fix with a bus: Coherence Protocol

* Use bus to broadcast writes or invalidation
* Simple protocols rely on presence of broadcast medium

#### False Sharing

在多個 Core 的架構下，因為 Cache line 彼此獨立，若兩個 Core 都載入某一段相同的 memory，且當 Core 1 更新某個值，會造成 Core 2 讀取某個變數的值時會 cache miss 而重新載入 memory，即使讀取的變數不是 Core 1 更新的變數。

False Sharing 最常在 SPMD (Single-Program, Multiple-Data) 的平行化程式中遇到，因為 Array 在記憶體中是連續的，且載入時會因為 Spatial Locality 而載入到 Cache Line 中，因此很容易造成 False Sharing。

Solution: Array Padding，避免載入到別的 Core 需要處理的資料到 Cache line 中。
