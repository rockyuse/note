---
title: 'Thread Pool - Pthread'
tags:
- PP
---

# Thread Pool

![](2020-01-28-16-02-02.png)

```cpp
typedef struct job_t {
    void (*function)(void*);
    void* argument;
} job_t;

typedef struct thread_data {
    ...
} thread_data;

class ThreadPool {
   public:
    int max_thread;
    queue<job_t> job_queue;
    sem_t queue_mutex;
    sem_t job_count;
    sem_t clean;
    bool shutdown = false;

    ThreadPool(int thread_count) {
        if (thread_count < 0) exit(1);
        this->max_thread = thread_count;
        // initial job queue mutex
        sem_init(&this->queue_mutex, 0, 1);
        sem_init(&this->job_count, 0, 0);
        sem_init(&this->clean, 0, 0);
        // start worker thread
        for (int i = 0; i < thread_count; i++) {
            pthread_t thread;
            pthread_create(&thread, NULL, worker_thread, (void*)this);
        }
    }

    void push_job(job_t job) {
        sem_wait(&this->queue_mutex);
        job_queue.push(job);
        sem_post(&this->queue_mutex);
        sem_post(&this->job_count);
    }

    ~ThreadPool() {
        this->shutdown = true;
        int destroy_thread_count = 0;
        while (destroy_thread_count != this->max_thread) {
            sem_post(&this->job_count);
            sem_getvalue(&this->clean, &destroy_thread_count);
        }
    }
};

static void* worker_thread(void* data) {
    ThreadPool* pool = (ThreadPool*)data;
    job_t job;
    while (1) {
        // keep waiting until new job comming
        sem_wait(&pool->job_count);

        if (pool->shutdown) break;

        // mutex to protect job queue
        sem_wait(&pool->queue_mutex);

        job = pool->job_queue.front();
        pool->job_queue.pop();

        sem_post(&pool->queue_mutex);

        // do task
        (*(job.function))(job.argument);
    }
    sem_post(&pool->clean);
    pthread_exit(NULL);
    return 0;
}

void job_func(void* arg) {
    ...
}

int main() {
    ThreadPool pool(thread_num);

    for (...) {
        job_t job;
        thread_data job_func_args;
        job.function = job_func;
        job.argument = job_func_args;
        pool.push_job(job);
    }
}
```