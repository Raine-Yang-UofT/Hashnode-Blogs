---
title: "Rust Learning Note: Global Variable"
datePublished: Wed Dec 27 2023 15:20:26 GMT+0000 (Coordinated Universal Time)
cuid: clqnxaigr000008jv22xuc7k3
slug: rust-learning-note-global-variable
tags: rust, variables-and-constants, static-variable, global-variables, atomic-type

---

This article is a summary of Chapter 4.7 of Rust Course ([course.rs/](https://course.rs/))

### Global Variables Initialized at Compile Time

**Static Constant**

**Static constants** are global constants that can be used anywhere in the program, regardless of the scope in which it is defined, since the lifecycle of a constant is always 'static. However, different references to the same constant do not guarantee the same memory address.

```rust
const MAX_ID: usize = usize::MAX / 2;
fn main() {
    println!({MAX_ID});
}
```

Here are a few things to note about static constants:

1. We use **const** instead of **let** to define a constant
    
2. Type annotations for constants cannot be omitted
    
3. A constant must be assigned to a specific value or an expression that can be evaluated at compiler time.
    
4. There cannot be duplicate constant names
    

**Static Variable**

Static variables are global variables that can be accessed throughout the program. A static variable must be accessed and mutated in **unsafe** block, since it is unsafe especially in multithreading. A static variable must be defined as a value known in compile time, and the value must implement **Sync** trait. Any references to a static variable always yield the same memory address.

```rust
static mut REQUEST_RECV: usize = 0;
fn main() {
    unsafe {
        REQUEST_RECV += 1;
    }
}
```

**Atomic Type**

If we want to use a static variable safely in multithreading, we can apply atomic types:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
static REQUEST_RECV: AtomicUsize = AtomicUsize::new(0);
fn main() {
    for _ in 0..100 {
        REQUEST_RECV.fetch_add(1, Ordering::Relaxed);
    }
}
```

Here an example global ID generator with global variables mentioned above:

```rust
use std::sync::atomic::{Ordering, AtomicUsize};

struct Factory {
    factory_id: usize,
}

static GLOBAL_ID_COUNTER: AtomicUsize = AtomicUsize::new(0);
const MAX_ID: usize = usize::MAX / 2;

fn generate_id() -> usize {
    let current_val = GLOBAL_ID_COUNTER.load(Ordering::Relaxed);
    if current_val > MAX_ID {
        panic!("Factory ids overflowed");
    }
    GLOBAL_ID_COUNTER.fetch_add(1, Ordering::Relaxed);
    let next_id = GLOBAL_ID_COUNTER.load(Ordering::Relaxed);
    if next_id > MAX_ID {
        panic!("Factory ids overflowed");
    }
    next_id
}

impl Factory {
    fn new() -> Self {
        Self {
            factory_id: generate_id()
        }
    }
}
```

### Global Variable Initialized at Runtime

**Lazy\_static**

The global variables above can only be initialized at compile time. However, sometimes we want to initialize global variables at runtime, such as creating a global Mutex lock. In this case, we can use macro **lazy\_static**:

```rust
use std::sync::Mutex
use lazy_static::lazy_static;
lazy_static! {
    static ref NAMES: Mutex<String> = Mutex::new(String::from("Sunface, Jack, Allen"));
}

fn main() {
    let mut v = NAMES.lock().unwrap();
    v.push_str(", Myth");
    println!("{}", v);
}
```

lazy\_static can also be used to load a global configuration that is initialized only after the program starts. Here's an example of implementing a global cache with lazy\_static:

```rust
use lazy_static::lazy_static;
use std::collections::HashMap;

lazy_static! {
    static ref HASHMAP: HashMap<u32, &'static str> = {
        let mut m = HashMap::new();
        m.insert(0, "foo");
        m.insert(1, "bar");
        m.insert(2, "baz");
        m
    };
}

fn main() {
    println!("The entry for 0 is {}", HASHMAP.get(&0).unwrap());
    println!("The entry for 1 is {}", HASHMAP.get(&1).unwrap());
}
```

In the code above, HASHMAP is initialized only when it is first invoked at the first line in main method.

**Box::leak**

**Box::leak** returns a variable with a 'static lifecycle. It can be used for reassignment of static variable. It is not permitted to reassign a global variable to a value of a local variable, since the local variable has a shorter lifecycle. However, we can achieve this with Box::leak.

```rust
#[derive(Debug)]
struct Config {
    a: String,
    b: String
}

static mut CONFIG: Option<&mut Config> = None;

fn main() {
    let c = Box::new(Config {
        a: "A".to_string(),
        b: "B".to_string()
    });

    unsafe {
        CONFIG = Some(Box::leak(c));
        println!("{:?}", CONFIG);
    }
}
```

Using Box::leak, we can also return a global cariable from a function:

```rust
#[derive(Debug)]
struct Config {
    a: String,
    b: String
}

static mut CONFIG: Option<&mut Config> = None;

fn init() -> Option<&'static mut Config> {
    let c = Box::new(Config {
        a: "A".to_string(),
        b: "B".to_string()
    });
    Some(Box::leak(c))
}

fn main() {
    unsafe {
        CONFIG = init();
        println!("{:?}", CONFIG)
    }
}
```