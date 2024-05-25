---
title: "PyTA Project: Refactoring test_check_on_dir() in test_check.py"
datePublished: Sat May 25 2024 20:09:28 GMT+0000 (Coordinated Universal Time)
cuid: clwmjnzti000b0ak3biurc0pu
slug: pyta-project-refactoring-testcheckondir-in-testcheckpy
tags: unit-testing, python, json, testing, pylint

---

Today's task is to refactor `test_check_on_dir()` function in `test_check.py` module for PythonTA ([https://github.com/pyta-uoft/pyta](https://github.com/pyta-uoft/pyta)).

According to the instruction, the current test\_check\_on\_dir() does two things simultaneously: testing whether `python_ta.check_all()` runs successfully with a directory name as parameter and testing checking all files in `examples` directory with PythonTA. The task is to change `test_check_on_dir()` to only do the former task by running it at a much smaller directory than `examples`, and create a separate unit test in `test_examples.py` that runs all files in `examples/pylint` and `examples/custom_checkers` with PythonTA.

### The `test_check_on_dir()` function

`test_check_on_dir()` tests `python_ta.check_all()` method with argument `module_name` be a directory. In the test, we can see that `examples` is used as the directory to test on.

```python
def test_check_on_dir():
    """The function, check_all() should handle a top-level dir as input."""
    reporter = python_ta.check_all(
        "examples",
        config={
            "output-format": "python_ta.reporters.JSONReporter",
            "pyta-error-permission": "no",
            "pyta-file-permission": "no",
        },
    )
    for filename, messages in reporter.messages.items():
        print(f"Checking file: {filename}")
        assert "astroid-error" not in {
            msg.message.symbol for msg in messages
        }, f"astroid-error encountered for {filename}"
```

`python_ta.check_all()` takes a parameter `module_name: str/list[str]` as the files to check with PythonTA. If the parameter is empty, the module in which `check_all()` is invoked is being checked.

`check_all()` invokes the `_check()` method, which does the actual checking. In `_check()` , the file paths specified by `module_name` are first examined by `_get_valid_files_to_check()` , which makes sure the paths exist, and then for every directory path, the files inside the directory is retrieved by `get_file_path()` . Note that the use of `os.walk()` ensures recursive searching in sub-directories in the directory. After these two steps, we can get the path of every file needs to be checked.

```python
# in python_ta.__init__
def _get_valid_files_to_check(module_name: Union[List[str], str]) -> Generator[AnyStr, None, None]:
    """A generator for all valid files to check."""
    # Allow call to check with empty args
    if module_name == "":
        m = sys.modules["__main__"]
        spec = importlib.util.spec_from_file_location(m.__name__, m.__file__)
        module_name = [spec.origin]
    # Enforce API to expect 1 file or directory if type is list
    elif isinstance(module_name, str):
        module_name = [module_name]
    # Otherwise, enforce API to expect `module_name` type as list
    elif not isinstance(module_name, list):
        logging.error(
            "No checks run. Input to check, `{}`, has invalid type, must be a list of strings.".format(
                module_name
            )
        )
        return

    # Filter valid files to check
    for item in module_name:
        if not isinstance(item, str):  # Issue errors for invalid types
            logging.error(
                "No check run on file `{}`, with invalid type. Must be type: str.\n".format(item)
            )
        elif os.path.isdir(item):
            yield item
        elif not os.path.exists(os.path.expanduser(item)):
            try:
                # For files with dot notation, e.g., `examples.<filename>`
                filepath = modutils.file_from_modpath(item.split("."))
                if os.path.exists(filepath):
                    yield filepath
                else:
                    logging.error("Could not find the file called, `{}`\n".format(item))
            except ImportError:
                logging.error("Could not find the file called, `{}`\n".format(item))
        else:
            yield item  # Check other valid files.


def get_file_paths(rel_path: AnyStr) -> Generator[AnyStr, None, None]:
    """A generator for iterating python files within a directory.
    `rel_path` is a relative path to a file or directory.
    Returns paths to all files in a directory.
    """
    if not os.path.isdir(rel_path):
        yield rel_path  # Don't do anything; return the file name.
    else:
        for root, _, files in os.walk(rel_path):
            for filename in (f for f in files if f.endswith(".py")):
                yield os.path.join(root, filename)  # Format path, from root.
```

