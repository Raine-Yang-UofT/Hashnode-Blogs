---
title: "Rust Learning Note: Smart Pointer"
datePublished: Wed Dec 20 2023 18:07:19 GMT+0000 (Coordinated Universal Time)
cuid: clqe365ns000808kxbvwv7eqo
slug: rust-learning-note-smart-pointer
tags: rust, drop, arc, smart-pointer, box, cell, deref, rc, refcell

---

This article is a summary of Chapter 4.4 in Rust Course ([course.rs](http://course.rs/))

### Box&lt;T&gt;

**Box** is used to allocate a value in the heap, and creates a pointer on the stack pointing to the value. It has these common uses:

**1 Allocate values on the heap**

Using Box, we can intentionally allocate values on the heap even if the value is by default allocated on the stack (such as i32).

```rust
fn main() {
    let a = Box::new(3);
    println!("a = {}", a);
```

At the println! statement, the smart pointer is automatically deferenced with Def, so we don't need to deference the pointer manually.

**2 Prevent copying values on the stack**

Variable reassignment on the stack are achieved through copying the entire values, and no transfer of ownership happens. However, values in the heap will not be transferred. Instead, a pointer to the value is transferred from one variable to another, which leads to ownership transfer.

```rust
fn main() {
    let arr = [0; 1000]    // stored in stack
    let arr1 = arr;    // copy the array, does not take ownership
    println!("{:?}", arr.len());
    println!("{:?}". arr1.len());

    let arr = Box::new([0; 1000]);    // stored in heap
    let arr1 = arr;    // arr1 takes the ownership of arr
```

**3 Invoking Dynamically Sized Type (DST)**

A data type is dynamically sized type (DST) if its size is unknown at compiler time. To give a few examples:

1 slicing: slicing types are DST, we can only use reference of slicing.

2 str: different from &str and String, str is a DST, since we cannot expect strings with different lengths to be stored in a data type with uniform size.

3 trait objects: since we do not know the exact type of objects that will be passed as trait objects, trait objects are DST.

4 recursive types: types that invoke itself in definition. For instance:

```rust
enum List {
    Cons(i32, List),
    Nil
{
```

All DST cannot be directly used. They can only be invoked through pointers, which have fixed size at compile time. For example, the recursive type above can be changed as follows:

```rust
enum List {
    Cons(i32, Box<List>),
    Nil
}
```

### Memory Allocation with Box

When we store a vector called vec1, the pointer is stored in stack and the elements are in the heap:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1703040241594/3acdaa6b-cb1c-4ee5-9f3b-47aa47c51761.png align="center")

Fig 1. Memory model of a vector. Image reproduced from (Rust Course)

If the elements are of type Box&lt;T&gt;, the pointer stored in stack points to a vector of Box in heap. Each Box pointers to its corresponding element.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1703040358869/694b674e-418d-4618-b162-c493ebed8ba6.png align="center")

Fig 2. Memory model of a vector with Box. Image reproduced from (Rust Course)

To retrieve values from vec2, we need to first deference vec2, and then dereference Box.

```rust
fn main() {
    let arr = vec![Box::new(1), Box::new(2)];
    let (first, second) = (&arr[0], &arr[1]);
    let sum = **first + **second
```

### Box::leak

Box::leak is an associated function that can consume Box and returns the value inside it. One use of it is to convert the lifecycle of an object to 'static at runtime. For example, in the code below, using Box::leak, we get a &str type with 'static lifecycle from the String type.

```rust
fn main() {
    let s = gen_static_str();
    println!("{}", s);
}

fn gen_static_Str() -> &'static str {
    let mut s = String::new();
    s.push_Str("hello, world");
    Box::leak(s.into_boxed_str())
}
```

### Automatic Dereferencing with Deref Trait

When we want to dereference an object with \*, the object must implement trait Deref. Here is an example implementation of Deref with MyBox

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

After MyBox implements Deref, we can use \*MyBox to get value x from MyBox(x). This is achieved through internally invoking

```rust
*(y.deref())
```

When an object that implements deref is passed as a function parameter, deref() can be automatically invoked based on the parameter signiture, for example:

