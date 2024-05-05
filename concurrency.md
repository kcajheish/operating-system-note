## Concurrency

Thread
- Multi thread program has multiple PC(program counter)
- Threads of a process share the same address space
- Context is switched when CPU stops thread A from running and start thread B. Use thread control block(TCB) to save/restore state.
- In a multi-threaded process, each thread has its own stack(namely, **thread local storage**) in address space.

Why thread
- Parallelism
    - Turn a single threaded program into multiple thread when multiple CPUs are available.
    - Tasks are processed in shorter time
- Avoid blocking due to slow IO
    - Thread enables overlap of IO with other activities within program

Thread vs Process
- Multi-threaded program and multiprogramming both speed thing up.
- Thread is useful when you like to share data among them.
- Process is preferred when tasks are independent of each other.

Why concurrency is a bad thing
- When two threads try to update same shared variable, the result is not deterministic

Uncontrolled scheduling
- e.g. Before thread A updates value in memory, timer interrupts the thread A, save the state of thread A in TCB, and then restore thread B from TCB.
- Race condition(data race)
    - Result depends on timing of interrupt
    - Nondeterministic
- Critical section
    - It’s code that accesses shared variable and thus must not be executed concurrently
- Mutual exclusion
    - To ensure one thread can enter critical section

Tool you should know
- gdb, a debugger
- valgrind/purify, memory profiler
- objdump, turn executable into assembler

Atomic
- All actions succeed or none of them works. There is no in-between state
- Two scenarios require atomicity
    - Multiple threads update a shared variable
    - One thread performs IO task to load data from disk to memory; another waits until IO thread completes its task

Transaction
- Group a sequence of actions to ensure atomic result

How does program ask hardware for atomicity in critical section?
- Synchronization primitives

# Lock

Declare a variable with lock, put it around critical section to ensure atomicity.
- Lock has two states: aquired or available
- Other states are possible, but they are hidden away from client.
- To acquired a lock, call lock()
    - if lock is already acquired, lock() won't return
- To release a lock, call unlock()

Why do we need lock after all?
- Although thread is created by programmer, it's scheduled by OS. The result of the program is non-deterministics. With lock, programmer ensures atomicity and thus gain control back.

Coarse grain lock
- use a global variable as lock
- con
    - low concurrency as many threads are waiting in lines for a single lock
- pro
    - easy to implement for programmer

Fine grain lock
- use different variable for different critical section
- pro
    - high concurrency
- con
    - complicated

what makes a lock successful?
- Mutual exclusion
    - Only one thread enters critical section.
- Faireness
    - Each contending thread has the same chance for acquiring the lock
- Performance
    - Overhead of acquire a lock

Control interrupt
- Disable interrupt when thread enters critical section
    - pro
        - User program can’t be trusted
            - An infinite loop is in critical section and CPU never gain control back.
            - User intentionally disables interrupt for infinite amount of time
        - Can’t work on multi-core
            - Interrupt only disable one core of CPUs
    - con
        - Easy to implement
        - OS can use it to update internal data structure because os is safe.

Load and store
- issue
    - low performance: spin wait waste a lot of time
    - timely interrupt makes both threads acquire the lock(set flag to one)
        - thread_a enters line 9
        - CPU interrupts and thread_b starts
        - thread b enter line 9, too
- code
```
1 typedef struct __lock_t { int flag; } lock_t;
2
3 void init(lock_t *mutex) {
4   // 0 -> lock is available, 1 -> held
5   mutex->flag = 0;
6 }
7
8 void lock(lock_t *mutex) {
9   while (mutex->flag == 1) // TEST the flag
10     ; // spin-wait (do nothing)
11   mutex->flag = 1; // now SET it!
12 }
13
14 void unlock(lock_t *mutex) {
15   mutex->flag = 0;
16 }
```

Test and set
- hardware makes test and set atomic
- for a single processor, need preemptive scheduling
    - otherwise, CPU never interrupts a thread when it spins