Based on how `check_all()` searches for the files, we may expect that in `test_check_on_dir()`, by passing `examples` as the parameter, every sub-directories and files within it will be checked by PythonTA. However, when I run the test, the actual result is not as expected. Firstly, the test finishes almost immediately after it runs, which seems suspecious since there are a lot of files in `examples`. Secondly, when I add a print statement in the test function and run it again, it turns out only the `__init__.py` in `examples` is being tested.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716258852991/2172483e-6bb0-4e8a-afe5-88cbeed4d407.png align="center")

The reason of the issue, it turns out, is that I didn't run the tests at the correct directory. The file paths specified in `test_check` are relative paths starting from `pyta` directory. we have to first go to `pyta` directory and run the tests:

```console
python -m pytest tests
```

Refactoring the relative paths used by the unit tests to allow pytest to run on from any directory could be an improvement for future. However, In the rest of the article, I will first refactor `test_check_on_dir()` to run on a smaller directory, and then write a test that runs PythonTA on `examples`.

### Refactoring `test_check_on_dir()` function

As specified by the instruction, we first create a directory called `sample_dir` in `examples`, and moved a subset of `examples` into the directory as the test cases. The subset contains test cases for pylint and custom checkers, organized in the hierarchy similar to `examples` for testing searching through nested directories. A few test cases from `syntax_errors` are also included to make sure PythonTA runs correctly even on codes that contain syntex errors.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716313712527/cda4f191-e43c-436b-bc01-9c39f0fc5067.png align="center")

Accordingly, we changed the `module_path` parameter for `python_ta.check_all()` :

```python
def test_check_on_dir():
    """The function, check_all() should handle a top-level dir as input."""
    reporter = python_ta.check_all(
        "examples/sample_dir",
        config={
            "output-format": "python_ta.reporters.JSONReporter",
            "pyta-error-permission": "no",
            "pyta-file-permission": "no",
        },
    )

    for filename, messages in reporter.messages.items():
        assert "astroid-error" not in {
            msg.message.symbol for msg in messages
        }, f"astroid-error encountered for {filename}"
```

### How Unit Test in `test_examples.py` Works

Now we need to create a unit test in `test_examples.py` to run PythonTA on all files in `examples/pylint` and `examples/custom_checkers` . There is already an existng test that runs `pylint` on `examples/pylint` . What we need to do is to first understand how the test works, and then imitate the existing unit test to implement our own, except running `python_ta` instead of `pylint` .

The unit test contains three steps:

1 `get_file_paths()` retrieves the paths of all test files in the given directory:

```python
_EXAMPLES_PATH = "examples/pylint/"

# The following tests appear to always fail (further investigation needed).
IGNORED_TESTS = [
    "e1131_unsupported_binary_operation.py",
    "e0118_used_prior_global_declaration.py",
    "w0125_using_constant_test.py",
    "w0631_undefined_loop_variable.py",
    "w1503_redundant_unittest_assert.py",
    "e1140_unhashable_dict_key.py",
    "r0401_cyclic_import.py",  # R0401 required an additional unit test but should be kept here.
]

def get_file_paths():
    """Gets all the files from the examples folder for testing. This will
    return all the full file paths to the file, meaning they will have the
    _EXAMPLES_PATH prefix followed by the file name for each element.
    A list of all the file paths will be returned."""
    test_files = []
    for _, _, files in os.walk(_EXAMPLES_PATH, topdown=True):
        for filename in files:
            if filename not in IGNORED_TESTS and filename.endswith(".py"):
                test_files.append(_EXAMPLES_PATH + filename)
    return test_files
```

