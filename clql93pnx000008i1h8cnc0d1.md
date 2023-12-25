---
title: "Rust Learning Note: Thread Sychronization"
datePublished: Mon Dec 25 2023 18:27:46 GMT+0000 (Coordinated Universal Time)
cuid: clql93pnx000008i1h8cnc0d1
slug: rust-learning-note-thread-sychronization
tags: multithreading, rust, threads, lock, mutex, deadlock, mpsc, atomic-type

---

this article is a summary of Chapter 4.6.3 - 4.6.6 of Rust Course ([course.rs/](https://course.rs/))

### Multiple Producer Single Consumer

The Rust standard library provides a channel among multiple threads called **std::sync::mpsc.** mpsc is short for multiple producer, single consumer, meaning that the channel supports multiple senders but only one receive. Here is a simple example use of mpsc:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(1).unwrap();
    });

    println!("receive {}", rx.recv().unwrap());
}
```

In the code above, **mpsc::channel()** creates a new channel, which returns a tuple containing the sender (tx), and receiver (rx). The spawned thread sends 1 with **tx**, which is received by the main thread with **rx**. We need to use move to transfer the ownership of tx into the closure. The **send** method returns a **Result&lt;T, E&gt;**, meaning that it may send the error if an error occurs.

**try\_recv method**

Method **recv()** would block the thread until it receives the message. Another method called try\_recv would try to read the message, returns an error if the message has not been sent, and proceed without blocking the thread.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(1).unwrap();
    });

    println!("receive {}", rx.try_recv().unwrap());
}
```

If we replace the rx.recv() previous code with rx.try\_recv(), we would likely get an Err(Empty) since the creation of a new thread is much slower than the processing of the main thread.

When sending a variable that has ownership (does not implement Copy), send() would transfer the ownership of the variable from the sender to the receiver.

**Multiple Senders**

To have multiple senders of a channel, we need to clone the tx in the channel and assign the copy of tx to the other thread.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let tx1 = tx.clone();
    thread::spawn(move || {
        for i in 0..10 {
            tx.send(i).unwrap();
        }
    });
    
    thread::spawn(move || {
        for i in 0..10 {
            tx1.send(i).unwrap();
        }
    });

    for received in rx {
        println!("Got: {received}");
    }
}
```

**Asychronized and Sychronized Channel**

The mpsc::channel we used above is asychronized, meaning that the sender continues executing the following codes after sending the message, regardless of whether the message is received. Using mpsc::sync\_channel we create a sychronized channel, in which the sender is blocked until the receiver receives the message.

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::sync_channel(0);

    let handle = thread::spawn(move || {
        println!("before sending");
        tx.send(1).unwrap();
        println!("after sending");
    });

    println!("before sleeping");
    thread::sleep(Duration::from_secs(3));
    println!("after sleeping");

    println!("receive {}", rx.recv().unwrap());
    handle.join().unwrap();
}
```

In this code, "after sending" by the spawned thread will be printed only after "after sleeping" in the main thread, indicating that the spawned thread is blocked when the main thread does not receive the message. The parameter inside mpsc::sync\_channel is the size of the cache. When the cache is set to N, the sender can send at most N messages to the receiver without being blocked, and is blocked when the cache is full.

### Lock

Besides the use of message system, we can also sychronize threads with shared memory. In this case, we need to assign locks to the data accessed by a thread to prevent data competing.

**Mutex**

**Mutex** stands for mutual exclusion. To access data in Mutex, the thread need to request a lock with lock(), which blocks the thread if the lock is acquired by other threads. This allows multiple threads to access the data one by one. Since Mutex is also a smart pointer, it implements Deref, allowing automatic dereferencing to retrieve the data inside, and Drop, allowing the lock to be released once it leaves the scope.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

