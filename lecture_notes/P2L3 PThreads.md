## Overview

In this section, `pthread`, or POSIX threads, will be used as a concrete representation of multithreading. `pthread` is the de facto standard in UNIX systems.
###### #DEFINE POSIX
> POSIX stands for **Portable Operating System Interface**, which describes an **interface that operating systems should support**. Its intent is to increase interoperability among OSes.

`pthread` therefore describes the threading interface that OS should support, which includes syntax specification and the semantics of threading-related operationgs.
## PThread Creation

Firstly, there must be a type associated with `pthread`:

```c
pthread_t aThread; /* thread type */
```

- The `pthread_t` data structure itself will maintain information such as the thread ID, execution state, and any other information that is relevant to the thread. As developers using the `pthread` library, this is mostly abstracted away from us.

### Fork and Join

An equivalent to the `Fork` operation would be the `pthread_create` function:

```c
int pthread_create(
        pthread_t * thread, 
        const pthread_attr_t * attr, 
        void * (start_routine)(void *),
        void * arg);
```

- The `pthread_t thread` data structure will be populated with relevant information such that the thread can start executing.
- The `attr` argument specifies how the thread manager should handle this particular thread. This will be covered in slightly more detail later.
- The `start_routine` and `arg` are similar to `proc` and `args` in the `Fork` operation.
- This function returns an integer that indicates whether the creation succeeded or failed.

Lastly, an equivalent to the `Join` operation would be `pthread_join`:

```c
int pthread_join(pthread_t thread, void ** status);
```
- The `thread` argument specifies the thread to join.
- The `status` argument will be populated with all of the relevant return information, as well as the results that are returned from the thread.
- This function also returns an integer that indicates whether the join operation succeeded or failed.
### Attributes

Recall that thread forking (`pthread_create`) allowed for a `pthread_attr_t attr` argument to be specified. This allows us to define features of this newly created thread. For example, it is possible to define:
- stack size,
- scheduling policy,
- priority,
- scope (system/process),
- inheritance (should this thread inherit attributes of the calling thread?)
- whether or not the thread is **joinable**.

There are default values in place, and can be invoked by passing `NULL` as the `attr` argument for `pthread_create`. 

Here are other commonly used functions that deal with `pthread_attr_t`:

```c
/* initialises attribute data structure */
int pthread_attr_init(pthread_attr_t * attr);

/* frees memory associated with attr */
int pthread_attr_destroy(pthread_attr_t * attr);

/* getters and setters */
pthread_attr_{set/get}{attribute};
```

One particular attribute will be given special attention: the **joinable attribute**. In order to discuss this attribute, we must first cover a new concept regarding detaching threads.

The default behaviour of `pthread` results in joinable threads. This is similar to what we have encountered during the abstract discussion regarding threads. As recap: the main parent thread must call `Join` after creating threads in order to block and wait for the child threads to finish executing. If `Join` was not called, the child threads might have already finished, but they were not exited in the correct manner.
###### #DEFINE Zombie Threads
> A zombie thread is a joinable thread which has **terminated, but which has not been joined.**. Normally, a thread should be joined at some time, or it should have been detached. 
> 
> The **OS maintains its state for some possible future join, which takes up resources.** A terminated thread must be joined by another thread in order to be recycled.

In `pthread`, there is a possibility for child threads to be detached from the parent thread. In this case, even if the parent exits, the child threads are free to continue. Note that detached threads **cannot be joined**.

It is possible to detach threads after creation by invoking `pthread_detach`:

```c
int pthread_detach(pthread_t thread);
```

It also also possible to **create detached threads** by setting the appropriate detached attribute:

```c
pthread_attr_setdetachstate(attr, PTHREAD_CREATE_DETACHED);
// ...
pthread_create(..., attr, ...);
```

Note that a detached thread must still exit in some manner, but it can do so itself by calling `pthread_exit`:

```c
void pthread_exit(void * status);
```

An example snipped showing detachable threads is shown below:

