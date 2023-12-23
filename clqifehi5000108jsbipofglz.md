---
title: "Rust Learning Note: Multithreading"
datePublished: Sat Dec 23 2023 19:00:48 GMT+0000 (Coordinated Universal Time)
cuid: clqifehi5000108jsbipofglz
slug: rust-learning-note-multithreading
tags: multithreading, concurrency, rust, threads

---

This article is a summary of chapter 4.6.1 and 4.6.2 of Rust Course ([course.rs/](https://course.rs/))

### Concurrent and Parallel Programming

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1703348456632/43348dcb-89b9-41e8-b035-cd930670ae16.png align="center")

Fig 1 Difference between concurrent and parallel. Image reproduced from (Rust Course)

**Concurrent** refers to having one CPU core dealing with multiple threads. Every there there is only one thread being executed, and the operating system has specific dispatch algorithms to handing the switching among multiple threads, so the threads appear to be running simultaneously.

**Parallel** refers to having a CPU core handling one thread, so multiple threads are indeed processed in parallel.

In a more formal definition, a system is concurrent if it allows multiple threads to **coexist**, and it is parallel if it allows multiple threads to be **executed simultaneously**. From this definition, parallel processing is a subset of concurrent procession.

Different programming languages have different implementations of concurrent programming. Rust uses **1:1 thread model,** meaning that it invokes API provided by the operating system to create threads, and the threads in the program are exactly the same as threads created by the operating systems. Some languages like Go use **M:N thread model**, meaning that the language has its own implementation of threading model, and M threads created in the language are mapped to N system threads based on the model.

### Creating Thread

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("number {} from the spawned thread", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
    
    for i in 1..5 {
        println!("number {} from the main thread", i);
        thread::sleep(Duration::from_millis(1));
    }
} 
```

We use **thread::spawn** to create a new thread. The code inside a thread is executed using a closure. **thread::sleep** will make a thread sleep for a certain amount of time, during which other threads will be processed. In the code above, the thread spawned may not be able to finish execution since the whole program will terminate once the main thread stops. To solve this program, we can use **handle.join**, which blocks the current thread until the thread it waits for finishes processing.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("number {} from the spawned thread", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    handle.join().unwrap();

    for i in 1..5 {
        println!("number {} from the main thread", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

In the code above, **handle.join().unwrap()** waits the tread handle to be executed before the for loop in the main thread. If we put handle.join().unwrap() at the bottom, the for loop in the thread handle and that in the main thread will be executed alternatively until they both finish.

### Transfer External Variables with Move

Keyword **move** is used to take the ownership of a variable when it is used in a closure. It can also be used to tranfer ownership from one thread to another.

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("{:?}", v);
    });

    handle.join().unwrap();
}
```

In this code, **move** transfers the ownership of v from main thread to the thread handle. An error would occur without **move** since Rust cannot determine the lifecycle of v compared with the lifecycle of the thread. It is possible that v is released before the thread.

### How Threads Terminate

In Rust, once a thread is created, it would not terminate until it is finished processing or the main thread finished, even if the thread that create it is finished. This is to prevent unexpected behavior for a thread to be terminated before it finishes.

```rust
use std::thread;
use std::time::Duration;
fn main() {
    let new_thread = thread::spawn(move || {
        thread::spawn(move || {
            loop {
                println!("I am a new thread");
            }
        })
    });

    new_thread.join().unwrap();
    println!("Child thread is finish");
    thread::sleep(Duration::from_millis(100));
}
```

In this case, the thread created inside the new\_thread will continuously print "I am a new thread" until the main thread finishes, even after new\_thread finishes.

### Thread Barrier

Thread barrier is used to block a thread until multiple threads reach a certain point.

```rust
use std::sync::{Arc, Barrier};
use std::thread;

fn main() {
    let mut handles = Vec::with_capacity(6);
    let barrier = Arc::new(Barrier::new(6));

    for _ in 0..6 {
        let b = barrier.clone();
        handles.push(thread::spawn(move|| {
            println!("before wait");
            b.wait();
            println!("after wait");
        )));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

In this example, we create a new Barrier that blocks the thread until 6 threads reach the barrier. b.wait() blocks a single thread, and when all 6 threads reach wait() the threads will continue executing.

### Thread Local Variable

A **thread local variable** is a variable that has a separate and independent value for in thread. Each thread would get the initial value of the variable and the change of the variable in each thread would not affect that in other threads. In Rust, we use macro **thread\_local** to initialize a thread local variable, and we use **with** to use the variable inside a thread. A thread local variable has a 'static lifecycle declared with keyword **static**.

```rust
use std::cell::RefCell;
use std::thread;

thread_local!(static FOO: RefCell<u32> = RefCell::new(1));

FOO.with(|F| {
    assert_eq!(*f.borrow(), 1);
    *f.borrow_mut() = 2;
});

let t = thread::spawn(move || {
    FOO.with(|f| {
        assert_eq!(*f.borrow(), 1);
        *f.borrow_mut() = 3;
    });
});

t.join().unwrap();

FOO.with(|f| {
    assert_eq!(*f.borrow(), 2);
});
```

### Conditional Variable

Using conditional variable with Mutex can suspend a thread and continue it after certain condition.

```rust
use std::thread;
use std::sync::{Arc, Mutex, Condvar};

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = pair.clone();

    thread::spawn(move || {
        let (lock, cvar) = &*pair2;
        let mut started = lock.lock().unwrap();
        println!("changing started");
        *started = true;
        cvar.notify_one();
    });

    let (lock, cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    while !*started {
        started = cvar.wait(started).unwrap();
    }

    println!("started changed");
}
```

In the code above, **Mutex** provides mutual exclusion, allowing only one thread to access the data at a time. **Condvar** is the conditional variable that enables the thread to wait for a particular condition to become true.

We create a Arc containing a tuple (Mutex, Condvar) to share the pair across multiple threads. In the thread, the thread acquires a lock on Mutex and changes started to true, then it notify the Condvar that the condition has been met. In the main thread, the main thread acquires a lock on Mutex, and loops until started become true. In the loop, cvar waits the notification of spawned thread that changes started.

### call\_once Method

Sometimes, we want a certain functions, like initializing global variables, to be only invoked once by one thread and is ignored by threads following it. This can be achieved through call\_once function in Once type

```rust
use std::thread;
use std::sync::Once;

static mut VAL: usize = 0;
static INIT: Once = Once::new();

fn main() {
    let handle1 = thread::spawn(move || {
        INIT.call_once(|| {
            unsafe {
                VAL = 1;
            }
        });
    });

    let handle2 = thread::spawn(move || {
        INIT.call_once(|| {
            unsafe {
                VAL = 2;
            }
        });
    });
    
    handle1.join().unwrap();
    handle2.join().unwrap();
        
    println!("{}", unsafe { VAL });
}
```

In this case, INIT.call\_once will be invoked by the first thread that calls the function. After that, the second thread would not invoke its INIT.call\_once method. Since the creation of threads are asynchorous and slow, the code above may produce 1 or 2.