---
title: "Journey to PythonTA: The Control Flow Graph Module (Part 1)"
datePublished: Mon Oct 07 2024 22:17:59 GMT+0000 (Coordinated Universal Time)
cuid: cm1zkp93x00000ala42nghwx2
slug: journey-to-pythonta-the-control-flow-graph-module-part-1
tags: python, graph, pylint, static-analysis, control-flow-graph

---

The **Journey to PythonTA** series aims to introduce various system components of PythonTA ([https://github.com/pyta-uoft/pyta](https://github.com/pyta-uoft/pyta)), a static code analysis tool for checking common code style errors in Python code, to new developers. In this series, in addition to outlining the system structure, I will focus on explaining the underlying purposes and software design decisions beneath the codebase.

### What is Control Flow Graph

If you’re reading this article as an incoming developer at SDS Team, you’d probably be familiar with PythonTA through your first year Python courses. If not, it may be working trying out the basic features of PythonTA before proceeding. Whenever we want to investigate into the codebase of an open-source software, it’s always a good idea to be really familiar with what this software does, especially some lesser-used features and customizable configurations. For a mature project, oftentimes a large propertion of code deals with corner cases. If you do not understand what these cases are, it may not be easy to comprehend why these code exist even if you understand what they are doing.

With that being said, before we dive into the CFG modules, we should first learn about what Control Flow Graph is and how it is used in PythonTA both as an internal tool for certain checkers and a diagram tool for users. A **Control Flow Graph (CFG)** is a graph representation of a program to illustrate the flow of a program. It models all possible paths that can be taken through a piece of code, where nodes represent a blocks of code, and edges represent the transitions between these blocks based on control statements like loops and conditionals.

There are only three based control flows for any programs (ever since GOTO is considered harmful): **sequential statements**, **conditional statements**, and **loops**. A **basic block** in a control flow graph (CFG) represents a series of statements that will always execute sequentially, one after the other. A block may consist of a single line of code or many lines, but as long as there are no conditional statements or loops within the block, the execution order is guaranteed.

When the program encounters an `if` statement, the control flow graph **branches** into two paths. One path is followed if the condition evaluates to `True`, and the other if the condition evaluates to `False`. After the corresponding branch is executed, the program reaches an **end node** (often represented as a black block) where the program control ends, typically following a `return` statement.

```python
def func(x: int):
    if x > 5:
        return x
    else:
        return 0
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1728067837291/70f79d61-f259-4758-bc9a-3a9c9c47f15a.png align="center")

In this case, even if you remove the `else` block and put `return 0` after the `if` statement, the structure of the control flow graph remains the same. This is because the flow of execution is still divided by the condition, and the outcome remains the same.

```python
def func(x: int):
    if x > 5:
        return x
    return 0
```

If there is **no return statement** within the `if` block, the structure of the CFG changes. Now, regardless of whether the condition is `True` or `False`, the control will eventually proceed to the code following the `if` statement. This represents a different control flow, as the code after the condition will always be executed.

```python
def func(x: int):
    if x > 5:
        x -= 1
    return x
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1728070642219/ca5d2c71-aed3-40e6-b1f2-c111236bb9de.png align="center")

In a **while loop**, the control flow graph diverges at the loop condition. If the condition evaluates to `True`, the flow proceeds to the loop body, with an edge leading back to the condition after each iteration. This indicates that the loop condition will be checked again after the loop body is executed. When the condition eventually evaluates to `False`, the control flows to the code following the `while` loop.

```python
def func(x: int):
    while x > 5:
        x -= 1
        print(x)
    return x
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1728306988269/4870d4a6-98b1-427b-8f67-441a9ec203c3.png align="center")

A **for loop** behaves similarly to a `while` loop,except that the condition node in the CFG represents the **loop target** **variable**. Like the `while` loop, the flow returns to this target node after each iteration until the loop completes, after which control proceeds to the next block.

```python
def func(x: int):
    for i in range(0, 5):
        print(x)
    print("done")
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1728307254200/c8b50c58-fdc1-4178-bc5e-a9debde3bb73.png align="center")

There are four types of **jump statements** that alter the standard control flow in CFGs:

* `break` and `continue` within loops,
    
* `return` and `raise` within functions.
    

These statements immediately change the flow of execution by exiting a loop (`break`), skipping to the next iteration (`continue`), or terminating a function early (`return` and `raise`).

Since the program structure is naturally recursive, more complex compound statements can still be represented a nesting of simple control flow structures:

```python
def func(x: int):
    while x > 5:
        x -= 1
        if x % 2 == 0:
            continue
        print(x)
    print("end")
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1728308595589/14b6347c-6229-47ca-a8d4-13d41e91f977.png align="center")

### How Control Flow Graph is Used in PythonTA

One of the most prominent features of the Control Flow Graph (CFG) module is its ability to display a **visual diagram** of a program’s CFG, making it easier for users to understand the program's structure and flow.

For detailed information about installing dependencies and using CFG in code, refer to the [PythonTA official documentation](https://www.cs.toronto.edu/~david/pyta/cfg/index.html).

To **generate a CFG diagram** from the command line, follow these steps:

1. Navigate to the `pyta` (root) directory.
    
2. Use the `draw_cfg.py` module located in the `examples/sample_usage` directory.
    

Run the following command in your terminal at `pyta` directory, replacing `<your-file.py>` with the path to your Python file:

> python -m examples.sample\_usage.draw\_cfg “&lt;your-file.py&gt;“

In addition to visualizing control flow graphs, the CFG module serves as an **internal tool** for various custom checkers that detect logical errors in a program’s control flow. As of the latest update (Version 2.8.1), the CFG module is integrated with the following checkers:

1. `one_iteration_checker`  
    This checker ensures that loops are not prematurely terminated in only one iteration. A loop might exhibit this behavior if it contains a `break` or `return` statement without a condition, forcing the loop to exit early. The checker works by examining whether there is a cycle in the control flow graph that leads back to the loop condition. If no such cycle exists, the loop is flagged as terminating in one iteration.
    
2. `inconsistent_or_missing_return_checker`  
    This checker detects inconsistent return statements in functions. A function may return a value in some branches but only has `return` in others (by PEP8 standard, an explicit `return None` is required), or it may be missing a return statement altogether in at least one branch. The checker identifies these inconsistencies by ensuring that all blocks leading to the end block contain a valid return statement.
    
3. `possibly_undefined_checker`  
    This checker identifies variables that might be used before they are defined. It traces back through the control flow graph from the point where a variable is used and checks whether every possible execution path includes an assignment to that variable. If an assignment is missing on any path, the variable is considered potentially undefined.
    
4. `redundant_assignment_checker`  
    This checker flags unnecessary variable assignments. An assignment is deemed redundant if there is a reassignment of the same variable before its next use on all possible execution paths. The checker verifies whether a new assignment occurs between the original assignment and the variable usage across all execution branches.
    

In the next article, we will dive into the control flow graph module and get to know the system components in general.