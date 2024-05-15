---
title: "PyTA Project: Add Test Cases to Pycodestyle Checker"
datePublished: Tue May 14 2024 21:03:01 GMT+0000 (Coordinated Universal Time)
cuid: clw6vqhad000b0ajw8u5oclyb
slug: pyta-project-add-test-cases-to-pycodestyle-checker
tags: unit-testing, python, testing, pylint

---

In this task, we will improve test coverage rate of `node_printer.py` module in PythonTA ([**https://github.com/pyta-uoft/pyta**](https://github.com/pyta-uoft/pyta)) by adding unit tests to uncovered cases in Pycodestyle checker.

### What Does `node_printers.py` Module Do

The module we need to test, `node_printer.py` (located at `/python_ta/reporters/node_printer.py`), is used to specify how the code snippets with errors should be highlighted. In most cases, we can just highlight the entire line that contains the error. However, in some cases, we may want to provide additional comments or highlight only part of the line that contains the error to provide clearer information, for example:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715715992561/0f6e59ce-af83-4c5e-85c2-1076fa1784e1.png align="center")

`node_printer.py` stores a dictionary that contains all the error types that need special highlighting format (note that some of the types also contain sub-types), and corresponding callback functions to handle specific formatting for each error.

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
}
```

The task is to write unit tests for branches of `node_printer.py` that are not covered. According to the Coveralls report, the errors being untested include:

* E115: Expected an indented block (comment)
    
* E122: Indentation contains mixed spaces and tabs
    
* E127: Continuation line over-indented for visual indent
    
* E131: Continuation line unaligned for hanging indent
    
* E125: Continuation line with same indent as next logical line
    
* E129: Visual indent doesn't match indentation of line above
    
* E223: Tab before operator
    
* E224: Tab after operator
    
* E227: Missing whitespace around bitwise or shift operator
    
* E228: Missing whitespace around modulo operator
    
* E265: Block comment should start with '# '
    
* E266: Too many leading '#' for block comment
    
* E275: Missing trailing comma in Python tuple/list
    
* E301: Expected 1 blank line, found 0
    
* E303: Too many blank lines (5)
    
* E304: Blank line found after function decorator
    

### How node\_printers.py Is Being Called

Before we write tests to `node_printers.py`, we need to first figure out how it interacts with the larger system, in other words, who are the callers of its functions that format the highlights. A look into `node_printers.py` reveals that `render_message()` its only "public" function that should be called by external classes. This method returns the appropriate rendering of the message based on the message type.

```python
def render_message(msg, node, source_lines):
    """Render a message based on type."""
    renderer = CUSTOM_MESSAGES.get(msg.symbol, render_generic)
    yield from renderer(msg, node, source_lines)
```

The reason why `yield from` instead of `return` statement is used (based on my best guess, since I didn't look into the specific functions in detail) is probably due to the need to retain certain internal states (such as local variables) of rendering functions among multiple calls. `yield` statement is similar to `return`, except that it does not immediately exit the function after returning the value. Instead, after `yield` statement, the function is suspended and returns the value, but all its states are stored. When the function is called again, it resumes from the point after the previous `yield` call. This mechanism provides a lightweight alternative to storing the internal states in a class.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715717786701/cee75170-fef1-4bb6-b605-16490e843da4.png align="center")

`render_message()` method's only caller is `PythonTAReporter` class, which inherits `BaseReporter` from Pylint. `render_message()` is called by its private method `_build_snippet()` that collects the generator returned by `render_message()` and returns the formatted code snippet for display. `_build_snippet()` is then called by `handle_node()`, which adds node attributes to the message for reporter.

The caller of `handle_node()` is the overriden version of `add_message()`method in Pylinter. `add_message()` is a method in Pylinter that converts the message class emits by checkers (**MessageDefinition**) to that received by reporters (**Message**) PythonTA provides a decorator over **Message** called **NewMessage** that adds additional attributes for PythonTA report display. In the module **messages**, the `add_message()` method in Pylinter is reassigned to a function called `new_add_message()` that converts **MessageDefinition** to **NewMessage** and calls `handle_node()`.

```python
def patch_messages():
    """Patch PyLinter to pass the node to reporter."""
    old_add_message = PyLinter.add_message

    def new_add_message(
        self,
        msg_id,
        line=None,
        node=None,
        args=None,
        confidence=UNDEFINED,
        col_offset=None,
        end_lineno=None,
        end_col_offset=None,
    ):
        old_add_message(
            self, msg_id, line, node, args, confidence, col_offset, end_lineno, end_col_offset
        )
        msg_info = self.msgs_store.get_message_definitions(msg_id)[0]
        # guard to allow for backward compatability with Pylint's reporters
        if hasattr(self.reporter, "handle_node"):
            self.reporter.handle_node(msg_info, node)

    PyLinter.add_message = new_add_message
```

Finally, `patch_message()` is included in `patch_all()` method of `patches` module, which is invoked in the `__init__` method of `python_ta` module. Thus, PythonTA will modifies the `add_message()` method of pylinter during initialization.

```python
# in patches/__init__.py
def patch_all(messages_config: dict):
    """Execute all patches defined in this module."""
    patch_checkers()
    patch_ast_transforms()
    patch_messages()
    patch_error_messages(messages_config)