- An old flag is returned. If the flag is already set to one, lock is acquired by a different thread and current thread spin
- code
```
1 int TestAndSet(int *old_ptr, int new) {
2   int old = *old_ptr; // fetch old value at old_ptr
3   *old_ptr = new; // store ’new’ into old_ptr
4   return old; // return the old value
5 }
```

evaluate spin lock
- performance
    - It’s bad for a single processor. Clock cycles of a single are wasted when a thread spin.
    - It’s good for multiple processor. One thread spins until another thread release the lock.
- fairness
    - No. Scheduling is controlled by CPU. We have no control over next thread that takes over the lock
- correctness
- yes, only one thread can enter critical section


Compare and swap
- acquire the lock when the lock status is expected
- the original status of the lock is returned no matter what
- code
```
1 void lock(lock_t *lock) {
2   while (CompareAndSwap(&lock->flag, 0, 1) == 1)
3     ; // spin
4 }

1 int CompareAndSwap(int *ptr, int expected, int new) {
2   int original = *ptr;
3   if (original == expected)
4     *ptr = new;
5   return original;
6 }
```

load link and store conditional
- idea
    - read the status of the lock; spinning when lock is already acquired. Then, return 1 if you can acquire lock, else return 0.
    - Only one thread can success in store conditional
- code
```
1 int LoadLinked(int *ptr) {
2   return *ptr;
3 }
4
5 int StoreConditional(int *ptr, int value) {
6   if (no update to *ptr since LL to this addr) {
7     *ptr = value;
8     return 1; // success!
9    } else {
10    return 0; // failed to update
11   }
12 }

1 void lock(lock_t *lock) {
2   while (1) {
3     while (LoadLinked(&lock->flag) == 1)
4       ; // spin until it’s zero
5     if (StoreConditional(&lock->flag, 1) == 1)
6       return; // if set-to-1 was success: done
7     // otherwise: try again
8 }
9 }
10
11 void unlock(lock_t *lock) {
12   lock->flag = 0;
13 }
```

fetch and add
- used to create a ticket lock
- thread acquires lock when it’s turn of thread.
- It’s fair because each thread is assigned a turn.
- code
```
// we need hardward support to make FetchAndAdd atomic.
1 int FetchAndAdd(int *ptr) {
2   int old = *ptr;
3   *ptr = old + 1;
4   return old;
5 }

1 typedef struct __lock_t {
2   int ticket;
3   int turn;
4 } lock_t;
5
6 void lock_init(lock_t *lock) {
7   lock->ticket = 0;
8   lock->turn = 0;
9 }
10
11 void lock(lock_t *lock) {
12   int myturn = FetchAndAdd(&lock->ticket);
13   while (lock->turn != myturn)
14     ; // spin
15 }
16
17 void unlock(lock_t *lock) {
18   lock->turn = lock->turn + 1;
19 }
```

spin too much
- for uniprocessor, spinning wastes a lot of time because thread are not making any progress during spinning.


How to avoid wasteful spin?
- Just yield the CPU to another thread

Yield
- Yield moves a thread from running to ready state. Essentially, the thread deschedules itself.
- drawback: When there are 100 threads contend for one lock, we need 99 test-and-set then yield. This is costly due to context switch.

Sleeping instead of spinning
- To avoid too much context switch, use Queue to keep track of thread that should be run next.
- A thread acquires guard lock and hence put manipulation of flag and park in a critical section.
- When a lock is already acquired, put current thread into queue and sleep.
- When a lock is released, wake up a thread in the head of the queue.
- waiting race
    - A lock is released before thread is parked. That thread in the queue may never be waken up.