```c
#include <stdio.h>
#include <pthread.h>

void * foo(void * arg) {
    printf("Foobar\n");
    pthread_exit(NULL);
}

int main() {
    int i;
    pthread_t tid;

    pthread_attr_t attr;
    pthread_attr_init(&attr); /* REQUIRED! */
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);
    pthread_create(&tid, &attr, foo, NULL);

    return 0;
}
```

- Notice that `pthread_attr_init` is required. This will allocate sufficient memory as well as initialise default values within `attr`.

## PThread Compilation

When dealing with the `pthread` library, here are a some things to pay attention to:

1. Remember to actually include the header file with `#include <pthread.h>`. 

2. Compile the source code with the `-lpthread` or `-pthread` flag. The `-pthread` flag tells the compiler to link the `pthread` library **as well as configure the compilation for threads**.
   
   ```bash
   $ gcc -o main main.c -lpthread
   $ gcc -o main main.c -pthread
   ```
   
   > Wherever possible use `-pthread` instead. 

3. Check the return values of common functions (e.g., when creating a thread or joining a thread). This will immediately inform the success of a particular operation, which will prove useful when dealing with multithreaded programs.

## Intermission: Quizzes

_Quiz: How many times will "Hello Thread" be printed?_

```c
#include <stdio.h>
#include <pthread.h>
#define NUM_THREADS 4;


void * hello(void * arg) {
    printf("Hello Thread\n");
    return 0;
}

int main() {
    int i;
    pthread_t tid[NUM_THREADS]; // same as an int array
    for (i = 0; i < NUM_THREADS; i++) { /* create/fork threads */
        pthread_create(&tid[i], NULL, hello, NULL);
    }
    for (i = 0; i < NUM_THREADS; i++) {
        pthread_join(tid[i], NULL);
    }
    return 0;
}
```

Four times, the number of threads created.

_Quiz: Select the possible outputs of the program below:_

```c
#include <stdio.h>
#include <pthread.h>
#define NUM_THREADS 4;


void * threadFunc(void * pArg) {
    int * p = (int*)pArg;
    int myNum = *p;
    printf("Thread number %d\n", myNum); 
    return 0;
}

int main() {
    int i;
    pthread_t tid[NUM_THREADS]; // same as an int array
    for (i = 0; i < NUM_THREADS; i++) { /* create/fork threads */
        pthread_create(&tid[i], NULL, threadFunc, &i);
    }
    for (i = 0; i < NUM_THREADS; i++) {
        pthread_join(tid[i], NULL);
    }
    return 0;
}
```

```
A) // POSSIBLE
Thread number 0
Thread number 1
Thread number 2
Thread number 3

B) // POSSIBLE
Thread number 0
Thread number 2
Thread number 1
Thread number 3

C) // POSSIBLE
Thread number 0
Thread number 2
Thread number 2
Thread number 3
```

Why is (C) a possible option?

- The variable `i` that is being passed to the threads is defined in `main`; it is a globally visible variable. Therefore, a change in one thread will also be visible in the other threads.

- Suppose we trace the execution of the above code, up to the point where a thread is called with the address of `i` (suppose `i` is set to the value of `1`). We would expect this thread to therefore print `Thread number 1`. 

- Before it could do so, it may be the case that main thread moved into the next iteration of the `for` loop, incrementing `i` from `1` to `2`. As such, the thread now sees the value at the address of `i` as `2`, and prints `Thread number 2` instead. 

To solve the above problem, it is possible to declare an array, where each element would be the "private storage" of threads of the corresponding index:

_Quiz: Select the possible outputs of the program below:_

```c
#include <stdio.h>
#include <pthread.h>
#define NUM_THREADS 4;


void * threadFunc(void * pArg) {
    int * p = (int*)pArg;
    int myNum = *p;
    printf("Thread number %d\n", myNum); 
    return 0;
}

int main() {
    int tNum[NUM_THREADS];
    int i;
    pthread_t tid[NUM_THREADS];
    for (i = 0; i < NUM_THREADS; i++) {
        pthread_create(&tid[i], NULL, threadFunc, &tNum[i]);
    }
    for (i = 0; i < NUM_THREADS; i++) {
        pthread_join(tid[i], NULL);
    }
    return 0;
}
```

