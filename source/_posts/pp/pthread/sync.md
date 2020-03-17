---
title: 'Synchronization - Pthread'
tags:
- PP
---

# Synchronization

## Busy-Waiting

Use flag and while loop

## Mutex

mutual exclusion, `pthread_mutex_t`

### pthread_mutex_init

```cpp
int pthread_mutex_init(pthread_mutex_t *restrict               mutex,
                       const pthread_mutexattr_t *restrict     attr);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_mutex_init.3p.html

### pthread_mutex_lock

```cpp
int pthread_mutex_lock(pthread_mutex_t *mutex);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_mutex_lock.3p.html

### pthread_mutex_unlock

```cpp
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_mutex_lock.3p.html

### pthread_mutex_destroy

```cpp
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_mutex_destroy.3p.html

### Example

```cpp
#include <pthread.h>

pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);

pthread_mutex_lock(&mutex);
// critical section
pthread_mutex_unlock(&mutex);

pthread_mutex_destroy(&mutex);
```

### Performance

* #threads < #cores
  * busy-waiting ≈ mutex
  * each thread only enters the critical section once
* #threads > #cores
  * busy-waiting degrade gradually

### Adaptability

Busy-waiting:

* the order in which the threads will execute the code in the critical section
* thread 0 is first, then thread 1, then thread 2, and so on

mutex:

* the order in which the threads execute the critical section is left to chance and the system

## Semaphore

* a data structure consisting of a **counter** and a **task descriptor queue**
* two operations, **wait** and **release(post)**
* `sem_t`

### sem_init

```cpp
int sem_init(sem_t *sem,
             int pshared,
             unsigned int value);
```

pshared = 0: shared between threads in the same process
pshared = nonzero: shared between processes

Ref: http://man7.org/linux/man-pages/man3/sem_init.3.html

### sem_wait

* block if the semaphore is 0
* decrement the semaphore and proceed if the semaphore is nonze

```cpp
int sem_wait(sem_t *sem);
```

Ref: http://man7.org/linux/man-pages/man3/sem_timedwait.3.html

### sem_post

* increments the semaphore

```cpp
int sem_post(sem_t *sem);
```

Ref: http://man7.org/linux/man-pages/man3/sem_post.3.html

### sem_destroy

```cpp
int sem_destroy(sem_t *sem);
```

Ref: http://man7.org/linux/man-pages/man3/sem_destroy.3.html

### Example

```cpp
#include <semaphore.h>

sem_t semaphore;
sem_init(&semaphore, 0, 0);

sem_wait(&semaphore)
// critical section
sem_wait(&semaphore)

sem_destroy(&semaphore);
```