- code
```
1 typedef struct __lock_t {
2   int flag;
3   int guard;
4   queue_t *q;
5 } lock_t;
6
7 void lock_init(lock_t *m) {
8   m->flag = 0;
9   m->guard = 0;
10   queue_init(m->q);
11 }
12
13 void lock(lock_t *m) {
14   while (TestAndSet(&m->guard, 1) == 1)
15     ; //acquire guard lock by spinning
16   if (m->flag == 0) {
17     m->flag = 1; // lock is acquired
18     m->guard = 0;
19   } else {
20     queue_add(m->q, gettid());
21     m->guard = 0;
22     park();
23   }
24 }
25
26 void unlock(lock_t *m) {
27   while (TestAndSet(&m->guard, 1) == 1)
28     ; //acquire guard lock by spinning
29   if (queue_empty(m->q))
30     m->flag = 0; // let go of lock; no one wants it
31   else
32     unpark(queue_remove(m->q)); // hold lock
33     // (for next thread!)
34     m->guard = 0;
35 }
```

Linux example
- linux use futex to wakeup and sleep a thread.
- A futex holds 32 bits
    - the highest bit implies whether lock is acquired or not
    - The rest of bits indicates number of sleeping thread
    - When the 32 bits value > 0, lock is not acquired
- note that In `unlock`, `0x80000000` hex is binary: `10 00000 00000 00000 00000 00000 00000` (31 zeros).
- code
```
1 void mutex_lock (int *mutex) {
2   int v;
3   // Bit 31 was clear, we got the mutex (fastpath)
4   if (atomic_bit_test_set (mutex, 31) == 0)
5     return;
6   atomic_increment (mutex);
7   while (1) {
8     if (atomic_bit_test_set (mutex, 31) == 0) {
9      atomic_decrement (mutex);
10     return;
11    }
12    // Have to waitFirst to make sure futex value
13    // we are monitoring is negative (locked).
14    v = *mutex;
15    if (v >= 0)
16      continue;
17    futex_wait (mutex, v);
18   }
19 }
20
21 void mutex_unlock (int *mutex) {
22   // Adding 0x80000000 to counter results in 0 if and
23   // only if there are not other interested threads
24   if (atomic_add_zero (mutex, 0x80000000))
25     return;
26
27   // There are other threads waiting for this mutex,
28   // wake one of them up.
29   futex_wake (mutex);
30 }
```

Two phase lock
- Spin lock is efficient when lock is soon to be release.
- Lock in linux can be acquired in two phase
    - spin for a fix amount of time; wait for release of the lock
    - stop the spin and put current thread into sleep

## Event based concurrency

What is event loop
- Events are gathered and are executed in order using event handler.
    - By execute in order, application has the control over scheduling.
- Code
```
while (1) {
	events = getEvents();
	for (e in events)
		processEvent(e);
}
```

No lock is required
- A thread execute the handler that processes one event at a time. Because thread is not interrupted, concurrency bugs are avoided

Potential issue: main thread waits for IO to complete.
- Client reads file from disk
    - open/read system call are made
    - Only one thread is running in the server. So server just waits and doesn’t make any progress
- Note: it’s not the case for threaded concurrency because IO overlaps with other activities

Solution: Asynchronous IO
- Call of async IO returns right away and thread continue making progress
- In Mac
    - Use AIO control block for IO request and aio_read for issuing request
    - To know the status of IO request, call aio_error
    - If there are thousands of IO requests, calling aio_error for each request can be slow.
        - Use interrupt to avoid checking too much
            - When IO request is completed, UNIX sends a signal to application. Thread stop what it's doing and execute signal handler if that signal can be recognized.
```
struct aiocb {
	int aio_fildes; // File descriptor
	off_t aio_offset; // File offset
	volatile void *aio_buf; // Location of buffer
	size_t aio_nbytes; // Length of transfer
};

int aio_read(struct aiocb *aiocbp);

int aio_error(const struct aiocb *aiocbp);

void handle(int arg) {
    printf("stop wakin’ me up...\n");
}

int main(int argc, char *argv[]) {
    signal(SIGHUP, handle);
    while (1)
        ; // doin’ nothin’ except catchin’ some sigs
    return 0;
}
```

Con of event loop
- Still need locks when multi CPUs are used
- Can’t integrate well with paging
    - When page fault becomes prevalent, server performance drops