From the code, we can see that this function searches for every python file in `examples/pylint` and returns a list of their relative paths starting from pyta.

2 `symbols_by_file()` runs pylint on every test file, producing a dictionary of file `name : list of pylint errors`

```python
@pytest.fixture(scope="session", autouse=True)
def symbols_by_file() -> Dict[str, Set[str]]:
    """Run pylint on all the example files and return the map of file name to the
    set of Pylint messages it raises."""

    sys.stdout = StringIO()
    lint.Run(
        [
            "--reports=n",
            "--rcfile=python_ta/config/.pylintrc",
            "--output-format=json",
            *get_file_paths()
        ], exit=False
    )
    jsons_output = sys.stdout.getvalue()
    sys.stdout = sys.__stdout__

    pylint_list_output = json.loads(jsons_output)

    file_to_symbol = {}
    for path, group in itertools.groupby(pylint_list_output, key=lambda d: d["path"]):
        symbols = {message["symbol"] for message in group}
        file = os.path.basename(path)

        file_to_symbol[file] = symbols

    return file_to_symbol
```

The decorator `@pytest.fixture(scope="session", autouse=True)` indicates that this function is invoked during the initialization of unit tests. `scope="session"` means that the function is called once before a test runs, and `autouse=True` means the function is being automatically invoked.

In the function body, since the default output of pylint is `sys.__stdout__` , we temporarily changes the standard output to a `StringIO`, an object that provides interface similar to a file and can store pylint's output. We also sets the pylint output to json format. The json file for a single pylint message looks as follows:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716572929913/06b6597e-cdee-4b67-92a2-9046818f3639.png align="center")

Finally, we can create the dictionary with `itertools.groupby()` method, which returns key and iterable of grouped items. For example, calling `itertools.groupby()` on **\[('a', 1), ('a', 2), ('b', 3), ('b', 4), ('a', 5)\]** would return **{'a': \[('a', 1), ('a', 2)\], 'b': \[('b', 3), ('b', 4)\], 'a': \[('a', 5)\]}**

3 `test_examples_files()` deduces the supposed error type by file name, and checks whether the error exists in the list of pylint errors corresponding to the file:

```python
_EXAMPLE_PREFIX_REGEX = r"[CEFRW]\d{4}"

@pytest.mark.parametrize("test_file", get_file_paths())
def test_examples_files(test_file: str, symbols_by_file: Dict[str, Set[str]]) -> None:
    """Creates all the new unit tests dynamically from the testing directory."""
    base_name = os.path.basename(test_file)
    if not re.match(_EXAMPLE_PREFIX_REGEX, base_name[:5]):
        return
    if not base_name.lower().endswith(".py"):
        assert False
    checker_name = base_name[6:-3].replace("_", "-")  # Take off prefix and file extension.

    test_file_name = os.path.basename(test_file)
    file_symbols = symbols_by_file[test_file_name]

    found_pylint_message = checker_name in file_symbols
    assert found_pylint_message, f"Failed {test_file}. File does not add expected message."
```

The decorator `@pytest.mark.parametrize("test_file", get_file_paths())` provides a convenient shortcut to invoke `get_file_paths()`. It means that when using parameter `test_file`, `get_file_paths()` will be invoked to provide the file, allowing us to use a function as a variable and dynamically get the function return values.

The function deduces the error message present in the files by interpreting the file names. In `examples/pylint` , every file follows the name convention &lt;error message id&gt;\_&lt;error name&gt;. The regular expression `r"[CEFRW]\d{4}"` matches files that start with one of C,E,F,R,W, following 4 digits. The error message name is rest of the file name with "\_" replaced as "-". Finally, we just need to check that whether the error message in the file name presents in pylint error messages for the file.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716574044691/6714cf36-5c92-4e26-8713-b95baa67f7ad.png align="center")