```rust
fn main() {
    let s = String::from("hello world");
    display(&s)
}

fn display(s: &str) {
    println!("{}", s);
}
```

In the code above, String implements Deref trait and can be deferenced to &str. When &s is passed into function display, its type &String is automatically deferenced to &str. Note at the parameter must be passed as a reference to trigger the dereferencing.

The automatic dereferencing can be invoked consecutively until finding a suitable data type, as the code below would also work:

```rust
fn main() {
    let s = Box::new(String::from("hello world"));
    display(&s)
}

fn display(s: &str) {
    println!("{}", s);
}
```

### Deref and DerefMut

**T: Deref&lt;Target=U&gt;** can convert &T to &U, or &mut T to &U

**T: DerefMut&lt;Target=U&gt;** can convert &mut T to &mut U

To implement DerefMut, we need to first implement Deref for the object, as the example below:

```rust
struct MyBox<T> {
    v: T
}

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox {v: x}
    }
}

use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.v
    }
}

use std::ops::DerefMut;

impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.v
    }
}

fn main() {
    let mut s = MyBox::new(String::from("hello, "));
    display(&mut s)
}

fn display(s: &mut String) {
    s.push_str("world");
    println!("{}", s);
}
```

### Object Release with Drop

When an object leaves the scope and is to be released, the compiler will automatically call the **drop** method of the object. Thus, **Drop** trait can be used to execute certain cleanup task before the object is released.

```rust
struct HasDrop1;
struct HasDrop2;
impl Drop for HasDrop1 {
    fn drop(&mut self) {
        println!("Dropping HasDrop1");
    }
}
impl Drop for HasDrop2 {
    fn drop(&mut self) {
        println!("Dropping HasDrop2");
    }
}

struct HasTwoDrops {
    one: HasDrop1,
    two: HasDrop2
}
impl Drop for HasTwoDrops {
    fn drop(&mut self) {
        println!("Dropping HasTwoDrops");
    }
}

struct Foo;
impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo")
    }
}

fn main() {
    let x = HasTwoDrops {
        two: HasDrop2,
        one: HasDrop1
    };
    let foo = Foo
    println!("Running");
```

In the code above, we implement Drop trait for HasDrop1, HasDrop2, HasTwoDrops, and Foo. The result of this code will be:

> Running
> 
> Dropping Foo
> 
> Dropping HasTwoDrops
> 
> Dropping HasDrop1
> 
> Dropping HasDrop2

The dropping of variables in a scope follows the reverse order, as x is created before foo and foo is dropped before x. The dropping of struct attributes follows the order in which the attributes are defined. Since HasDrop2 is defined after HasDrop1 (even though assigned first), HasDrop1 is dropped before HasDrop2.

Even if we do not define Drop trait for x, the code would still run as follows, since almost every data type in Rust has implemented Drop by default.

When we want to drop an object manually, we can invoke the **drop** method of the object. However, we have to write in the form of **drop(object)**, instead of **object.drop(),** since in the latter case the drop method only uses a mutable borrowing of the object, and the variable is still accessible even when the object it refers to is dropped, leading to a compile error. **drop(object)** takes the ownership of the variable, so both the object and the variable are invalidated.

```rust
fn main() {
    let foo = Foo;
    // foo.drop();    ERROR
    drop(foo);
```

A final thing to note about is that trait Copy and Drop cannot coexist. Copy implies shallow copy semantics where the original and the copy are independent, while Drop indicates the object needs special cleanup when the value is dropped. The coexistence of Copy and Drop can lead to undefined behaviors such as double-free.

### Reference Counting with Rc&lt;T&gt; and Arc&lt;T&gt;

The ownership and borrowing mechanism requires that every reference type object can only have one reference. However, in some cases, we may want an object to have mutiple references. For example, in a graph data structure we may have multiple edges pointing to one node, and in multithreading we want multiple threads to hold one data. The solutions to these problems are **Rc** (in a single thread), and **Arc** (in multithreading).