- Hard to maintain
    - Routine can be blocking/non-blocking. They need different styles of coding
- Async IO requires both select() and AIO for network IO and disk IO

## Conditional variable
To build concurrent program, we need
1. lock
2. conditional variable

with conditional variable, parent thread check status of the child thread before continuing.

How?
- parent thread spin and wait for condition to hold true
    - con: parent thread wastes a lot of cpu cycles.
- put the parent thread to sleep; wake up the parent thread after the child thread exits and returns.

```
1 int done = 0;
2 pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
3 pthread_cond_t c = PTHREAD_COND_INITIALIZER;
4
5 void thr_exit() {
6   Pthread_mutex_lock(&m);
7   done = 1;
8   Pthread_cond_signal(&c);
9   Pthread_mutex_unlock(&m);
10 }
11
12 void *child(void *arg) {
13  printf("child\n");
14  thr_exit();
15  return NULL;
16 }
17
18 void thr_join() {
19  Pthread_mutex_lock(&m);
20  while (done == 0)
21  Pthread_cond_wait(&c, &m);
22  Pthread_mutex_unlock(&m);
23 }
24
25 int main(int argc, char *argv[]) {
26  printf("parent: begin\n");
27  pthread_t p;
28  Pthread_create(&p, NULL, child, NULL);
29  thr_join();
30  printf("parent: end\n");
31  return 0;
32 }
```
Why acquire lock before put parent thread to sleep(see line 19)?
- Avoid race. Before parent thread is put to sleep queue, timer interrupts parent process. Child thread runs to complete and signal the sleeping queue but there are none. Later, parent thread is put to queue and stuck in there forever.

Why is done variable needed? Why not parent thread waits for signal?
- If the child thread sent a signal before parent thread is put to sleep, then parent stucks in the queue forever. Thus, we need done variable to determine to skip wait. See below as a bad example.
```
1 void thr_exit() {
2   Pthread_mutex_lock(&m);
3   Pthread_cond_signal(&c);
4   Pthread_mutex_unlock(&m);
5 }
6
7 void thr_join() {
8   Pthread_mutex_lock(&m);
9   Pthread_cond_wait(&c, &m);
10  Pthread_mutex_unlock(&m);
11 }
```
why is lock acquired before wait and signal?
```
1 void thr_exit() {
2   done = 1;
3   Pthread_cond_signal(&c);
4 }
5
6 void thr_join() {
7   if (done == 0)
8   Pthread_cond_wait(&c);
9 }
```


Producer/Consumer(Bound/Buffer)
- Consider a buffer with size 1. Producer thread puts data in the buffer when buffer is empty. Consumer thread get data from the buffer when buffer is full.
- Mesa semantics
    - When a thread is woken up, it is first put to ready state.
    - Although condition is met by the time the thread is woken up, that condition may not hold true when the thread is running.
    - This happends because there are more than one producer/consumer thread
- Hoarse semantics
    - ensure the thread is running after the thread is woken up.
- below is code
    - why two condition variable is needed?
        - Producer threads wakes up producer threads, and vice versa.
            - Imagine a consumer thread is put to sleep; a consumer thread and a producer thread is in the queue. If the consumer thread is woken, it put to sleep because buffer is still empty. Now, all threads stuck in queue.
    - why is while loop used over if statement?
        - after the thread is woken up, the condition may change before that thread is running. Thus, use a while loop to check the condition again.
    - why is lock used?
        - To avoid race
            - two consumer threads compete for the same data in the buffer.