In the code above, we use Arc to allow multiple ownership of data, and Mutex to achieve internal mutability. In single thread, Rc&lt;T&gt;/RefCell&lt;T&gt; is a common combination to achieve internal mutability and multiple ownership. Its multithreading version is Arc&lt;T&gt;/Mutex&lt;T&gt;.

**Deadlock**

**Deadlock** would happen if a lock is being requested when it is not released yet. Here is a single example in one thread:

```rust
use std::sync::Mutex;

fn main() {
    let data = Mutex::new(0);
    let d1 = data.lock();
    let d2 = data.lock();
}
```

In multithreading, if two threads each acquires a lock and tries to acquire the lock owned by the other thread, deadlock may happen:

```rust
use std::{sync::{Mutex, MutexGuard}, thread};
use std::thread::sleep;
use std::time::Duration;

use lazy_static::lazy_static;
lazy_static! {
    static ref MUTEX1: Mutex<i64> = Mutex::new(0);
    static ref MUTEX2: Mutex<i64> = Mutex::new(0);
}

fn main() {
    let mut children = vec![];
    for i_thread in 0..1 {
        children.push(thread::spawn(move || {
            if i_thread % 2 == 0 {
                let guard1: MutexGuard<i64> = MUTEX1.lock().unwrap();
                println!("MUTEX1 locked by {i_thread}");
                let guard1 = MUTEX2.lock().unwrap();
            } else {
                let guard2: MutexGuard<i64> = MUTEX2.lock().unwrap();
                println!("MUTEX2 locked by {i_thread}");
                let guard2 = MUTEX1.lock().unwrap();
            }
        }));
    }

    for child in children {
        let _ = child.join();
    }
}
```

In this code, thread 1 locks MUTEX1 and tries to acquire MUTEX2, and thread 2 locks MUTEX2 and tries to acquire MUTEX1. Both threads will then be blocked indefinitely.

**try\_lock**

In contrast with **lock**, **try\_lock** tries to acquire the lock once without blocking the thread, and returns an error if the lock has been acquired.

**RwLock**

Mutex would assign a lock for both reading and writing operations, which could lead to inefficiency when we have large number of reading operations from multiple threads. RwLock does not set locks for reading operations, and only blocks other threads when writing operations occur. In short, RwLock allows either reading operations from multiple threads or writing operations from one thread at a time.

```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);

    {
        // multiple readings allowed
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        assert_eq!(*r1, 5);
    }

    {
        // only one writing allowed
        let mut w = lock.write().unwrap();
        *w += 1;
        assert_eq!(*w, 6);    
    }
}
```

In general, Mutex is more efficient in its operations. We should use RwLock only when we want multiple threads to read the data simultaneouly and possibly acquire the data for a long time.

### Condition Variable

Condition variable can suspend a thread and continue it only after certain contations. It can be used to control the order of data access of multiple threads.

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread::{spawn, sleep};
use std::time::Duration;