```

In conclusion, renderer functions in `node_printer.py` will be triggered whenever a corresponding error message is being reported by `PyLinter.add_message()`. Therefore, what we need to do is to write test cases that trigger that pep8 errors.

### Writing Tests for `node_printer.py`

Pylint provides `pylint.testutils.CheckerTestCase` package to implement unit tests for pylint checkers. Documentations and example implementations of Pylint checkers can be found here:([https://pylint.pycqa.org/en/latest/development\_guide/how\_tos/custom\_checkers.html#testing-a-checker](https://pylint.pycqa.org/en/latest/development_guide/how_tos/custom_checkers.html#testing-a-checker))

The errors listed above all belong to pep8 errors, and are checked by PycodestyleChecker. PycodestyleChecker inherits BaseRawFileChecker in Pylint, meaning that it directly analyzes file text without building AST, and invokes pycodestyle to check for errors.

```python
"""Checker for the style of the file"""

from typing import List, Tuple

import pycodestyle
from astroid import nodes
from pylint.checkers import BaseRawFileChecker
from pylint.lint import PyLinter


class PycodestyleChecker(BaseRawFileChecker):
    """A checker class to report PEP8 style errors in the file.

    Use options to specify the list of PEP8 errors to ignore"""

    name = "pep8_errors"
    msgs = {"E9989": ("Found pycodestyle (PEP8) style error %s at %s", "pep8-errors", "")}

    options = (
        (
            "pycodestyle-ignore",
            {
                "default": (),
                "type": "csv",
                "metavar": "<pycodestyle-ignore>",
                "help": "List of Pycodestyle errors to ignore",
            },
        ),
    )

    def process_module(self, node: nodes.NodeNG) -> None:
        style_guide = pycodestyle.StyleGuide(
            paths=[node.stream().name],
            reporter=JSONReport,
            ignore=self.linter.config.pycodestyle_ignore,
        )
        report = style_guide.check_files()

        for line_num, msg, code in report.get_file_results():
            self.add_message("pep8-errors", line=line_num, args=(code, msg))


class JSONReport(pycodestyle.StandardReport):
    def get_file_results(self) -> List[Tuple]:
        self._deferred_print.sort()
        return [
            (line_number, f"line {line_number}, column {offset}: {text}", code)
            for line_number, offset, code, text, _ in self._deferred_print
        ]


def register(linter: PyLinter) -> None:
    """Required method to auto-register this checker to the linter"""
    linter.register_checker(PycodestyleChecker(linter))
```

The unit tests for PycodestyleChecker lies in file `pyta/tests/test_custom_checkers/test_pycodestylecheckers.py`. There already exists a list of unit tests for other errors, and all we need to do is to imitate the format to implement tests for uncovered errors.

```python
import os

import pylint.testutils
from astroid.astroid_manager import MANAGER

from python_ta.checkers.pycodestyle_checker import PycodestyleChecker

DIR_PATH = os.path.join(__file__, "../../../examples/custom_checkers/e9989_pycodestyle/")


class TestPycodestyleChecker(pylint.testutils.CheckerTestCase):
    CHECKER_CLASS = PycodestyleChecker
    CONFIG = {"pycodestyle_ignore": ["E24"]}

    def test_error_e123(self) -> None:
        """Tests that PEP8 error E123 closing bracket does not match indentation of opening bracket's line triggers"""
        mod = MANAGER.ast_from_file(os.path.join(DIR_PATH, "e123_error.py"))
        with self.assertAddsMessages(
            pylint.testutils.MessageTest(
                msg_id="pep8-errors",
                line=3,
                args=(
                    "E123",
                    "line 3, column 4: closing bracket does not match indentation of opening bracket's line",
                ),
            ),
            ignore_position=True,
        ):
            self.checker.process_module(mod)

    def test_no_error_e123(self) -> None:
        """Tests that PEP8 error E123 closing bracket does not match indentation of opening bracket's line
        is NOT triggered"""
        mod = MANAGER.ast_from_file(os.path.join(DIR_PATH, "e123_no_error.py"))
        with self.assertNoMessages():
            self.checker.process_module(mod)

    """Unit tests for other errors"""
```

By analyzing the previously implemented tests, we can observe that for each error there are two tests: the error exists in one test and does not exist in the other. In addition, the source code to check are located in a different directory as specified in DIR\_PATH.

Now we want to implement custom checker for E115: Expected an indented block (comment). This errors occurs when a comment in an indented block does not follow the indentation as the code. It would not lead to an indentation error (since it is a comment rather than source code being wrongly indented), but will reduces readability of the code.

Firstly, in `pyta/examples/custom_checkers/e9989_pycodestyle/` directory, we create two test cases **e115\_error.py** and **e115\_no\_error.py**, in which the comment is not indented in the former and correctly indented in the latter.

In **e115\_error.py:**

```python
if True:
# No indented block follows the colon
    pass
```

In **e115\_no\_error.py:**

```python
if True:
    # Correctly indented block follows the colon
    pass
```

Finally, we can implement our unit tests for the two cases above as follows:

```python
    def test_error_e115(self) -> None:
        """Test that PEP8 error E115 (Expected an indented block) is triggered"""
        mod = MANAGER.ast_from_file(DIR_PATH + "e115_error.py")
        with self.assertAddsMessages(
            pylint.testutils.MessageTest(
                msg_id="pep8-errors",
                line=2,
                args=("E115", "line 2, column 0: expected an indented block (comment)")
            )
        ):
            self.checker.process_module(mod)

    def test_no_error_e115(self) -> None:
        """Test that PEP8 error E115 (Expected an indented block) is NOT triggered"""
        mod = MANAGER.ast_from_file(DIR_PATH + "e115_no_error.py")
        with self.assertNoMessages():
            self.checker.process_module(mod)
```