### Creating Unit Test with PythonTA

Now what we need to do is to make some modifications on the three steps above to create a unit test that runs example files with PythonTA.

1. Pylint only runs test cases in `examples/pylint` . However, PythonTA needs to run both `examples/pylint` and `examples/custom_checkers` , which tests the custom checkers PythonTA expanded on Pylint. For expandability concerns, instead of creating a separate function for this test, we modify `get_file_paths()` to take a string or list of strings as the paths to gather the test files:
    

```python
def get_file_paths(paths: Union[str, List[str]]):
    """
    Gets all the files from the examples folder for testing. This will
    return all the full file paths to the file, meaning they will have the
    path prefix followed by the file name for each element.
    A list of all the file paths will be returned.

    :param paths: The paths to retrieve the files from.
    """
    test_files = []

    if isinstance(paths, str):
        paths = [paths]

    for path in paths:
        for _, _, files in os.walk(path, topdown=True):
            for filename in files:
                if filename not in IGNORED_TESTS and filename.endswith(".py"):
                    test_files.append(path + filename)

    return test_files
```

2. We create a function similar to `symbols_by_file()` that runs PythonTA to get the corresponding error messages for every file. In addition to calling PythonTA instead of Pylint, we also need to make minor modifications on the handling of json message since PythonTA's json output has a different format (see the screenshot below).
    

```python
@pytest.fixture(scope="session")
def symbols_by_file_pyta() -> Dict[str, Set[str]]:
    """Run python_ta on all the example files and return the map of file name to the
    set of PythonTA messages it raises."""
    sys.stdout = StringIO()
    python_ta.check_all(
        module_name=get_file_paths([_EXAMPLES_PATH, _CUSTOM_CHECKER_PATH]),
        config={
            "output-format": "python_ta.reporters.JSONReporter",
        }
    )

    jsons_output = sys.stdout.getvalue()
    sys.stdout = sys.__stdout__
    pyta_list_output = json.loads(jsons_output)

    file_to_symbol = {}

    def get_msg_path(d):
        assert d["msgs"], "The 'msgs' list should not be empty"
        return d["msgs"][0]["path"]

    for path, group in itertools.groupby(pyta_list_output, key=get_msg_path):
        symbols = {message["msgs"][0]["symbol"] for message in group}
        file = os.path.basename(path)

        file_to_symbol[file] = symbols

    return file_to_symbol
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716575393124/a591ce18-13fb-44e4-b894-83818a007740.png align="center")

The updated `test_examples_files_pyta()` method is mostly the same as `test_examples_files()` except calling `symbols_by_file_pyta()`.

```python
@pytest.mark.parametrize("test_file", get_file_paths([_EXAMPLES_PATH, _CUSTOM_CHECKER_PATH]))
def test_examples_files_pyta(test_file: str, symbols_by_file_pyta: Dict[str, Set[str]]) -> None:
    """Creates all the new unit tests dynamically from the testing directory."""
    base_name = os.path.basename(test_file)
    if not re.match(_EXAMPLE_PREFIX_REGEX, base_name[:5]):
        return
    if not base_name.lower().endswith(".py"):
        assert False
    checker_name = base_name[6:-3].replace("_", "-")  # Take off prefix and file extension.

    test_file_name = os.path.basename(test_file)
    file_symbols = symbols_by_file_pyta[test_file_name]

    found_pylint_message = checker_name in file_symbols
    assert found_pylint_message, f"Failed {test_file}. File does not add expected message."
```

Here's the complete code of `test_examples` module with updated version of both unit tests:

```python
from typing import Dict, Set, Union, List

import os
import subprocess
import re
import pytest
import json
import itertools
from pylint import lint
from io import StringIO
import sys
import python_ta