fn main() {
    let flag = Arc::new(Mutex::new(false));
    let cond = Arc::new(Condvar::new());
    let cflag = flag.clone();
    let ccond = cond.clone();

    let hdl = spawn(move || {
        let mut lock = cflag.lock().unwrap();
        let mut counter = 0;

        while counter < 3 {
            while !*lock {
                lock = ccond.wait(lock).unwrap();
            }

            *lock = false;
            counter += 1;
            println!("inner counter: {counter}");
        }
    });

    let mut counter = 0;
    loop {
        sleep(Duration::from_millis(1000));
        *flag.lock().unwrap() = true;
        counter += 1;
        if counter > 3 {
            break;
        }
        println!("outer counter: {counter}");
        cond.notify_one();
    }
    hdl.join().unwrap();
    println!("{:?}", flag);
}
```

In this case, if the thread hdl reaches the while loop before the main thread sets the flag to true, it would be blocked by wait() until the main thread invokes notify\_one(). The result will be alternative printings from the main thread and hdl triggered by the main thread.

> outer counter: 1
> 
> inner counter: 1
> 
> outer counter: 2
> 
> inner counter: 2
> 
> outer counter: 3
> 
> inner counter: 3
> 
> Mutex { data: true, poisoned: false, .. }

### Semaphore

Semaphore allows us to control the maximum tasks in the program. The code below creates a semaphore with capacity 3. Where are are more than 3 tasks, the rest of them need to wait after the tasks being executed.

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3));
    let mut join_handles = Vec::new();
    
    for _ in 0..5 {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        join_handles.push(tokio::spawn(async move {
            // process the task
            drop(permit);
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

### Atomic Type

The use of atomic type is another way to sychronize among multiple threads. An atomic operation refers to an indivisible operation on shared memory, ensuing that no other operation can happen simultaneously. One typical application of atomic type is the creation of gobal variables:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread::{self, JoinHandle};

const N_TIMES: u64 = 1000000;
const N_THREADS: usize = 10;

static R: AtomicU64 = AtomicU64::new(0);

fn add_n_times(n: u64) -> JoinHandle<()> {
    thread::spawn(move || {
        for _ in 0..n {
            R.fetch_add(1, Ordering::Relaxed);
        }
    })
}

fn main() {
    let mut threads = Vec::with_capacity(N_THREADS);
    
    for _ in 0..N_THREADS {
        threads.push(add_n_times(N_TIMES));
    }

    for thread in threads {
        thread.join().unwrap();
    }
    assert_eq!(N_TIMES * N_THREADS as u64, R.load(Ordering::Relaxed));
}
```

In the code above, we have 10 threads adding 1 continuously to atomic type R. The final result of R is equal to N\_TIMES \* N\_THREADS demonstates the concurrent safety of atomic type. Also, compared with locks such as Mutex, atomic types have much greater efficiency.

One thing to note about is the parameter inside atomic type operation, Odering::Relaxed. This parameter is used to indicate the memory ordering. When the source code is compiled and run, the order of native code does not always strictly follow the order of the source code due to reordering during compiler optimization and CPU cache. There are five orders to choose from:

1. **Relaxed**: no restrictions on reordering
    
2. **Release**: set a memory barrier to ensure that all operations before will always come before it at runtime.
    
3. **Acquire**: set a memory barrier to ensure that all operations after will always come after it at runtime.
    
4. **AcqRel**: the combination of Release and Acquire
    
5. **SeqCst**: the strictest memory ordering, ensuring total order of all operations.
    

Normally, Release is used for writing operations, and Acquire is used for reading operations. A safer approach is to always use SeqCst, even though it may impair efficiency to some extent.

Here is another example that implements a spinning lock with Atomic:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::{hint, thread};

fn main() {
    let spinlock = Arc::new(AtomicUsize::new(1));
    let spinlock_clone = Arc::clone(&spinlock);
    let thread = thread::spawn(move || {
        spinlock_clone.store(0, Ordering::SeqCst);
    });

    while spinlock.load(Ordering::SeqCst) != 0 {
        hint::spin_loop();
    }

    if let Err(panic) = thread.join() {
        println!("{:?}", panic);
    }
}
```

### Thread Safety Based on Send and Sync Traits

An object must implement Send and Sync traits to be passed among differen threads. These two traits are marker traits, meaning that they do not have concrete methods. **Send** trait allows a type to be passed among threads via ownership transfer. **Sync** trait allows a type to be shared among treads through reference. If an object implements Sync, its reference must implement Send (that is, if T implements Sync, &T implements Send)

Most types in Rust implement Send and Sync by default. However, there are a few exceptions:

1. **raw pointer**: raw pointer has no safety guarantee, so it implements neither traits.
    
2. **Cell and RefCell**: Cell and RefCell are not Sync since their base implementation UnsafeCell is not Sync.
    
3. **Rc**: Rc is neither Send nor Sync, so it cannot be used in multithreading.
    

For a compound data type, if all of its attributes are Send and Sync, then the type also implements Send and Sync.