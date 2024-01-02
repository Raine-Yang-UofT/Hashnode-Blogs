---
title: "Rust Learning Note: Running Multiple Future Simultaneously"
datePublished: Tue Jan 02 2024 19:30:26 GMT+0000 (Coordinated Universal Time)
cuid: clqwqv44s000208l5co0k7muw
slug: rust-learning-note-running-multiple-future-simultaneously
tags: rust, future, select, asyncawait, join

---

This article is a summary of Chapter 4.11.5 of Rust Course ([course.rs/](http://course.rs/))

### join!

.await does not allow us to run multiple Future simultaneously. We have to wait for one Future to finish processing before another. Sometimes, we want to process a few Future concurrently, which requires the use of **join!** macro:

```rust
use futures::join;

async fn enjoy_book_and_music() -> (Book, Music) {
    let book_fut = enjoy_book();
    let music_fut = enjoy_music();
    join!(book_fut, music_fut)
}
```

**join!** takes a tuple of two Future, and returns a tuple containing the results of the two Future after they are both completed. If we want to concurrently run multiple Future stored in an array, we can use **futures::future::join\_all** method

```rust
use futures::future::join_all;

async fn async_task(id: u32) -> u32 {
    async_std::task::sleep(std::time::Duration::from_secs(1)).await;
    println!("Task {} completed", id);
    id
}

#[tokio::main]
async fn main() {
    let tasks = vec![
        async_task(1),
        async_task(2),
        async_task(3)
    ];

    let results = join_all(tasks).await;
    println!("All tasks completed");

    for result in results {
        println!("Result: {}", result);
    }
}
```

### try\_join!

join! is completed executing only when all the Future inside finish. If we want to terminal all Future if any Future throws an error, we can use **try\_join!**

```rust
use futures::try_join;

async fn get_book() -> Result<Book, String> {Ok(Book)};
async fn get_music() -> Result<Music, String> {Ok(Music)};

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book();
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```

Note that the Future in try\_join! must have the same error type. If the functions return different errors, we can use **map\_err** and **err\_info** in futures::future::TryFutureExt for error conversion.

```rust
use futures::{
    future::TryFutureExt,
    try_join
};

async fn get_book() -> Result<Book, ()> {Ok(Book)};
async fn get_music() -> Result<Music, String> {Ok(Music)};

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book().map_err(|()|, "unable to get book".to_string());
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```

### select!

join! only returns when all Future complete. Sometimes, we want to wait for a few Future, and returns whenever one Future completes and does not wait for the rest of the Future. In this case, we can use **select!** macro.

```rust
use futures::{
    future::FutureExt,
    pin_mut,
    select
};

async fn task_one() {};
async fn task_two() {};

async fn race_tasks() {
    let t1 = task_one().fuse();
    let t2 = task_two().fuse();
    pin_mut!(t1, t2);
    select! {
        () = t1 => println!("task 1 finishes first"),
        () = t2 => println!("task 2 finishes first")
    }
}
```

In the code above, we use **fuse()** to implement **FusedFuture** for Future, which makes select unable to poll a Future once it's finished. **pin\_mut!()** is used to implement **Unpin** trait, which allows select to obtain a mutable reference of Future. These two traits **FusedFuture** and **Unpin** are required for select to work properly.

select! also supports **default** and **complete** branch. **complete** branch is called when all Future and Stream are completed. It is commonly used together with loop. **default** branch is called when none of the Future or Stream are at Ready state.

```rust
use futures::future;
use futures::select;
pub fn main() {
    let mut a_fut = future::ready(4);
    let mut b_fut = future::ready(6);
    let mut total = 0;

    loop {
        select!{
            a = a_fut => total += a,
            b = b_fut => total += b,
            complete => break,
            default => panic!()
        };
    }
    assert_eq!(total, 10);
}
```

With Stream, we need to implement FusedStream with fuse(). When a Stream implements FusedStream, the Future it returns with next() will also implement FusedFuture.

```rust
use futures::{
    stream::{Stream, StreamExt, FusedStream},
    select
}

async fn add_two_streams(
    mut s1: impl Stream<Item = u8> + FusedStream + Unpin,
    mut s2: impl Stream<Item = u8> + FusedStream + Unpin
) -> u8 {
    let mut total = 0;

    loop {
        let item = select! {
            x = s1.next() => x,
            x = s2.next() => x,
            complete => break
        };
        if let Some(next_num) = item {
            total += next_num;
        }
    }
    total
}
```