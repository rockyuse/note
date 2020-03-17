---
title: 'Pthread'
tags:
- PP
---

# Pthread

The POSIX threading **interface**. The implementation of pthread depends on the platform. (e.g. Linux: kernel thread, Windows: Win32 threads)

## Process vs Thread

![multi-threading](2020-01-27-22-35-55.png)

| Property                    | Multi-processing                   | Multi-threading                                       |
| --------------------------- | ---------------------------------- | ----------------------------------------------------- |
| Variables                   | copies all variables               | share global variables                                |
| IDs                         | new process ID                     | same process ID, unique thread ID                     |
| Communication               | explicitly communicate (e.g. pipe) | may communicate with return value or shared variables |
| Parallelism (one CPU)       | concurrent                         | concurrent                                            |
| Parallelism (multiple CPUs) | may be executed simultaneously     | kernel threads may be executed simultaneously         |

## Operations

```cpp
#include <pthread.h>
```

Compile and link with -pthread.

### pthread_create

create a thread

![](2020-01-27-23-16-27.png)

```cpp
int pthread_create(pthread_t            *thread,                    // pthread_t object
                   const pthread_attr_t *attr,                      // thread attribute
                   void                 *(*start_routine) (void *), // function
                   void                 *arg);                      // function arguments
```

Ref:

* https://pages.mtu.edu/~shene/NSF-3/e-Book/FUNDAMENTALS/thread-management.html
* http://man7.org/linux/man-pages/man3/pthread_create.3.html

### pthread_join

wait for a thread

![](2020-01-27-23-34-19.png)

```cpp
int pthread_join(pthread_t  thread,
                 void       **retval);  // return value of thread
```

Ref: http://man7.org/linux/man-pages/man3/pthread_join.3.html

### pthread_exit

exit a thread without exiting process

```cpp
void pthread_exit(void *retval);
```

Ref: http://man7.org/linux/man-pages/man3/pthread_exit.3.html

### pthread_detach

set thread to release resources

Ref: http://man7.org/linux/man-pages/man3/pthread_detach.3.html

### pthread_equal

test two thread IDs for equality

Ref: http://man7.org/linux/man-pages/man3/pthread_equal.3.html

### pthread_kill

send a signal to a thread

Ref: http://man7.org/linux/man-pages/man3/pthread_kill.3.html

### pthread_cancel

send a cancellation request to a thread

Ref: http://man7.org/linux/man-pages/man3/pthread_cancel.3.html

### pthread_self

find out own thread ID

Ref: http://man7.org/linux/man-pages/man3/pthread_self.3.html

## Thread Attributes

A thread may be created as a 

* **joinable** thread (the default)
  * When it terminates, its exit state is still **kept** in the system until another thread calls `pthread_join()` to obtain its return value
* **detached** thread
  * cleaned up automatically after thread terminates
  * can't `pthread_join` or obtain its return value

### Basic management

Create (initialize with default values) and destroy a thread attribute object

* [pthread_attr_init](http://man7.org/linux/man-pages/man3/pthread_attr_init.3.html)
* [pthread_attr_destroy](http://man7.org/linux/man-pages/man3/pthread_attr_destroy.3p.html)

### Specifying stack information

Get and set the stack size in the attribute object for new threads

* [pthread_attr_getstacksize](http://man7.org/linux/man-pages/man3/pthread_attr_getstacksize.3p.html)
* [pthread_attr_setstacksize](http://man7.org/linux/man-pages/man3/pthread_attr_setstacksize.3.html)

### Thread scheduling attributes

Get and set the value of the schedule policy attribute of a thread attributes object

* [pthread_attr_getschedpolicy](http://man7.org/linux/man-pages/man3/pthread_attr_getschedpolicy.3p.html)
* [pthread_attr_setschedpolicy](http://man7.org/linux/man-pages/man3/pthread_attr_setschedpolicy.3.html)

### Detachable or joinable (undetachable)

Control whether the thread is created in the joinable state

* [pthread_attr_getdetachstate](http://man7.org/linux/man-pages/man3/pthread_attr_getdetachstate.3p.html)
* [pthread_attr_setdetachstate](http://man7.org/linux/man-pages/man3/pthread_attr_setdetachstate.3.html)

### Example

Set attribute to detached state

```cpp
int main () {
    pthread_attr_t attr;
    pthread_t thread;

    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    pthread_create(&thread, &attr, &thread_function, NULL);
    pthread_attr_destroy(&attr);

    // do something

    // no need to join thread
    return 0;
}
```