**Rc&lt;T&gt;** uses reference counting to keep track of references to the object. The counting pluses one when a new variable refers to Rc&lt;T&gt; object and minus one when a variable is released. The Rc&lt;T&gt; object is released when the counter goes to 0. Using **Rc::clone()** we can assign a variable to an existing Rc&lt;T&gt; object, and with **Rc::strong\_count()** we can know the number of references to Rc&lt;T&gt;

```rust
use std::rc::Rc;
fn main() {
    let a = Rc::new(String::from("test ref counting"));
    println!("count after creating a {}", Rc::strong_count(&a));    // 1
    let b = Rc::clone(&a);
    println!("count after creating b {}", Rc::strong_count(&a));    // 2
    {
        let c = Rc::clone(&a);
        println!("count after creating c {}", Rc::strong_count(&c));    // 3
    }
    println!("count after c goes out of scope {}", Rc::strong_count(&a));    // 2
}
```

Rc&lt;T&gt; pointer is an immutable reference to the object in heap, since we do not allow the coexistence of multiple mutable references to the object.

**Arc&lt;T&gt;** (Atomic Rc) has the same API as Rc&lt;T&gt;. However, Arc&lt;T&gt; supports sharing data among multiple threads. Arc&lt;T&gt; is much less efficient than Rc&lt;T&gt;, so we should use Rc&lt;T&gt; when possible unless we're dealing with multithreading.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let s = Arc::new(String::from("test Arc"));
    for _ in 0..10 {
        let s = Arc::clone(&s);
        let handle = thread::spawn(move || {println!("{}", s)});
    }
}
```

### Internal Mutability with Cell&lt;T&gt; and RefCell&lt;T&gt;

**Cell&lt;T&gt;** is used when T implements Copy. YIt has method get() to retrieve the element and set() to change the element. Cell&lt;T&gt; can be used as a workaround of Rust borrowing rule, as in this example:

```rust
use std::cell::Cell;
fn main() {
    let c = Cell::new("asdf");
    let one = c.get();
    c.set("qwer");
    let two = c.get();
    println!("{}, {}", one, two);    // asdf, qwer
}
```

If we use reference in the code above, aftering assigning one to be an immutable borrowing, we cannot change c to another value, since that would lead to coexistence of mutable and immutable borrowing. However, with Cell we are able to change the value even when it has an immutable borrowing.

**RefCell&lt;T&gt;** is used when T does not implement Copy. It allows mutable and immutable reference to coexist at compile time. However, it cannot ignore the rule as it checks the borrowing errors at runtime instead. RefCell&lt;T&gt; is used when the compiler misidentify the code as violating the borrowing rule.

One of such cases is the implementation of internal mutability: the ability to mutate data even when accessed through immutable borrowing. In the example below, we want to implement a message queue that stores value in cache in send method. However, assume that the send method uses an immutable borrowing &self, we need to use RefCell to mutuate self.msg\_cache

```rust
use std::cell::RefCell;
pub trait Messenger {
    fn send(&self, msg: String);
}

pub struct MsgQueue {
    msg_cache: RefCell<Vec<String>>
}

impl Messenger for MsgQueue {
    fn send(&self, msg: String) {
        self.msg_cache.borrow_mut().push(msg)
    }
}

fn main() {
    let mq = MsgQueue { msg_cache: RefCell::new(Vec::new()) };
    mq.send("hello, world".to_string());
}
```

### Combining Rc and RefCell

Rc allows multiple ownership of an object, and RefCell allows internal mutability. A common combination in Rust is using Rc and RefCell together:

```rust
use std::cell::RefCell;
use std::rc::Rc;
fn main() {
    let s = Rc::new(RefCell::new("123".to_string()));
    let s1 = s.clone();
    let s2 = s.clone();
    s2.borrow_mut().push_str("4");
    println!("{:?}\n{:?}\n{:?}", s, s1, s2);
}
```

The result of the code above would be:

> RefCell { value: "1234" }
> 
> RefCell { value: "1234" }
> 
> RefCell { value: "1234" }

Note that when one of the reference mutates the object with borrow\_mut(), all references refer to the updated object.