_EXAMPLES_PATH = "examples/pylint/"
_CUSTOM_CHECKER_PATH = "examples/custom_checkers/"
_EXAMPLE_PREFIX_REGEX = r"[CEFRW]\d{4}"


# The following tests appear to always fail (further investigation needed).
IGNORED_TESTS = [
    "e1131_unsupported_binary_operation.py",
    "e0118_used_prior_global_declaration.py",
    "w0125_using_constant_test.py",
    "w0631_undefined_loop_variable.py",
    "w1503_redundant_unittest_assert.py",
    "e1140_unhashable_dict_key.py",
    "r0401_cyclic_import.py",  # R0401 required an additional unit test but should be kept here.
    "e9999_forbidden_import_local.py"  # This file itself (as an empty file) should not be checked
]


def get_file_paths(paths: Union[str, List[str]]):
    """
    Gets all the files from the examples folder for testing. This will
    return all the full file paths to the file, meaning they will have the
    path prefix followed by the file name for each element.
    A list of all the file paths will be returned.

    :param paths: The paths to retrieve the files from.
    """
    test_files = []

    if isinstance(paths, str):
        paths = [paths]

    for path in paths:
        for _, _, files in os.walk(path, topdown=True):
            for filename in files:
                if filename not in IGNORED_TESTS and filename.endswith(".py"):
                    test_files.append(path + filename)

    return test_files


@pytest.fixture(scope="session")
def symbols_by_file() -> Dict[str, Set[str]]:
    """Run pylint on all the example files and return the map of file name to the
    set of Pylint messages it raises."""

    sys.stdout = StringIO()
    lint.Run(
        [
            "--reports=n",
            "--rcfile=python_ta/config/.pylintrc",
            "--output-format=json",
            *get_file_paths(_EXAMPLES_PATH)
        ], exit=False
    )
    jsons_output = sys.stdout.getvalue()
    sys.stdout = sys.__stdout__

    pylint_list_output = json.loads(jsons_output)

    file_to_symbol = {}
    for path, group in itertools.groupby(pylint_list_output, key=lambda d: d["path"]):
        symbols = {message["symbol"] for message in group}
        file = os.path.basename(path)

        file_to_symbol[file] = symbols

    return file_to_symbol


@pytest.mark.parametrize("test_file", get_file_paths(_EXAMPLES_PATH))
def test_examples_files(test_file: str, symbols_by_file: Dict[str, Set[str]]) -> None:
    """Creates all the new unit tests dynamically from the testing directory."""
    base_name = os.path.basename(test_file)
    if not re.match(_EXAMPLE_PREFIX_REGEX, base_name[:5]):
        return
    if not base_name.lower().endswith(".py"):
        assert False
    checker_name = base_name[6:-3].replace("_", "-")  # Take off prefix and file extension.

    test_file_name = os.path.basename(test_file)
    file_symbols = symbols_by_file[test_file_name]

    found_pylint_message = checker_name in file_symbols
    assert found_pylint_message, f"Failed {test_file}. File does not add expected message."


@pytest.fixture(scope="session")
def symbols_by_file_pyta() -> Dict[str, Set[str]]:
    """Run python_ta on all the example files and return the map of file name to the
    set of PythonTA messages it raises."""
    sys.stdout = StringIO()
    python_ta.check_all(
        module_name=get_file_paths([_EXAMPLES_PATH, _CUSTOM_CHECKER_PATH]),
        config={
            "output-format": "python_ta.reporters.JSONReporter",
        }
    )

    jsons_output = sys.stdout.getvalue()
    sys.stdout = sys.__stdout__
    pyta_list_output = json.loads(jsons_output)

    file_to_symbol = {}

    def get_msg_path(d):
        assert d["msgs"], "The 'msgs' list should not be empty"
        return d["msgs"][0]["path"]

    for path, group in itertools.groupby(pyta_list_output, key=get_msg_path):
        symbols = {message["msgs"][0]["symbol"] for message in group}
        file = os.path.basename(path)

        file_to_symbol[file] = symbols

    return file_to_symbol


