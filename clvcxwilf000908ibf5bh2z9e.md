---
title: "PyTA Project: Implement a custom pylint checker for PythonTA"
datePublished: Tue Apr 23 2024 22:10:36 GMT+0000 (Coordinated Universal Time)
cuid: clvcxwilf000908ibf5bh2z9e
slug: pyta-project-implement-a-custom-pylint-checker-for-pythonta
tags: python, pylint, visitor-pattern

---

In this article, I will provide an overview of how Pylint static code analysis works, provide an implementation of a custom pylint checker following the example in Pylint Documentation ([https://pylint.pycqa.org/en/latest/development\_guide/how\_tos/custom\_checkers.html](https://pylint.pycqa.org/en/latest/development_guide/how_tos/custom_checkers.html)), and finally demonstrate how to integrate the custom checker into PythonTA ([https://github.com/pyta-uoft/pyta](https://github.com/pyta-uoft/pyta))

### How Pylint Works

Pylint is an open-source code linter that can detect errors and violations of coding conventions based on a list of pre-defined rules (known as checkers). As a static code analysis tool, it makes detections without executing the code being examined. Instead, it constructs an **abstract syntax tree (AST)** from the source code, traverses the AST in depth-first order, and applies checkers on the AST to detect problems and give refactoring advice.

The interactions between the AST being analyzed and the checkers follow the **visitor pattern**. Visitor pattern places the behaviors to be added into a new set of visitor classes (in this case, the checkers), and passes the objects being visited (AST nodes) as parameters into corresponding callback methods in the visitors. In this way, we do not need to make any changes to our source code to make it analyzible by Pylint. For more information on the visitor pattern, check ([https://refactoring.guru/design-patterns/visitor](https://refactoring.guru/design-patterns/visitor))

There are two types of callback functions for a checker that allows the checker to access the AST node being traversed: `visit_<node_name>(self, node)` is called when an AST node with &lt;node\_name&gt; is being visited, and `leave_<node_name>(self, node)` is called when Pylint leaves &lt;node\_name&gt;. For example, for FunctionDef node (which represents a function definition with `def` keyword), `visit_functiondef(self, node)` and `leave_functiondef(self, node)` are called when Pylint reaches and leaves a FunctionDef node during its traverse.

### Implement a Custom Checker

Following the example from pylint documentation, we will implement a custom checker that raises a warning when multiple return statements in a function have the same return value.

```python
# module name: unique_return_checker.py
from astroid import nodes
from typing import Optional

from pylint.checkers import BaseChecker
from pylint.lint import PyLinter


class UniqueReturnChecker(BaseChecker):

    name = "unique-returns"
    msgs = {
        "W0001": (    # message id
            "Returns a non-unique constant.",    # displayed message
            "non-unique-returns",    # message symbol
            "All constants returned in a function should be unique.",    # message description
        ),
    }
    options = (
        (
            "ignore-ints",
            {
                "default": False,
                "type": "yn",
                "metavar": "<y or n>",
                "help": "Allow returning non-unique integers",
            },
        ),
    )
```

First, a custome check needs to inherit BaseChecker class and provides instantiations of the following attributes: name, msgs, and options. **name** is used to generate a specia configuration when **options** is provided. **options** are optional configurations that uses can set. In this case, the option ignore\_ints determines whether we allow duplicate integer return values.

**msgs** is the message being outputed by the checker. It is a dictionary where the key represents the **message id**, starting with one of C, W, E, F, R, indicating "convention", "warning", "error", "fatal", "refactoring" respectively. The following 4 digits should be unique for each message and the first two digits should be the same among messages defined in one checker (except for shared messages). The first and third value entries are **displayed-message** and **message-description**, which correspond to the message title and description displayed for users. The second entry **message-symbol** represents a alias of the message id that can be used interchangeably with the id.

```python
    def __init__(self, linter: Optional[PyLinter] = None) -> None:
        super().__init__(linter)
        self._function_stack = []

    def visit_functiondef(self, node: nodes.FunctionDef) -> None:
        self._function_stack.append([])

    def leave_functiondef(self, node: nodes.FunctionDef) -> None:
        self._function_stack.pop()

    def visit_return(self, node: nodes.Return) -> None:
        if not isinstance(node.value, nodes.Const):
            return
        for other_return in self._function_stack[-1]:
            if node.value.value == other_return.value.value and not(
                self.linter.config.ignore_ints and node.value.pytype() == int
            ):
                self.add_message("non-unique-returns", node=node)

        self._function_stack[-1].append(node)
```

`visit_functiondef()` and `leave_functiondef()` are called when Pylint accesses and leaves a FunctionDef node. In order to handle nested function definitions, we creates a stack for each FunctionDef node, pushes a new list into the stack in visit\_functiondef() to store its return value, and pops the list after finishing travsering the function.

`visit_return()` is called when Pylint accesses a Return node, which represent a return statement. In this method, if the return statement (stored in node.value) is not a literal value (represented as a Const node), we stops the checking since the return value is a expression that requries further evaluations. Otherwise, we adds the return value into the list, and check whether it is equal to a previous return value (also handling the case when ignore\_ints option is True and the return values are integers). If duplicate return values are detected, the error message is emitted through `self.add_message("non-unique-returns", node=node)` .

```python
def register(linter: PyLinter) -> None:
    linter.register_checker(UniqueReturnChecker(linter))
```

Finally, we need a **register()** function in the module and outside of UniqueReturnChecker class to register the checker into Pylint. The register() function will be automatically detected and invoked by Pylint.

Here is the complete code for the checker:

```python
from astroid import nodes
from typing import Optional

from pylint.checkers import BaseChecker
from pylint.lint import PyLinter


class UniqueReturnChecker(BaseChecker):

    name = "unique-returns"
    msgs = {
        "W0001": (
            "Returns a non-unique constant.",
            "non-unique-returns",
            "All constants returned in a function should be unique.",
        ),
    }
    options = (
        (
            "ignore-ints",
            {
                "default": False,
                "type": "yn",
                "metavar": "<y or n>",
                "help": "Allow returning non-unique integers",
            },
        ),
    )

    def __init__(self, linter: Optional[PyLinter] = None) -> None:
        super().__init__(linter)
        self._function_stack = []

    def visit_functiondef(self, node: nodes.FunctionDef) -> None:
        self._function_stack.append([])

    def leave_functiondef(self, node: nodes.FunctionDef) -> None:
        self._function_stack.pop()

    def visit_return(self, node: nodes.Return) -> None:
        if not isinstance(node.value, nodes.Const):
            return
        for other_return in self._function_stack[-1]:
            if node.value.value == other_return.value.value and not(
                self.linter.config.ignore_ints and node.value.pytype() == int
            ):
                self.add_message("non-unique-returns", node=node)

        self._function_stack[-1].append(node)


def register(linter: PyLinter) -> None:
    linter.register_checker(UniqueReturnChecker(linter))
```

### Integrating with PythonTA

To add the checker into PythonTA, simple put the module under the directory `pyta\python_ta\checkers` . Here is a test program and the corresponding PythonTA output:

```python
def fun_test():
    if True:
        return 5
    return 5


def fun_test2():
    if True:
        return 1
    return 5


if __name__ == '__main__':
    import python_ta
    python_ta.check_all()
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1713910183026/d8154391-54d3-4fdf-a8da-74e309c89a26.png align="center")