```
1 int buffer[MAX];
2 int fill_ptr = 0;
3 int use_ptr = 0;
4 int count = 0;
5
6 void put(int value) {
7   buffer[fill_ptr] = value;
8   fill_ptr = (fill_ptr + 1) % MAX;
9   count++;
10 }
11
12 int get() {
13  int tmp = buffer[use_ptr];
14  use_ptr = (use_ptr + 1) % MAX;
15  count--;
16  return tmp;
17 }
```
```
1 int loops; // must initialize somewhere...
2 cond_t empty, fill;
3 mutex_t mutex;
4
5 void *producer(void *arg) {
6   int i;
7   for (i = 0; i < loops; i++) {
8       Pthread_mutex_lock(&mutex); // p1
9       while (count == MAX) // p2
10          Pthread_cond_wait(&empty, &mutex); // p3
11      put(i); // p4
12      Pthread_cond_signal(&fill); // p5
13      Pthread_mutex_unlock(&mutex); // p6
14  }
15 }
16
17 void *consumer(void *arg) {
18  int i;
19  for (i = 0; i < loops; i++) {
20      Pthread_mutex_lock(&mutex); // c1
21      while (count == 0) // c2
22          Pthread_cond_wait(&fill, &mutex); // c3
23      int tmp = get(); // c4
24      Pthread_cond_signal(&empty); // c5
25      Pthread_mutex_unlock(&mutex); // c6
26      printf("%d\n", tmp);
27  }
28 }
```
## Common concurrency problem

Watch out for concurrency bugs so we can write robust code.

Atomicity violation
- We assumes memory access to the certain value is serializable. That causes subtle bugs sometime since interrupt can happens anytime.
```
Thread 1::
if (thd->proc_info) {
	fputs(thd->proc_info, ...);
}

Thread 2::
thd->proc_info = NULL;
```
- The solution is quite simple. Acquire lock before accessing those variables.
```
pthread_mutex_t proc_info_lock = PTHREAD_MUTEX_INITIALIZER;
Thread 1::
pthread_mutex_lock(&proc_info_lock);
if (thd->proc_info) {
	fputs(thd->proc_info, ...);
}
pthread_mutex_unlock(&proc_info_lock);

Thread 2::
pthread_mutex_lock(&proc_info_lock);
thd->proc_info = NULL;
pthread_mutex_unlock(&proc_info_lock);
```
- Dereference: access value in a memory address

Ordering violation
- We want to execute threads in a certain order. Use conditional variable to ensure order.
- e.g. mThread in Thread1 should be created before Thread2 is executed and accessing state in mThread
```
Thread 1::
void init() {
	mThread = PR_CreateThread(mMain, ...);
}

Thread 2::
void mMain(...) {
	mState = mThread->State;
}
```
- sol:
```
pthread_mutex_t mtLock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t mtCond = PTHREAD_COND_INITIALIZER;
int mtInit = 0;
Thread 1::
void init() {
	mThread = PR_CreateThread(mMain, ...);
	// signal that the thread has been created...
	pthread_mutex_lock(&mtLock);
	mtInit = 1;
	pthread_cond_signal(&mtCond);
	pthread_mutex_unlock(&mtLock);
}
Thread 2::
void mMain(...) {
	// wait for the thread to be initialized...
	pthread_mutex_lock(&mtLock);
	while (mtInit == 0)
		pthread_cond_wait(&mtCond, &mtLock);
	pthread_mutex_unlock(&mtLock);
	mState = mThread->State;
}
```

Deadlock
- Circular dependency where each thread is waiting for another thread to release the lock
- Why deadlock happens easily
    - System is complex and deadlock
    - Interface hides detail of lock
```
Vector v1, v2;
v1.AddAll(v2);
```

To prevent deadlock, consider the following:
- Mutual exclusion
- Hold and waits
    - Acquire all locks at once
    - Con
        - Programmer has to know order of lock
        - Less concurrency
- No preemption
    - If we can’t acquire the lock, then release the lock
    - However, livelock is introduced.
- Circular waits
    - Introduce lock ordering
        - e.g. acquire L1 before acquiring L2. One trick is to use Lock whose address is low.
        - Can be full and partial(multiple groups)
        - Con
            - A complicate lock protocol that programmer adheres to

Dead lock avoidance
- Schedule lock in a way that deadlock can’t happen
- Con
    - Processing time takes longer

Detect and recover
- If deadlock doesn’t happen a lot. It’s pragmatic to detect it when it happens and reboots the system