@pytest.mark.parametrize("test_file", get_file_paths([_EXAMPLES_PATH, _CUSTOM_CHECKER_PATH]))
def test_examples_files_pyta(test_file: str, symbols_by_file_pyta: Dict[str, Set[str]]) -> None:
    """Creates all the new unit tests dynamically from the testing directory."""
    base_name = os.path.basename(test_file)
    if not re.match(_EXAMPLE_PREFIX_REGEX, base_name[:5]):
        return
    if not base_name.lower().endswith(".py"):
        assert False
    checker_name = base_name[6:-3].replace("_", "-")  # Take off prefix and file extension.

    test_file_name = os.path.basename(test_file)
    file_symbols = symbols_by_file_pyta[test_file_name]

    found_pylint_message = checker_name in file_symbols
    assert found_pylint_message, f"Failed {test_file}. File does not add expected message."


def test_cyclic_import() -> None:
    """Test that examples/pylint/R0401_cyclic_import adds R0401 cyclic-import.

    Reason for creating a separate test:
    This test is separate as pylint adds the R0401 message to the final module within
    the batch of given modules, not to the R0401_cyclic_import or cyclic_import_helper.
    It is unintuitive to force R0401_cyclic_import to be the final batched module so that
    the parametrized test suite passes, so cyclic-import is ignored in the paramterized suite
    and this additional test is created on the side.
    """

    cyclic_import_helper = "examples/pylint/cyclic_import_helper.py"
    cyclic_import_file = "examples/pylint/r0401_cyclic_import.py"

    sys.stdout = StringIO()
    lint.Run(
        [
            "--reports=n",
            "--rcfile=python_ta/config/.pylintrc",
            "--output-format=json",
            cyclic_import_helper, cyclic_import_file
        ], exit=False
    )
    jsons_output = sys.stdout.getvalue()
    sys.stdout = sys.__stdout__

    pylint_list_output = json.loads(jsons_output)

    found_cyclic_import = any(
        message["symbol"] == "cyclic-import" for message in pylint_list_output
    )
    assert found_cyclic_import, f"Failed {cyclic_import_file}. File does not add expected message."
```

### Current Issues with the Unit Test

While running the tests with pytest, the first error encountered is when messages are empty for certain files. The problem, it turns out, is caused by the file `e9999_forbidden_import_local.py` . In PythonTA, importing external libraries and local modules are not permitted unless specified. The test file `e9999_forbidden_import_local.py` is an empty file that is being imported elsewhere, triggering the local import error. However, `e9999_forbidden_import_local.py` itself does not raise any PythonTA errors and should not be checked.

To fix this, we can simply add it to the list IGNORED\_TESTS, files that will not be gathered by `get_files_paths()`

```python
# The following tests appear to always fail (further investigation needed).
IGNORED_TESTS = [
    "e1131_unsupported_binary_operation.py",
    "e0118_used_prior_global_declaration.py",
    "w0125_using_constant_test.py",
    "w0631_undefined_loop_variable.py",
    "w1503_redundant_unittest_assert.py",
    "e1140_unhashable_dict_key.py",
    "r0401_cyclic_import.py",  # R0401 required an additional unit test but should be kept here.
    "e9999_forbidden_import_local.py"  # This file itself (as an empty file) should not be checked
]
```

Another problem is that the current implementation of PythonTA test does not check for test cases for PEP8 errors, located at `examples/custom_checkers/e9989_pycodestyle` directory. Recall that the test looks for test files by matching the first five characters in file name that correspond to error id, and determines error type by getting the rest of file name. However, the PEP8 error test cases do not match the naming conventions of other files and are not being tested. Further studies is needed to determine a way to effectively handle these test cases specifically.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716667616435/3b5fb7f1-2070-46fc-9e7d-2d60d48743bb.png align="center")