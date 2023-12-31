---
title: "Rust Learning Note: Pin and UnPin"
datePublished: Sun Dec 31 2023 22:15:31 GMT+0000 (Coordinated Universal Time)
cuid: clqu1vpxk000008l24mb2gu4k
slug: rust-learning-note-pin-and-unpin
tags: pointers, rust, future, asyncawait, pin, unpin

---

This article is a summary of Chapter 4.11.3 of Rust Course ([course.rs/](http://course.rs/)) and blog ([folyd.com/blog/rust-pin-unpin/](https://folyd.com/blog/rust-pin-unpin/))

### The Use of Pin

**Pin** is a smart pointer that encapsulates another pointer. It is represented as Pin&lt;P&lt;T&gt;&gt;, in which P is a pointer pointing to an object of type T. Pin makes the object T unable to move its location in memory if T does not implement Unpin trait (that is, T is type **!Unpin**). If T implements Unpin, Pin has no effect on P&lt;T&gt; at all.

Pin is mainly used in resolving self-reference type, a types whose some attribute is a reference to its other attributes, as the example below:

```rust
#[derive(Debug)]
struct Test {
    a: String,
    b: *const String
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null()
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        assert!(!self.b.is_null());
        unsafe { &*(self.b) }
    }
}
```

In this code, we create a raw pointer b and points it to the memory address of a in init() method. An error would occur if the memory address of a is moved yet b is still pointing to the old address, as in this example:

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {} b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {} b: {}", test2.a(), test2.b());
}
```

In the code, we expect the second println statement would print "a: test1, b: test1". However, the output is actually "a: test1, b: test2". This is because b is still pointing to the previous memory address which is now in test1.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704057251779/a0ea4c7b-bd6c-4251-8ea6-737f029c396e.png align="center")

Fig 1. The raw pointer pointing to a moved address. Image reproduced from (Rust Course)

### Fix the Value on Stack

With Pin, we can solve the problem concerning moved data above. If an object does not implement Unpin trait, when it is put inside smart pointer Pin, we would be unable to get its mutable reference, and thus cannot invoke functions like swap. However, almost all Rust types automatically implement the Unpin trait, with only a few exceptions

1. PhantomPinned
    
2. Future in async/await
    

Since a composite data type would only automatically implement a trait only if all its attribute implement the trait, if we include at least one !Unpin attribute to the composite type, it would not implement Unpin. Normally, we include PhantomPinned into a type to make it !Unpin.

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned
        }
    }

    fn init(self: Pin<&mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn b(self: Pin<&Self>) -> &String {
        assert!(!self.b.is_null());
        unsafe { &*(self.b) }
    }
}
```

In this code, we will not be able to get a mutable reference of Test object in safe Rust. Note that in init we used a method **get\_unchecked\_mut** to get the mutable reference in the unsafe block. get\_unchecked\_mut returns a mutable reference of the pointer even if the type is !Unpin. Using this method requires us to follow the rules of Pin manually:

1. If P&lt;T&gt; implements Unpin, P&lt;T&gt; must be unpinned throughout its lifecycle
    
2. If P&lt;T&gt; implements !Unpin, P&lt;T&gt; must be pinned throughout its lifecycle
    

### Fix the Value on Heap

Using smart pointer Box and method **Box::pin**, we can allocation an !Unpin type on the heap and fix its location with Pin.

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn b(self: Pin<&Self>) -> &String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let test1 = Test::new("test1");
    let test2 = Test::new("test2");
    println!("a: {}, b: {}", test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}", test2.as_ref().a(), test2.as_ref().b());
}    
```

### The Use of Pin in Future

The underlying implementation of async/await programming is the Future trait. Future is one of the few traits that implement !Unpin. In fact, the introduction of Pin in Rust is largely due to issues related to Future. This is the API of Future trait:

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

Here's a standard use of async/await: {

```rust
let fut_one = /*... */ // Future 1
let fut_two = /* ... */ // Future 2
async move {
    fut_one.await;
    fut_two.await;
}
```

The code above is actually a syntatic suger. Its underlying implementation is a Future type with poll method:

```rust
struct AsyncFuxture {
    fut_one: FutOne,
    fut_two: FutTwo,
    state: State
}

enum State {
    AwaitingFutOne,
    AwaitingFutTwo,
    Done
}

impl Future for AsyncFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self.state {
                State::AwaitingFutOne => match self.fut_one.poll(..) {
                    Poll::Ready(()) => self.state = State::AwaitingFutTwo,
                    Poll::Pending => return Poll::Pending
                State::AwaitingFutTwo => match self.fut_two.poll(..) {
                    Poll::Ready(()) => self.state = State::Done,
                    Poll::Pending => return Poll::Pending
                }
                State::Done => return Poll::Ready(())
            }
        }
    }
}
```

If we include reference types in async block, issues may occur:

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

This code will be converted to:

```rust
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8]
}

struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf
}
```

In this case, ReadIntoBuf has an attribute buf, which is a pointer to x. As a result, if AsyncFuture is moved, x will be moved as well, leading to an invalid address for buf. The way to prevent that is to fix Future to a specific memory location