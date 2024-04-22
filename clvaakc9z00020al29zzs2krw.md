---
title: "PyTA Project: Astroid Abstract Syntax Tree Nodes"
datePublished: Mon Apr 22 2024 01:41:45 GMT+0000 (Coordinated Universal Time)
cuid: clvaakc9z00020al29zzs2krw
slug: pyta-project-astroid-abstract-syntax-tree-nodes
tags: python, compiler, pylint, abstract-syntax-tree

---

This blog covers reading notes of Astroid abstract syntax tree documentation and source code, including definitions of Const, BinOp, Call, List, Expr, If, For, FunctionDef nodes ([https://pylint.readthedocs.io/projects/astroid/en/latest/api/astroid.nodes.html](https://pylint.readthedocs.io/projects/astroid/en/latest/api/astroid.nodes.html))

### Const

Const node represents any literal value, including a number, string, boolean, None, etc. It has attributes **self.value**, representing the value, and **self.kind**, the string prefix

1. **bool\_value()**: returns bool(self.value)
    
2. **getitem()**: retrieve self.value by indexing or slicing. Raises errors if self.value does not support indexing or the index is out of range.
    
3. **has\_dynamic\_geattr():** if the node has `__getattr__` or `__getattribute__` . Always return False.
    
    Note: `__getattribute__` is called when an attribute of the class instance is being accessed. `__getattr__` is called when an attribute that does not exist is being called.
    
4. **itered():** returns an iterator over the elements if self.value is str. Otherwise self.value is not iterable and raises TypeError
    

### BinOp

BinOp node represents a binary operation between two expressions, stored as child nodes **self.left** and **self.right**. The operator, stored as **self.op**, includes "+", "-", "\*", "/", "//", "%", "\*\*"

1. **get\_children()**: get the two operands. Note that in the source code, since the method needs to return two values, it uses yield instead of return. yield is used to pass a value to the caller while retaining the state of the method, and can passes the next value when called again
    
    ```python
    def get_children(self):
        yield self.left
        yield self.right
    ```
    
2. **op\_left\_associative()**: return whether the operating is associative. In this case, only exponentiation operation (\*\*) is not associative.
    
3. **type\_errors()**: return a list of BadBinaryOperationMessage that includes errors being identified during inference
    

### Call

Call node represents a function call. It stores the function being called as **self.func**, as well as arguments (**self.args**) and keywords (**self.keywords**) passed to the function.

In python, function arguments are values being passed into the function by the caller. The arguments are passed in order by default. If the caller explicitly names the arguments being passed, such arguments are called **keyword arguments**. Keyword arguments does not need to following the order defined in the function.

```python
python
Copy code
def greet(name, message):
    print(f"Hello, {name}! {message}")

greet("Alice", "How are you?")    # function arguments
greet(message="How are you?", name="Bob")    # keyword arguments
```

Call node also supports starred arguments, stored in properties **starargs** and **kwargs**. Starred arguments and keywords are used to pass varying number of parameters into the function. Starred keywords receives a variable number of keywords and can be used as a dictionary, whose keys are argument names and values are argument values.

```python
def my_function(*args):
    for arg in args:
        print(arg)

my_function(1, 2, 3, 4)

def my_function(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")

my_function(name="Alice", age=30, city="New York")
```

### List

List node represents a list. The list elements are stored in **self.elt**. The node also stores a Context type called **self.ctx**, which represents the context in which is list is used. Context is **Store** if the list is being assigned to a variable, and **Load** otherwise.

1. **getitem()**: returns list item with a given index. This method is implemented by invoking the \_getitem() in parent class BaseContainer
    

### Expr

Expr node represents a python expression that has a return value, and is a child class of Statement. A python expression is wrapped in Expr if its return value is not used or stored. What the expression does is stored in **self.value()**. For example:

```python
print("a")
Expr(value=Call(
    func=Name(name="print"),
    arg=Const(value="a"),
    keywords=None
))

1 + 2
Expr(value=BinOp(
    op="+",
    left=Const(value=1),
    right=Const(value=2)
))
```

### If

If node represents an if statement. It contains **self.test**, a single node that should evaluate a boolean for if condition, **self.body**, a list of nodes to execute in the if branch, and **self.orelse**, a list of nodes in the else branch. There is no specific node for elif statement, as it is implemented as a nested if statement in the orelse branch.

1. **has\_elif\_branch()**: whether there is an elif branch after the if statement. This is implemented by checking whether the first node in orelse is an If node.
    
2. **block\_range()**: get the number of lines in a branch after a given line.
    

### For

For node represents a for loop statement. **self.target** holds a single node as the loop variable. **self.iter** refers to a node that is being iterated by the loop. **self.body** is a list of nodes being executed in the loop, and **self.orelse** holds the list of nodes for the else branch.

One interesting feature of python for loop is that it supports an else branch. The else branch is executed if the for loop finishes on its own, not interrupted by break statement. The else branch can be used to distinguishes different cases of returning from loop (whether it finishes natually, or finishes by a break statement). For example:

```python
def is_prime(num):
    if num <= 1:
        return False
    for i in range(2, num):
        if num % i == 0:
            # If num is divisible by any number in this range, it's not prime
            break
    else:
        # This else block runs if no `break` occurred in the for loop
        return True
    return False
```

The code above checks whether num is a prime number. It breaks the loop once a divisor is found and False is returned. If the break statement is not executed and the loop finishes, the else branch is executed and True is returned. This is equivalent with the code below:

```python
def is_prime(num):
    if num <= 1:
        return False
    for i in range(2, num):
        if num % i == 0:
            return False
    return True
```

### FunctionDef

FunctionDef node represents a function definition. It has the following attributes:

1. **self.name**, the function name as raw string (not a Const node).
    
2. **self.args**, the function arguments represented as an Arguments node.
    
3. **self.body**, a list of nodes to be executed in the function.
    
4. **self.decorators**, a list of decorators being applied to the function, in the order from outer to inner (the first decorator is applied last).
    
5. **self.returns**, the type annotation of function return value represented as a Name node.
    
6. **self.type\_params**, the type annotations for parameters.
    
7. **self.doc\_node**, the function docstring stored in a Const node.
    

Some of its public methods are listed below:

1. **argnames()**: iterates through self.args and return the name of every argument, including variable-length arguments like \*args and \*\*kwargs
    
2. **block\_range()**: get the number of lines in the node body starting from a given line.
    
3. **decoratornames()**: iterators through self.decorators and get the name of the function decorators
    
4. **display\_type()**: returns "Method" if the function is a class method, and "Function" otherwise. This method is internally implemented with type() method, which identifies the function as "function", "method", "classmethod" (including `__new__, __init_subclass__, __class_getitem__`), "staticmethod" based on multiple conditions, such as whether the parent node is ClassDef.
    
5. **is\_bound()**: whether the function is either a method or classmethod (bounded to a class)
    
6. **is\_abstract()**: whether the method is an abstract method. Even though python does not actually have "abstract class" as many other object-oriented languages, a method is considered as abstract if:
    
    1. Its only statement is `raise NotImplementedError`
        
    2. It raises any exceptions when any\_raise\_is\_abstract is True
        
    3. Its only statement is `pass` and pass\_is\_abstract is True