```
A)
Thread number 0
Thread number 0
Thread number 2
Thread number 3

B) // POSSIBLE
Thread number 0
Thread number 2
Thread number 1
Thread number 3

C) // POSSIBLE
Thread number 3
Thread number 2
Thread number 1
Thread number 0
```
## PThread Mutexes

Recall that mutexes exist to solve mutual exclusion problems among concurrent threads. Mutual exclusion ensures that threads access shared state in a controlled manner, such that only one thread can modify/access the shared state at any given time.

Again, there must be a type associated with a mutex:

```c
pthread_mutex_t aMutex;
```

### Lock

The lock and unlock operation must be explicitly called within `pthread`:

```c
int pthread_mutex_lock(pthread_mutex_t * mutex); /* lock */
int pthread_mutex_unlock(pthread_mutex_t * mutex); /* unlock */
```

- Any code that appears between these two explicit lock/unlock statements will be considered the **critical section**.

### Other Mutex Operations

Here are other commonly used mutex operations:

```c
int pthread_mutex_init(
        pthread_mutex_t * mutex, 
        const pthread_mutexattr_t * attr);
```

- Mutex attributes specify mutex behaviour. Similar to thread attributes, set this to `NULL` for default behaviour. For example, it is possible to allow the sharing of mutexes between processes, when the default behaviour would be that a mutex is only available *to threads within a singular process*.

```c
int pthread_mutex_trylock(pthread_mutex_t * mutex);
```

- Trylock is similar to a regular mutex lock, though this function returns immediately if the mutex cannot currently be locked. In other words, this function does not block and wait for mutex acquisition. 

```c
int pthread_mutex_destroy(pthread_mutex_t * mutex);
```

- Of course, memory allocated must be freed.
### Mutex Safety Tips

1. Shared data should always be accessed through a single mutex.

2. Mutex scope must be visible to all threads. In other words, a mutex should not be defined in a function (including `main`). Mutexes must be defined as global variables.

3. Establish a global order for locking mutexes and enforce this order for all threads. Remember that this is one foolproof method of preventing deadlocks.

4. Remember to unlock the **correct** mutex. Remember to just generally unlock mutexes after locking them as well.

## PThread Condition Variables

You know the drill. Condition variable type:

```c
pthread_cond_t aCond;
```

### Wait

```c
int pthread_cond_wait(pthread_cond_t * cond, pthread_mutex_t * mutex);
```

- This is exactly the same as the theoretical construct seen above. 

- If a thread must wait, it will automatically release the mutex and place itself on the wait queue associated with the condition variable.

- Upon waking up, the mutex will be automatically acquired before exiting the wait operation.

### Signal and Broadcast

```c
int pthread_cond_signal(pthread_cond_t * cond);

int pthread_cond_broadcast(pthread_cond_t * cond);
```

- Recall that signal wakes up one thread waiting on the condition variable, while broadcast wakes up all waiting threads (on that condition variable).

### Other Condition Variable Operations

Again, init functions and destroy functions must be called:

```c
int pthread_cond_init(
        pthread_cond_t * cond,
        const pthread_condattr_t * attr);

int pthread_cond_destroy(pthread_cond_t * cond); 
```

- Like what we saw with mutexes, attributes for condition variables can be specified. `NULL` once again specifies default behaviour. 

### Condition Variable Safety Tips

1. Do not forget to notify waiting threads. Whenever a predicate changes, signal or broadcast the **correct** condition variable that these threads are waiting on.

2. When in doubt, broadcast. There will be a performance loss but its better than an incorrect program.

3. Since mutexes are not required for signal/broadcast, consider releasing the mutex before signalling or broadcasting. Of course, this depends on the situation (as seen in the previous section).

