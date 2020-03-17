---
title: 'Shared Memory'
tags:
- PP
---

# Shared Memory

多個 process 共享同一個 Memory space

## API

### shmget()

create a block of shared memory

```cpp
int shmget(key_t key, size_t size, int shmflg);
```

通常在 IPC (Inter-Process Communication) 中，每個 IPC Object 都會對應一個 key，通過不同的 key 來辨別對象。

shmget 中的 key 可以指定為 `IPC_PRIVATE` (0) 或是透過 `ftok(filepath, id)` 取得 key 後明確指定，若成功則會回傳該 key 對應的 shared memory identifier。

`IPC_PRIVATE` 通常用在父子進程在 fork 之後的溝通。

Ref:

* http://man7.org/linux/man-pages/man2/shmget.2.html
* http://man7.org/linux/man-pages/man3/ftok.3.html

### shmat()

attach shared memory to the current process's address space 

```cpp
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

Ref: http://man7.org/linux/man-pages/man2/shmat.2.html

### shmdt()

detach shared memory from the current process's address space

```cpp
int shmdt(const void *shmaddr);
```

Ref: http://man7.org/linux/man-pages/man3/shmdt.3p.html

### shmctl()

control shared memory

```cpp
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

Ref: http://man7.org/linux/man-pages/man2/shmctl.2.html

## Example

```cpp
int main() {
    int dim;
    cout << "Input the matrix dimension: ";
    cin >> dim;

    int arrSize = sizeof(int) * dim * dim;
    int shmid_A = shmget(IPC_PRIVATE, arrSize, IPC_CREAT | 0600);
    uint* A = (uint*)shmat(shmid_A, NULL, 0);
    int shmid_B = shmget(IPC_PRIVATE, arrSize, IPC_CREAT | 0600);
    uint* B = (uint*)shmat(shmid_B, NULL, 0);
    int shmid_C = shmget(IPC_PRIVATE, arrSize, IPC_CREAT | 0600);
    uint* C = (uint*)shmat(shmid_C, NULL, 0);

    if (shmid_A < 0 || shmid_B < 0 || shmid_C < 0) {
        perror("shmget");
        return 0;
    }
    if (A == (uint*)-1 || B == (uint*)-1 || C == (uint*)-1) {
        perror("shmat");
        return 0;
    }

    // using fork to do something with pointer A, B, C

    shmdt(A);
    shmdt(B);
    shmdt(C);

    shmctl(shmid_A, IPC_RMID, NULL);
    shmctl(shmid_B, IPC_RMID, NULL);
    shmctl(shmid_C, IPC_RMID, NULL);

    return 0;
}
```