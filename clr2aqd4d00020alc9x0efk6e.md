---
title: "Rust Learning Note:Writing a Simple Tokio"
datePublished: Sat Jan 06 2024 16:45:27 GMT+0000 (Coordinated Universal Time)
cuid: clr2aqd4d00020alc9x0efk6e
slug: rust-learning-notewriting-a-simple-tokio
tags: rust, future, asyncawait, tokio

---

This article is a summary of Chapter 6.8 of Rust Course ([course.rs/](http://course.rs/))

### Future Trait

In async/await programming, an **async** function will not be executed until it is invoked by **.await**. This delayed execution is implemented through Future trait, and the return value of any async function is a trait object of Future. Here is the definition of Future trait:

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

**Future** trait defines a series of asynchronous computations. **poll()** method is used to progress in the computation. It has an associated type **Ouput**, which stands for the return type when Future completes the computation. poll() returns an Poll enum type, including **Poll::Ready&lt;Output&gt;**, indicating that the computation is finished, and **Poll::Pending**, indicating that the computation has not been completed.

Here we have a struct Delay that implements Future. Its poll method waits for a certain time interval before returning Poll::Ready("done").

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant
}

impl Future for Delay {
    type Output = &'static str;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<&'static str> {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done");
        } else {
            // ignore this code by now
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };
    let out = future.await;
    assert_eq!(out, "done");
}
```

### Executor

In the code above, we need to create an async main function from tokio to run the Future. As we mentioned above, only .await can invoke a Future. However, .await can only be declared in async functions, whose return types themselves are Future. As a result, we need to implement an Executor to process the Future in the outmost layer. Here is an example code:

```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

struct MiniTokio {
    tasks: VecDeque<Task>
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new()
        }
    }

    fn spawn<F>(&mut self, future: F)
    where F: Future<Output = ()> + Send + 'static {
        self.tasks.push_back(Box::pin(future));
    }

    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker);
        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

In this code, the **run** method continuously retrieves Future from the queue calls the poll() method. If the Future returns Poll::Pending, it puts the Future back to the queue. However, this method is inefficient. The Executor would poll all Future continuously, and very likely most polling are meaningless as the Future simply returns Poll::Pending.

A more ideal solution is that Future could notify the Executor when to poll. Only when the Future is able to continue its processing (such as when it finally gets a result from another calculation or I/O) would it be polled again. This approach is implemented through Waker.

### Waker

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context)
    -> Poll<Self::Output>;
```

In the poll() method, **Waker** is contained in parameter Context and can be retrieved by calling **cx.waker()**. The **wake()** method defined in Waker is used to notify the executor that the Future is ready to be polled again.

Here we implement Waker for Delay:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant
}

impl Future for Delay {
    type Output = &'static str;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<&'static str> {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done");
        } else {
            let waker = cx.waker().clone();
            let when = self.when;
            thread::spawn(move || {
                let now = Instant::now();
                if now < when {
                    thread::sleep(when - now);
                }
                waker.wake();
            });
            Poll::Pending
        }
    }
}
```

In this updated code, we rewrite the else branch in which Delay returns Poll::Pending. We spawn a new thread and sleeps for the given time interval, and calls **waker.wake()** when the thread wakes up. Note that **when a Future returns Poll::Pending, wake() must be invoked**, otherwise the Future will be suspended indefinitely and will never be polled again.

### Processing wake()

We will now rewrite MiniTokio to allow it receives wake notification. We will use a message channel to store all Future waiting to be polled. Once a Future calls wake(), it will be added into the message channel. To implement this, we will decorate Future with a new struct **Task**, which includes the Future object and the Sender of message channel. Once wake() is invoked, the sender will send the Task into the message channel, which will be polled by the Executor.

Since the message receiver and sender may be in different threads, the message channel need to implement Sync and Send. However, the message channel in standard library does not implement Sync, so we need to use library crossbeam

> // in Cargo.toml \[dependencies\]
> 
> crossbeam = "0.8"

```rust
struct MiniTokio {
    scheduled: channel::Receiver<Arc<Task>>,
    sender: channel::Sender<Arc<Task>>
}

impl MiniTokio {
    fn new() -> MiniTokio {
        let (sender, scheduled) = channel::unbounded();
        MiniTokio { scheduled, sender }
    }
}

struct Task {
    future: Mutex<Pin<Box<dyn Future<Output = ()> + Send>>>,
    executor: channel::Sender<Arc<Task>>
}

impl Task {
    fn schedule(self: &Arc<Self>) {
        self.executor.send(self.clone());
    }
}
```

In the code, MiniTokio holds the receiver (scheduled) and sender (sender) of the message channel. Task contains the Future trait object and the sender of message channel. Method **schedule()** is used to send a message to MiniTokio.

In order to convert Task in to a Waker, we can make it implement **ArcWake** trait in futures module. First we need to add futures to dependencies in Cargo.toml

> futures = "0.3"

And then implement ArcWake for Task

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.schedule();
    }
}
```

When the Future calls waker.wake(), the schedule() method in Task will be invoked. After that, we can implement two other methods for Task: **poll()** for passing the waker to the poll() method in Future and calling poll() in Future, and **spawn()** for creating a new Task from a Future and sending the Task to MiniTokio for initial polling.

```rust
impl Task {
    fn poll(self: Arc<Self>) {
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);
        let mut future = self.future.try_lock().unwrap();
        let _ = future.as_mut().poll(&mut cx);
    }

    fn spawn<F>(future: F, sender: &channel::Sender<Arc<Task>>)
    where F: Future<Output = ()> + Send + 'static {
        let task = Arc::new(Task {
            future: Mutex::new(Box::pin(future)),
            executor: sender.clone() 
        });
    
        let _ = sender.send(task);
    }
}
```

Finally, we need to implement the **run()** method for MiniTokio that retrieves Task from message channel and calls poll() method of the Task.

```rust
impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    fn spawn<F>(&self, future: F)
    where F: Future<Output = ()> + Send + 'static {
        Task::spawn(future, &self.sender);
    }
}
```

Here is the complete code for MiniTokio:

```rust
use std::future::Future;
use futures::task::{self, ArcWake};
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use std::thread;
use crossbeam::channel;
use std::sync::{Arc, Mutex};

struct Delay {
    when: Instant
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self:Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<&'static str> {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            let waker = cx.waker().clone();
            let when = self.when;

            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                waker.wake();
            });

            Poll::Pending
        }
    }
}


struct MiniTokio {
    scheduled: channel::Receiver<Arc<Task>>,
    sender: channel::Sender<Arc<Task>>
}

impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    fn new() -> MiniTokio {
        let (sender, scheduled) = channel::unbounded();
        MiniTokio {scheduled, sender}
    }

    fn spawn<F>(&self, future: F)
    where F: Future<Output = ()> + Send + 'static {
        Task::spawn(future, &self.sender);
    }
}


struct Task {
    future: Mutex<Pin<Box<dyn Future<Output = ()> + Send>>>,
    executor: channel::Sender<Arc<Task>>
}

impl Task {
    fn schedule(self: &Arc<Self>) {
        self.executor.send(self.clone());
    }

    fn poll(self: Arc<Self>) {
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);
        let mut future = self.future.try_lock().unwrap();
        let _ = future.as_mut().poll(&mut cx);
    }

    fn spawn<F>(future: F, sender: &channel::Sender<Arc<Task>>)
    where F: Future<Output = ()> + Send + 'static {
        let task = Arc::new(Task {
            future: Mutex::new(Box::pin(future)),
            executor: sender.clone()
        });

        let _ = sender.send(task);
    }
}

impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.schedule();
    }
}
```