## PThreads Usage Example

To tie everything together, we will look at the classic producer/consumer scenario using the `pthread` library.
### Global Declarations

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define BUF_SIZE 3

int buffer[BUF_SIZE]; /* shared buffer */
int add = 0; /* place to add next element */
int rem = 0; /* place to remove next element */
int num = 0; /* number of elements in buffer */

/* mutex lock for buffer */
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
/* consumer waits on cv */
pthread_cond_t c_cons = PTHREAD_COND_INITIALIZER;
/* producer waits on cv */
pthread_cond_t c_prod = PTHREAD_COND_INITIALIZER;

void * producer(void * param);
void * consumer(void * param);
```

- The `PTHREAD_MUTEX_INITIALIZER` is a macro that will automatically initialise the mutex for us. The same is true for `PTHREAD_COND_INITIALIZER`.
### Main

```c
int main(int argc, char * argv[]) {
    pthread_t tid1, tid2; /* thread identifiers */
    int i;

    if (pthread_create(&tid1, NULL, producer, NULL) != 0) {
        fprintf(stderr, "Unable to create producer thread\n");
        exit (1);
    }
    if (pthread_create(&tid2, NULL, consumer, NULL) != 0) {
        fprintf(stderr, "Unable to create consumer thread\n");
        exit(1);    
    }

    pthread_join(tid1, NULL); /* wait for producer */
    pthread_join(tid2, NULL); /* wait for consumer */
    printf("Parent quiting.\n");
    return 0;
}
```
### Producer

```c
void * producer(void * param) {
    int i;
    for (i = 1; i <= 20; i++) {
        pthread_mutex_lock(&m);

        if (num > BUF_SIZE) { exit(1); } /* error check overflow */
        while (num == BUF_SIZE) {
            pthread_cond_wait(&c_prod, &m);
        }
        buffer[add] = i;
        add = (add + 1) % BUF_SIZE; /* wrap around buffer index */
        num++;
        
        pthread_mutex_unlock(&m);
        pthread_cond_signal(&c_cons);
        printf("Producer: inserted %d\n", i); fflush (stdout);
    }
    printf("Producer quitting\n"); fflush (stdout);
    return 0;
}
```

- The producer loops twenty times, and tries to produce and element on the shared buffer per iteration.

- It first tries to acquire `m` with `pthread_mutex_lock`. Once `m` is acquired, it will potentially wait on `c_prod` with `pthread_cond_wait` if the shared buffer is full (`num == BUFSIZE`). 

- Otherwise, the producer will add its element to the buffer, and update the shared variables `add` and `num` to reflect this change.

- Once the producer unlocks the mutex `m`, it signals on `c_cons` with `pthread_cond_signal` to let consumers know that there is data to consume.

- Since only one element is added, it makes more sense to use a signal (notify one waiting consumer) instead of a broadcast (notify all waiting consumers).
### Consumer

```c
void * consumer (void * param) {
    int i;
    while (1) {
        pthread_mutex_lock(&m);
        
        if (num < 0) { exit(1); } /* error check underflow */
        while (num == 0) {
            pthread_cond_wait(&c_cons, &m);
        }
        i = buffer[rem];
        rem = (rem + 1) % BUF_SIZE /* wrap around buffer index */
        num--;

        pthread_mutex_unlock(&m);
        pthread_cond_signal(&c_prod);
        printf("Consumed value %d\n", i); fflush (stdout);
    }
}
```

- The consumer loops forever, attempting to remove elements from the shared buffer.

- It first tries to acquire `m` with `pthread_mutex_lock`. Once `m` is acquired, the consumer will potentially wait on `c_cons` with `pthread_cond_wait` if the shared buffer is empty (`num == 0`).

- Otherwise, the consumer will remove an element from the buffer, and update variables `rem` and `num` to reflect this change.

- Once the consumer unlocks the mutex, it signals on `c_prod` with `pthread_cond_signal` to let producers know that there is space in the buffer to add more data.
