---
title: "PyTA Project: Implement a Custom Checker for Inconsistent or Missing Returns"
datePublished: Wed Jun 19 2024 02:20:56 GMT+0000 (Coordinated Universal Time)
cuid: clxl7i52t000009mg4aamftek
slug: pyta-project-implement-a-custom-checker-for-inconsistent-or-missing-returns
tags: unit-testing, python, pylint, abstract-syntax-tree, control-flow-graph

---

Today's task is to implement a custom checker for inconsistent or missing return statements for PythonTA ([**https://github.com/pyta-uoft/pyta**](https://github.com/pyta-uoft/pyta)), as a replacement of pylint's R1710 checker. A basic introduction of how to create a custom pylint checker can be found in this previous post ([https://raineyang.hashnode.dev/pyta-project-implement-a-custom-pylint-checker-for-pythonta](https://raineyang.hashnode.dev/pyta-project-implement-a-custom-pylint-checker-for-pythonta)). For this task, we want to base our implementation on PythonTA's control flow graph introduced in my last article ([https://raineyang.hashnode.dev/pyta-project-depth-first-search-for-control-flow-graph](https://raineyang.hashnode.dev/pyta-project-depth-first-search-for-control-flow-graph)), and provide custom message rendering in the PythonTA reports. Specifically, we want to highlight the function where the return statement is missing along with the line where the user should insert the return statement.

### Pylint R1710 Checker

Pylint message `inconsistent-return-statements (R1710)` actually checks for two related errors: firstly, whether a return statement is missing in certain branches of the function, and secondly, if there are branches that have return values, whether branches without return values explicitly `return None`.

The first case is easy to understand. For example, these are functions that are missing return statements in at least one branches:

```python
def missing_return() -> int:
    print("no return")

def missing_return_in_branch() -> int:
    a = 1
    if a > 3:
        print("no return")
    else:
        return a
```

The second case means that, if the return type of a function is not `None` (even if it's `Optional[T]` or `None | T`), the branches that are intended to have no return values must explicit write `return None` in the end, instead of writing just `return` or having no return statements at all, as in the code below:

```python
def inconsistent_return() -> int:
    a = 1
    if a > 2:
        return 2
    return
```

### Defining Custom Error Messages

The first step of our implementation is to disable the original R1710 checker. However, pylint only allows us to disable a certain error message instead of a checker. We can edit the configuration file `.pylintrc` and add `R1710` to the list `disable`

Since we want to handle inconsistent returns and missing returns differently during the message rendering, we need to seperate these two cases to two messages. The first message we define, R9710, is an exact copy of pylint's R1710 with same message description. This message is intended to report inconsistent return statements. Note that although R1710 has been disabled, its message symbol `inconsistent-return-statements` still cannot have duplicates, so we can only name our message something similar, like `inconsistent-returns` . The second message, R9711 `missing-return-statements` , is used to indicate missing returns.

```python
    name = "inconsistent-or-missing-returns"
    msgs = {
        # as R1710 is being disabled, we replace it with an identical message
        "R9710": (
            """This function has at least one case where the function body will execute without ending in an explicit return statement.
            You should check your code to make sure every possible execution path through the function body ends in a return statement.
            Note: one common source of this error is if you're using if statements without an explicit else branch.
            In this case, you should consider revising your code to either add an else branch,
            or, if you are confident that the if and elif conditions cover all possible cases,
            "you can convert the final "elif " into an " else ".",""",
            "inconsistent-returns",
            "Used to replace R1710 message inconsistent-return-statements",
        ),
        "R9711": (
            "Missing return statement in function",
            "missing-return-statements",
            "Used when a function does not have a return statement and whose return type is not None",
        ),
    }
```

### Implementing the Checker

The checking is triggered when visiting a function definition. The general idea for the checker is that it first checks whether the return type annotation (`node.returns` attribute) is empty or set to None, which, in this case, the checking would be skipped for the function. Then it creates a control flow graph out of the function, get the `end` block of the graph, and traverses through statements in all blocks connected to `end` (since there cannot be code blocks after the return statements in a function). If a code block does not contain a return statement, a `missing-return-statements` is reported. If a return statement does not have a return value (`statement.value is None`), a `inconsistent-returns` is reported. The algorithm itself seems straightforward. However, there are a few implementation details worth noting, which I will example below.

```python
    def visit_functiondef(self, node) -> None:
        """Visit a function definition"""
        self._check_return_statements(node)

    def _check_return_statements(self, node) -> None:
        """
        Construct a CFG from the function. Check for inconsistent returns if there are
        multiple return statements, and missing return statements if there are none.
        """
        if (
            node.returns is None
            or isinstance(node.returns, nodes.Const)
            and node.returns.value is None
        ):
            return

        # get the end of CFG
        cfg = ControlFlowGraph()
        cfg.start = node.cfg_block
        # The last element of cfg.get_blocks_postorder() does not guarantee to be the end block.
        # However, based on the initialization of CFG, end block must have id == 1
        end = [block for block in cfg.get_blocks_postorder() if block.id == 1][0]
        end_blocks = [edge.source for edge in end.predecessors]

        # gather all return statements
        for block in end_blocks:
            has_return = False  # whether a return statement exists for this branch
            for statement in block.statements:
                if isinstance(statement, nodes.Return):
                    has_return = True
                    if statement.value is None:
                        # check for inconsistent returns
                        self.add_message("inconsistent-returns", node=statement)

            # check for missing return statement
            if not has_return:
                # for rendering purpose, the line is set to the last line of the function branch where return statement is missing
                self.add_message(
                    "missing-return-statements", node=node, line=block.statements[-1].fromlineno
                )
```

In checking the function's return type annotation, the annotation is represented as an AST node. Thus, if the type annotation is `None`, it is represented as a `Const` node whose `value` is `None`, instead of a "None" literal.

A tricky part of the algorithm is to get the `end` node of the control flow graph. We can get all blocks in post-order with `get_block_postorder()` method for `ControlFlowGraph` . In my previous implementation, I simply retrieved the first element of `get_block_postorder()` as the end block. However, that works incorrectly as when dealing with `for` loops and `while` loops, the check would sometimes mark blocks inside the loop as missing return statements.

The reason is that `get_block_postorder()` is implemented with a depth-first search from the `start` node of the graph that returns nodes in a reversed order. However, the "end" of the graph could be either the actual `end` node or the block inside a loop, as the searching never travels back from a cycle to prevent infinite recursions. As a result, the first element of `get_block_postorder()` does not guarantee to be the `end` node.

```python
    def get_blocks_postorder(self) -> Generator[CFGBlock, None, None]:
        """Return the sequence of all blocks in this graph in the order of
        a post-order traversal."""
        yield from self._get_blocks_postorder(self.start, set())

    def _get_blocks_postorder(self, block: CFGBlock, visited) -> Generator[CFGBlock, None, None]:
        if block.id in visited:
            return

        visited.add(block.id)
        for succ in block.successors:
            yield from self._get_blocks_postorder(succ.target, visited)

        yield block
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718759261386/d64eab66-00ec-4ecb-8955-79c14b5cef01.png align="center")

The solution is to identify the `end` node based on the id. In control flow graph, every node has a unique id, and based on the constructor method of `ControlFlowGraph` , we can ensure that the `start` and `end` nodes, being first and second created, must have id 0 and 1. Thus, we can find the `end` node by id 1.

```python
# in python_ta/cfg/graph.py    
def __init__(self, cfg_id: int = 0) -> None:
        self.block_count = 0
        self.cfg_id = cfg_id
        self.unreachable_blocks = set()
        self.start = self.create_block()
        self.end = self.create_block()
```

Here's the complete checker code:

```python
"""
Check for inconsistent return statements in functions and missing return statements in non-None functions.
"""

from typing import Optional

from astroid import nodes
from pylint.checkers import BaseChecker
from pylint.lint import PyLinter

from python_ta.cfg import ControlFlowGraph


class InconsistentReturnChecker(BaseChecker):

    name = "inconsistent-or-missing-returns"
    msgs = {
        # as R1710 is being disabled, we replace it with an identical message
        "R9710": (
            """This function has at least one case where the function body will execute without ending in an explicit return statement.
            You should check your code to make sure every possible execution path through the function body ends in a return statement.
            Note: one common source of this error is if you're using if statements without an explicit else branch.
            In this case, you should consider revising your code to either add an else branch,
            or, if you are confident that the if and elif conditions cover all possible cases,
            "you can convert the final "elif " into an " else ".",""",
            "inconsistent-returns",
            "Used to replace R1710 message inconsistent-return-statements",
        ),
        "R9711": (
            "Missing return statement in function",
            "missing-return-statements",
            "Used when a function does not have a return statement and whose return type is not None",
        ),
    }

    def __init__(self, linter: Optional[PyLinter] = None) -> None:
        super().__init__(linter=linter)

    def visit_functiondef(self, node) -> None:
        """Visit a function definition"""
        self._check_return_statements(node)

    def _check_return_statements(self, node) -> None:
        """
        Construct a CFG from the function. Check for inconsistent returns if there are
        multiple return statements, and missing return statements if there are none.
        """
        if (
            node.returns is None
            or isinstance(node.returns, nodes.Const)
            and node.returns.value is None
        ):
            return

        # get the end of CFG
        cfg = ControlFlowGraph()
        cfg.start = node.cfg_block
        # The last element of cfg.get_blocks_postorder() does not guarantee to be the end block.
        # However, based on the initialization of CFG, end block must have id == 1
        end = [block for block in cfg.get_blocks_postorder() if block.id == 1][0]
        end_blocks = [edge.source for edge in end.predecessors]

        # gather all return statements
        for block in end_blocks:
            has_return = False  # whether a return statement exists for this branch
            for statement in block.statements:
                if isinstance(statement, nodes.Return):
                    has_return = True
                    if statement.value is None:
                        # check for inconsistent returns
                        self.add_message("inconsistent-returns", node=statement)

            # check for missing return statement
            if not has_return:
                # for rendering purpose, the line is set to the last line of the function branch where return statement is missing
                self.add_message(
                    "missing-return-statements", node=node, line=block.statements[-1].fromlineno
                )


def register(linter: PyLinter) -> None:
    linter.register_checker(InconsistentReturnChecker(linter))
```

### Implementing Custom Rendering in node\_printer.py

`node_printer.py` module handles the rendering of code snippts in PythonTA's report. By default, the line that contains the error is highlighted. For `missing-return-statements` message, we want to customize the rendering to display for useful information for the users:

1. Display the function defintion at the front, which helps pinpoint the function that contains the error.
    
2. Display two lines of code before the missing return statement. If there are more previous lines in the function, a commit `"""MORE CODE OMITTED"""` is shown.
    
3. Highlight the location where the return statement is missing with comment `"""MISSING RETURN STATEMENT"""`
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718761743945/97523ebb-7739-429a-9da2-7cbef67f6b06.png align="center")

In order to achieve this effect, we need to somehow passes the location of the function definition and the missing return statement in the error message, which is implemented at this part in the checker:

```python
# for rendering purpose, the line is set to the last line of the function branch where return statement is missing
self.add_message(
    "missing-return-statements", node=node, line=block.statements[-1].fromlineno
)
```

the `node` argument indicates the AST node that contains the error, and `line` specifies the line where this error occur. By default, `line` uses the line number of `node` , but we can specify a different line number. Here I passes the `functiondef` node as `node` and the last line of the function branch as `line`.

With this information received, we can begin our implementation of the custom rendering. A helper function provided in this module, `render_context()` , displays code snippet between two given lines, providing a convenient way to display the context around the error for users to better locate.

We start by showing the function header by rendering `_node.fromlineno` , the line number of the beginning of the function, and then decide whether inserting `"""MORE CODE OMITTED"""` comment based on the number of lines between the function header and the line for missing return. Following it are two lines of context before the error line. After that, we calculate the indentation of the error line, and display `"""MISSING RETURN STATEMENT"""` after the error line with same indentation. The message is ended with one line of context after the error line.

```python
def render_missing_return_statements(msg, _node, source_lines=None):
    """
    Render a missing return statements message
    """
    line = msg.line
    end = _node.tolineno

    # render function header and context
    if line - 1 > _node.fromlineno:
        yield from render_context(_node.fromlineno, _node.fromlineno + 1, source_lines)
        if line - 1 > _node.fromlineno + 1:
            yield (None, slice(None, None), LineType.CONTEXT, '"""MORE CODE OMITTED"""')
    yield from render_context(line - 1, line + 1, source_lines)

    # calculate indentation for the insertion point
    body = source_lines[end - 1]
    indentation = len(body) - len(body.lstrip())
    insertion_text = body[:indentation] + '"""MISSING RETURN STATEMENT"""'

    # insert the message
    yield (
        None,
        slice(indentation, None),
        LineType.ERROR,
        insertion_text,
    )

    yield from render_context(end + 1, end + 2, source_lines)
```

After completing the rendering function, we need to add an entry in `CUSTOM_MESSAGES` dictionary to tell the module on which message it should use the rendering function.

```python
CUSTOM_MESSAGES = {
    "missing-module-docstring": render_missing_docstring,
    "missing-class-docstring": render_missing_docstring,
    "missing-function-docstring": render_missing_docstring,
    "trailing-newlines": render_trailing_newlines,
    "trailing-whitespace": render_trailing_whitespace,
    "missing-return-type": render_missing_return_type,
    "too-many-arguments": render_too_many_arguments,
    "missing-space-in-doctest": render_missing_space_in_doctest,
    "pep8-errors": render_pep8_errors,
    "missing-return-statements": render_missing_return_statements,
}
```

### Writing Tests for the Custom Checker

Now that we completed the code for the checker, we also need to create a few unit tests to make sure it's working as intended. One problem with writing unit tests is that we need to construct the control flow graph for the test case before running the checker. Following the same approach as existing tests on CFG-based checkers, we can implement the tests as follows:

```python
   
def test_missing_return_in_branch(self):
        src = """
        def missing_return_in_branch() -> int:
            a = 1
            if a > 3:
                print("no return")
            else:
                return a
        """

        mod = astroid.parse(src)
        mod.accept(CFGVisitor())
        func_node = next(mod.nodes_of_class(nodes.FunctionDef))

        with self.assertAddsMessages(
            pylint.testutils.MessageTest(
                msg_id="missing-return-statements",
                node=func_node,
            ),
            ignore_position=True,
        ):
            self.checker.visit_functiondef(func_node)
```

1. `astroid.parse()` method constructs the AST from the source code
    
2. The construction of control flow graph uses **visitor pattern**, which is why generating the control flow graph from the AST is implemented as `mod.accept(CFGVisitor())`
    
3. `nodes_of_class()` returns a Generator that returns every node with given type. We can use `next()` to get the element from Generator. Alternatively, to unpack multiple nodes, we can use pattern matching as the example below
    
    ```python
        
    def test_function_with_nested_functions(self):
            src = """
            def outer_function():
                def inner_function() -> int:
                    print("inner function")
                print("no return")
            """
    
            mod = astroid.parse(src)
            mod.accept(CFGVisitor())
            outer_func_node, inner_func_node = mod.nodes_of_class(nodes.FunctionDef)
    
            with self.assertAddsMessages(
                pylint.testutils.MessageTest(
                    msg_id="missing-return-statements",
                    node=inner_func_node,
                ),
                ignore_position=True,
            ):
                self.checker.visit_functiondef(outer_func_node)
                self.checker.visit_functiondef(inner_func_node)
    ```
    
    I won't cover every unit test in detail. To sum up, we need to write both positive and negative cases on different types of control flow graph structures, including sequential structure, if statements with and without elif and else branches, while loops, for loops, inner functions, try-except statements.