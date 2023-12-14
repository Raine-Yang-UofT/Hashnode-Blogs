---
title: "Rust Learning Note: Generics and Traits"
datePublished: Thu Dec 14 2023 20:37:10 GMT+0000 (Coordinated Universal Time)
cuid: clq5nvrgf000608jnf0icauxd
slug: rust-learning-note-generics-and-traits
tags: rust, traits, generics, trait-object, static-dispatch, dynamic-dispatch

---

This blog is a summay of Chapter 2.8 of Rust Course ([**course.rs**](https://course.rs/))

### Generics

Generics can be used in structs, enumerators, and methods, as shown in the example below:

**1 Using generics in structs:**

```rust
struct Point<T> {
    x: T,
    y: T
}

fn main() {
    let integer = Point {x: 5, y: 10};
    let float = Point {x: 1.0, y: 5.0};
}
```

In thie code, we declare a generic parameter T for struct Point, and assigns variables x, y to be type T. When we instantiate the generic struct, we can make T to be type i32 or f64.

It is worth noting that generics in Rust are achieve in compile time, meaning that during compile time the generic type T is converted to a concrete type like i32 or f64, a process known as **monomorphization**. As a result, one generic variable can only represent one data type, so it would cause an error if the x and y in Point have differen types, for example

```rust
let p = Point {x: 1, y: 1.0};    // ERROR
```

A solution to this is to assign different generic variables for each type, for instance:

```rust
struct Point<T, U> {
    x: T,
    y: U
}
```

**2 Using generics in enumerator**

The most notable use of generics in enumerator in Rust is probably the Option enumerator used to indicate whether a value exists, which has elements Some(T) (with value), and None (without value).

```rust
enum Option<T> {
    Some(T),
    None
}
```

Similarly, enumerator Result is used to show whether a value is correct, and can provide different output for correct and incorrect values.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E)
}
```

**3 Using generics in methods:**

We can also use generics in methods:

```rust
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

In the code above, we first need to define genric parameter T in impl&lt;T&gt; before using it. The Point&lt;T&gt; here is not a generic variable declaration, but the name of the struct we defined earlier.

We can also add additional generic parameters in the functions in addition to generic variables in the struct:

```rust
impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y
        }
    }
}
```

We can also implement methods for a specific type in the generic struct. The code below defines a method for only a specific type Point&lt;f32&gt;, instead of the generic Point&lt;T&gt;:

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

### const Generics

Generics is used to represent different data types, and **const generics** is used to represent different values. For example, if we want to write a function that works for arrays with any given length, we can use a const generic variable to define the input array length.

```rust
fn display_array<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32, 3] = [1, 2, 3];
    display_array(arr);
    let arr: [i32; 2] = [1, 2];
    display_array(arr);
}
```

### Trait

Trait in Rust is similar to interface in many other languages. Trait is used to provide an abstraction for a certain behavior.

**1 implement trait for struct:**

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct Post {
    pub title: String,
    pub author: String,
    pub content: String
}

impl Summary for Post {
    fn summarize(&self) -> String {
        format!("Post{}, author{}", self.title, self.author)
    }
}

pub struct Comment {
    pub username: String,
    pub content: String
}

impl Summary for Comment {
    fn summarize(&self) -> String {
        format!("{} comment: {}", self.username, self.content)
    }
}
```

In the case above, we define a trait Summary and define method summarize in the trait. Every struct that implements Summary trait must provide an implementation of summarize method. The two struct, Post and Comment, both implement Summary trait and provide their respective implementation of summarize method.

**2 Default Implementation:**

Instead of only providing the method signiture, we can also provide a default implementation of a trait in the trait definition. The structs that implement a trait need to provide implementations for traits without default implementation. Of course, they can also override the default implementation.

```rust
// a default implementation of trait
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

It is also allowed for a default implementation to invoke another method in trait, even if that method has no default implementation. In this way, we can reuse some parts of functionality and override other parts in the trait, as in this example:

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}

impl Summary for Comment {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```

Many built-in traits have default implementations. Oftentimes, we can simply add these traits to our types and use these default functions. This is done through **derive** keyword. For instance, **#\[derive(Debug)\]** allows printing a struct object with println!("{:?}")

### Orphan Rules for Traits

**Orphan rules** refer that if we want to implement trait T for type A, at least one of A and T is defined in the current scope. For example, we can implement trait Display (defined in standard library) for type Post (defined locally), but we cannot implement trait Display for type String (also defined in standard library) in our code.

A workaround of this rule is **newtype pattern**. Newtype encapsulates the type in a local struct, so the type is defined locally now:

```rust
use std::fmt
struct Wrapper(Vec<String>);
impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}
```

A drawback of newtype is that we always need to invoke self.0 to retrieve the String inside. A solution to that is to derive a trait called Deref that can convert Weapper to Vec&lt;String&gt;

### Use Trait as Function Parameters and Return Values

**1 Use trait as function parameter:**

We can use trait as data type for function parameters. For example, the parameter &item in the code below means "any item that implements Summary trait"

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

This code above is in fact just a syntactic sugar for trait bound. Trait bound refers to adding additional requirements on generic variables that require the variable to implement certain traits.

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}". item.summarize());
}
```

We can also add multiple trait bounds:

```rust
pub fn notify(item: &(impl Summary + Display)) {}
pub fn notify<T: Summary + Display>(item: &T) {}
```

If we have multiple trait bounds, we can use **where** keyword to simplify the format

```rust
fn function<T, U>(t: &T, u: &U) -> i32 
    where T: Display + Clone,
          U: Clone + Debug
{}
```

**2 Use trait as function return value:**

**impl Trait** can also be used as type annotation for function return values:

```rust
fn return_summarizable() -> impl Summary {
    Comment {...}    // omit attributes
}
```

However, this approach has a limitation. It can only return a specific data type that implements Summary. In order to return multiple types that implements a trait, such as returning either Comment or Post in the code above, we need to use trait object.

### Trait Object

Consider the case when we have a trait Draw and mutiple types that all implement draw. We want a function to accept parameters of any type that implements Draw method. This can be achieved through trait object:

```rust
trait Draw {
    fn draw(&self) -> String;
}

fn draw1(x: Box<dyn Draw>) {
    x.draw();
}

fn draw2(x: &dyn Draw) {
    x.draw();
}
```

We define trait object with keyword **dyn**. In the case above there are two implementations: using Box and reference. Note that we cannot use trait object itself as the parameter or return value since the exact size of a trait object is unknown at compile time. We can only use a pointer (which has a known size) to the object.

### Static Dispatch and Dynamic Dispatch

**Static dispatch** refers to determining the data type during the compile time. As mentioned earlier, generics in Rust uses static dispatch since generic types are converted to concert types during compile type.

**Dynamic dispatch** refers to determining the data type at runtime. Trait object is a type of dynamic dispatch, since the compiler cannot know all possible data types that will be passed as trait objects in advance. Each pointer to a trait object contains two information: a pointer to the object being referred to and a pointer to virtual function table indicating the object's implementations of the trait.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702585084284/3baa6794-2bc8-45b8-bbb6-1d07474132bf.png align="center")

Fig 1. Static dispatch and dynamic dispatch. Image reproduced from (Rust Course)

Since the actual data type of the trait object is unknown, we cannot use methods in the data type other than methods defined in the trait. Also the trait must satisfy **object security**, including the following requirements:

1 The method return type cannot be Self: Self refers to the type of the object. However, the type is unknown here.

2 The method cannot contain generic parameters: Similarly, generics requries knowledge of the data type due to static dispatch.