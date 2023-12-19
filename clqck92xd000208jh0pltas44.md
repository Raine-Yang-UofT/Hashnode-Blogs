---
title: "Rust Learning Note: Closure and Iterator"
datePublished: Tue Dec 19 2023 16:29:56 GMT+0000 (Coordinated Universal Time)
cuid: clqck92xd000208jh0pltas44
slug: rust-learning-note-closure-and-iterator
tags: closure, rust, iterator

---

This blog is a summay of Chapter 4.2 of Rust Course ([**course.rs**](https://course.rs/))

### Closure

**Closure** is an anonymous function that can be assigned to variables and passed as parameters or return values. In addition, it can capture values in the scope where it is invoked. Here is an example of closure:

```rust
fn main() {
    let x = 1;
    let sum = |y| x + y;
    assert_eq!(3, sum(2));
}
```

This closure expression |y| x + y has a parameter y, and captures variable x in main function. sum(2) adds y=2 with x=1.

Closure can utilize the type inference of the compiler, that is, the compiler can automatically assign data types on closure variables based on how the closure is invoked. Note that this is not the same as generics, as there can be only one data type for the variable.

### Closure in Struct

We can assign a closure as a struct attribute:

```rust
struct Cacher<T> where T: Fn(u32) -> u32 {
    query: T,
    Value: Option<u32>
}
```

The type bound of T, Fn(u32) -&gt; u32, represents a function or closure with a u32 type parameter and a u32 type return value.

We can also implement methods for this struct

```rust
impl<T> Cacher<T> where T: Fn(u32) -> u32 {
    fn new(query: T) -> Cacher<T> {
        Cacher {
            query,
            value: None
        }
    }

    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.query)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}
```

### Three Fn Traits: Fn, FnOnce, FnMut

There are three ways for a closure to capture a variable in a scope: taking ownership, mutable borrowing, and immutable borrowing. These three ways correspond with three types of closure traits, FnOnce, FnMut, and Fn.

**FnOnce:**

FnOnce means the closure can be run only once, since it takes away the ownership of values owned by the variables it use.

```rust
fn fn_once<F>(func: F) where F: FnOnce(usize) -> bool {
    println!("{}", func(3));
    println!("{}", func(4));
}

fn main() {
    let x = vec![1, 2, 3];
    fn_once(|z| {z == x.len()})
}
```

The code above would throw an error, since the first time func is involked, the ownership of the vector by x is already taken by the closure. In the second time func(4), variable x is not valid. The solution to this problem is to implement trait Copy for F, so the vector referred to by x is copied, instead of transfered.

```rust
fn fn_once<F>(func: F) where F: FnOnce(usize) -> bool + Copy {
    println!("{}", func(3));
    println!("{}", func(4));
}

fn main() {
    let x = vec![1, 2, 3];
    fn_once(|z| {z == x.len()})
}
```

If we want to intentionally transfer the ownerships of variables captured by closure, we can use **move** keyword. This is commonly used when the lifecycle of the closure is longer than that of the variable used.

**FnMut:**

FnMut captures the variable through mutable borrowing, so the values referred to by the variables can be changed in closure.

```rust
fn exec<'a, F: FnMut(&'a str)>(mut f: F) {
    f("hello")
}

fn main() {
    let mut s = String::new();
    let update_string = |str| s.push_str(str);
    exec(update_string);
    println!("{:?}". s);
}
```

In this case, variable s is passed as a mutable reference into the closure and is appended with string "hello". Note that whether the closure is mutable is nothing to do with the mutability of the variable assigned with the closure, update\_string. In the code, the ownership of the closure is transferred from update\_string to the function. This is because the closure does not implement Copy trait, so update\_string is passed into exec() through ownership transfer. By default, if all variables captured by the closure implement Copy, the closure would also implement Copy.

**Fn:**

Fn captures variables with immutable borrowing, so it cannot alter the variables it uses.

```rust
fn exec<'a, F: Fn(String) -> ()>(f: F) {
    f("world".to_string())
}

fn main() {
    let s = "hello, ".to_string();
    let update_string = |str| println!("{}, {}", s, str)
    exec(update_string);
    println!("{:?}", s);
}
```

Note that the type of Fn trait depends on how the closure uses the variable it captures internally (whether the use is ownership transfer, immutable borrowing, or mutable borrowing), instead of how the closure captures the variable. As a result, even if a closure captures a variable with **move**, it may still be FnMut or Fn if it only uses the variable as mutable borrowing or immutable borrowing.

The three traits are not independent from each other. In fact, a closure that implements FnMut must implement FnOnce, and a closure that implements Fn must implement FnMut. All closures need to implement FnOnce. Here is a illustration of the relationships among the three traits:

```rust
pub trait Fn<Args>: FnMut<Args> {
}

pub trait FnMut<Args>: FnOnce<Args> {
}

pub trait FnOnce<Args> {
    type Output;
}
```

### Closure as Function Return Value

Since in Rust, the size of return value of a function must be known in compile time, the closure itself, with indeterminate size, cannot be a return value. One way is to use impl Trait to return the object that implements closure trait:

```rust
fn factory() -> impl Fn(i32) -> i32 {
    let num = 5;
    |x| x + num
}
```

However, the restriction of this approach is that we can only return one closure, since two closures, even with the same signiture, are considered as hacing different types. The solution to this is using trait object:

```rust
fn factory(x:i32) -> Box<dyn Fn(i32) -> i32> {
    let num = 5;
    if x > 1 {
        Box::new(move |x| x + num)
    } else {
        Box::new(move |x| x - num)
    }
}
```

### Iterator

An **Iterator** allow us to loop through a collection data type, like array, Vec, HashMap without cinsidering indexing. The built-in collection types normally implements **IntoIterator** trait, which allows we to convery them to iterators with method **into\_iter**.

```rust
let arr = [1, 2, 3];
for v in arr {
    println!("{}", v);
}
```

The code above shows a simple foreach loop. In fact, foreach is a syntatic suger for the use of iterator. It's underlying implementation is like this:

```rust
let arr = [1, 2, 3];
for v in arr.into_iter() {
    println!("{}", v);
}
```

In addition to into\_iter() there are three methods in IntoIterator trait that converts an object to iterator:

into\_iter: taking the ownership of iterated elements

iter: immutable borrowing of iterated elements

iter\_mut: mutable borrowing of iterated elements

### The Iterator Trait

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

An object can become iterator if it implements the Iterator trait shown above. To implement this trait, we only need to implement the **next** method, which indicates how to retrieve values from the collection. All other methods have default implementations.

The **next** method returns an Option type, in which Some(Item) means the next value and None indicates reaching the end of iterator. Also, next consumes the values it iterates, meaning that after looping through the elements with next, there will be no more values in the iterator.

Here is an example of implementing Iterator in a custome struct:

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

### Iterator Consumer and Adapter

**Iterator consumers** are methods that invoke the next method in the iterator. Since next method takes the ownership of iterator elements, an iterator consumer would take the ownership of the iterator and return a value.

Here are a few examples of iterator consumers:

**sum:** sum method returns the sum of iterator elements.

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    let v1_iter = v1.iter()
    let total: i32 = v1_iter.sum()'
    assert_eq!(total, 6);
    println!("{:?}", v1);    // v1_iter borrows v1, so v1 can still be used
    println!(":?", v1_iter);    // error: v1_iter borrowed by sum
}
```

**collect:** collect method gathers the values from the interator into a collection data type. We need to specify the collection type we want to gather.

```rust
let v1: Vec<i32> = vec![1, 2, 3];
let v2: Vec<_> = v1.iter().collect();
```

**Iterator adapters** are methods that returns a new iterator from the iterator. Since iterator adapters are lazy, we need an iterator consumer in the end to get a specific result from the iterator

Here are examples of iterator adapters:

**1 map**: map takes a closure as a parameter and returns a new iterator containing the return values of the closure with elements in the original iterator as parameters.

```rust
let v1: Vec<i32> = vec![1, 2, 3];
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();
assert_eq!(v2, vec![2, 3, 4]);
```

**2 zip**: zip takes two iterators \[a1, a2, ...\] and \[b1, b2, ...\] with the same length and returns an iterator in the form \[(a1, b1), (a2, b2), ...\]

```rust
use std::collections::HashMap
fn main() {
    let names = ["A", "B"];
    let ages = [15, 16];
    let folks: HashMap<_, _> = names.into_iter().zip(ages.into_iter()).collect();
}
```

**3 filter**: filter takes a closure as parameter. The closure takes the iterator element as parameter and returns a bool. If the closure returns true, the element is kept in the new iterator, otherwise it is discarded.

```rust
struct Shoe {
    size: u32,
    style: String
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}
```

**4 enumerate**: enumerate returns an iterator containing (index, element) of the original iterator.

```rust
let v = vec![1u64, 2, 3, 4, 5, 6];
let val = v.iter()
            .enumerate()    // [(0, 1), (1, 2), (2, 3), (3, 4), (4, 5), (5, 6)]
            .filter(|&(idx, _)| idx % 2 == 0)    // [(0, 1), (2, 3), (4, 5)]
            .map(|(_, val)| val)    // [1, 3, 5]
            .fold(0u64, |sum, acm| sum + acm);    // 1 + 3 + 5 = 9
```