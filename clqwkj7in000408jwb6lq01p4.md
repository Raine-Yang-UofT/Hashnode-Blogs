---
title: "Rust Learning Note: async/await and Stream"
datePublished: Tue Jan 02 2024 16:33:12 GMT+0000 (Coordinated Universal Time)
cuid: clqwkj7in000408jwb6lq01p4
slug: rust-learning-note-asyncawait-and-stream
tags: stream, rust, lifecycle, asyncawait

---

This article is a summary of Chapter 4.11.4 of Rust Course ([course.rs/](http://course.rs/))

### The Lifecycle of async

If an async function has reference type parameters, the parameter it refers to must life longer than the Future the function returns. For instance, the code below would throw an error since the variable x only lives inside function bad(), but the async function borrow\_x, after being returned, lives in a larger scope than x.

```rust
use std::future::Future;
fn bad() -> impl Future<Output = u8> {
    let x = 5;
    borrow_x(&x)
}

async fn borrow_x(x: &u8) -> u8 {*x}
```

One solution to this problem is to put the variables the function refer to along with the function inside a single async block. In this way, the function and the variables it refers to always exist in the same scope, and thus have the same lifecycle.

```rust
use std::future::Future;

async fn borrow_x(x: &u8) -> u8 { *x }

fn good() -> impl Future<Output = u8> {
    async {
        let x = 5;
        borrow_x(&x).await
    }
}
```

Another solution is to use **move** keyword to transfer the ownership of the variable into the async function, which is similar to **move** in closures. However, this will no longer allow us to use the moved variable anywhere else.

```rust
fn move_block() -> impl Future<Output = ()> {
    let my_string = "foo".to_string();
    async move {
        println!("{my_string}");
    }
}
```

### .await in Multithreading Executor

Variable in async blocks need to be passed among multiple threads (since async/await is a M:N threading model), so data types that do not implement Send and Sync traits, like Rc, RefCell, cannot be used in async blocks. Also, the Mutex in std::sync::Mutex is not safe to use in async/await, since it is possible that one .await function that acquires a lock is suspended before releasing the lock, and another function currently processed by the thread tries the acquire the lock, leading to a deadlock. We should use the Mutex in **futures::lock::Mutex** instead.

### Stream Processing

**Stream** trait is similar to Future trait, except that it can return multiple Future objects. Stream is similar to Iterator.

```rust
trait Stream {
    type Item;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<Self::Item>>;
}
```

One use of Stream is the **Receiver** of message channel. The Stream receives a Some(val) when the sender sends a message, and None when the channel is closed. In the code below, Stream is implicitly used in the receiver (rx). The method call **rx.next()** is invoking the Stream trait implementation for the receiver.

```rust
async fn send_recv() {
    const BUFFER_SIZE: usize = 10;
    let (mut tx, mut rx) = mpsc::channel::<i32>(BUFFER_SIZE);

    tx.send(1).await.unwrap();
    tx.send(2).await.unwrap();
    drop(tx);

    assert_eq!(Some(1), rx.next().await);
    assert_eq!(Some(2), rx.next().await);
    assert_eq!(None, rx.next().await);   
}
```

Similar to an Iterator, we can also iterate a Stream and use methods like map, filter, and fold. However, we cannot use a for loop to loop through a Stream. Instead, we can use **while let** for looping.

```rust
async fn sum_with_next(mut stream: Pin<&mut dyn Stream<Item = i32>>) -> i32 {
    use futures::stream::streamExt;
    let mut sum = 0;
    while let Some(item) = stream.next().await {
        sum += item;
    }
    sum
}

async fn sum_with_try_next(
    mut stream: Pin<&mut dyn Stream<Item = Result<i32, io::Error>>>
) -> Result<i32, io::Error> {
    use futures::stream::TryStreamExt;
    let mut sum = 0;
    while let Some(item) = stream.try_next().await? {
        sum += item;
    }
    Ok(sum)
}
```

However, the approach above processes only one value each time, and blocks the thread with await when waiting for the next value, which loses the meaning of concurrent programming. We can use **for\_each\_concurrent** or **try\_for\_each\_concurrent** to process multiple values from Stream concurrently.

```rust
async fn jump_around(
    mut stream: Pin<&mut dyn Stream<Item = Result<u8, io::Error>>>
) -> Result<(), io::Error> {
    use futures::stream::TryStreamExt;
    const MAX_CONCURRENT_JUMPERS: usize = 100;

    stream.try_for_each_concurrent(MAX_CONCURRENT_JUMPERS, |num| async move {
        jump_n_times(num).await?;
        report_n_jumps(num).await?;
        Ok(())
    }).await?;
    
    Ok(())
}
```

In this example, **try\_for\_each\_concurrent** method is used to apply an asynchronous closure to each item in the stream concurrently. Inside the closure, we defines two custome functions and uses ? to propagate error.