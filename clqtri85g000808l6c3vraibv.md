---
title: "Rust Learning Note: Creating a Timer with Async/Await"
datePublished: Sun Dec 31 2023 17:25:05 GMT+0000 (Coordinated Universal Time)
cuid: clqtri85g000808l6c3vraibv
slug: rust-learning-note-creating-a-timer-with-asyncawait
tags: rust, future, await, async, asyncawait, waker

---

This article is a summary of Chapter 4.11.2 of Rust Course ([course.rs/](https://course.rs/))

### Future Trait

**Future** trait is the fundamental building block for Rust async/await mechanism. It represents a computation that may produce a value in the future. Future trait defines a poll method that is used to produce the output asynchronously.

```rust
trait Future {
    type Output;
    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending
}
```

When the poll method is invoked, it attempts to make progress in its computation. If the computation is completed in one poll, Ready&lt;T&gt; is returned indicating the task is completed. If the computation cannot be completed in a poll, Pending is returned. In addition, the **Waker** obtained from parameter **cx** is stored in future. **Waker** is used to wake up a task by notifying the executor that the task can be polled again.

Here is a Timer that uses Future trait. It creates a Future that completes after a specified duration. When the timer is created, a thread a started a sleep for a certain duration. The thread notifies the Future when the sleep ends.

```rust
use std::{
    future::Future,
    pin::Pin,
    sync::{Arc, Mutex},
    task::{Context, Poll, Waker},
    thread,
    time::Duration
};


pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>
}

struct SharedState {
    completed: bool,
    waker: Option<Waker>
}

impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}

impl TimerFuture {
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker:None
        }));

        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}
```

Explanation:

1. SharedState struct is used to sychronize the state between Future and thread. When the thread finishes the sleep, it changes completed to true and wakes up the Future. Since the SharedState struct need to be accessed and mutated by multiple threads, it is inside Arc&lt;Mutex&lt;&gt;&gt;
    
2. We implement Future trait for TimerFuture. The poll method returns Ready (meaning the task is completed) when shared\_state.completed is set to true. If it is false, we store the waker get from the parameter cx and returns Pending (meaning the task is not completed).
    
3. We then implement constructor for TimerFuture. The constructor creates a new SharedState, and creates a thread the sleeps for a certain duration. When the thread wakes up from sleep, it sets shared\_state.completed to true and notifies the Future to poll again through calling **wake()** method.
    

To use the Future timer, we need an **Executor** to initially poll the timers. After that, the Executor only polls a timer when it is notified by the **wake** method. Here is a simple implementation of an executor:

```rust
use {
    futures::{
        future::{BoxFuture, FutureExt},
        task::{waker_ref, ArcWake}
    },
    std::{
        future::Future,
        sync::mpsc::{sync_channel, Receiver, SyncSender},
        sync::{Arc, Mutex},
        task::{Context, Poll},
        time::Duration
    },
    timer_future::TimerFuture
};


struct Executor {
    ready_queue: Receiver<Arc<Task>>
}

#[derive(Clone)]
struct Spawner {
    task_sender: SyncSender<Arc<Task>>
}

struct Task {
    future: Mutex<Option<BoxFuture<'static, ()>>>,
    task_sender: SyncSender<Arc<Task>>
}

fn new_executor_and_spawner() -> (Executor, Spawner) {
    const MAX_QUEUED_TASKS: usize = 10000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    (Executor {ready_queue}, Spawner {task_sender})
}

impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            task_sender: self.task_sender.clone()
        });
        self.task_sender.send(task).expect("The task queue is full");
    }
}

impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let cloned = arc_self.clone();
        arc_self.task_sender.send(cloned).expect("The task queue is full");
    }
}

impl Executor {
    fn run(&self) {
        while let Ok(task) = self.ready_queue.recv() {
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&*waker);
                if future.as_mut().poll(context).is_pending() {
                    *future_slot = Some(future);
                }
            }
        }
    }
}


fn main() {
    let (executor, spawner) = new_executor_and_spawner();

    spawner.spawn(async {
        println!("howdy!");
        TimerFuture::new(Duration::new(2, 0)).await;
        println!("done!");
    });

    drop(spawner);

    executor.run();
}
```

1. **Executor** is used to receive tasks from the mpsc channel and execute them. **Spawner** is used to create new Future timers and send them to the mpsc channel. **Task** contains the Future timer and sender to send itself to Executor.
    
2. new\_executor\_and\_spawner() method initializes a sychronous mpsc channel with capacity 10000, and assigns the channel sender and receiver to Spawner and Executor.
    
3. The **spawn** method of Spawner takes a trait object of Future and wraps it in a Box to make its lifecycle 'static. It creates a Task from the Future object and sends it to the mpsc channel.
    
4. **ArcWake** trait enables Task to wake itself up when it is ready to be polled again. This trait contains **wake\_by\_ref** method, in which we make a clone of an Arc pointer to the task and send it back to the mpsc channel.
    
5. Finally, we implement **run** method for Executor to contiously polling tasks from the mpsc channel.