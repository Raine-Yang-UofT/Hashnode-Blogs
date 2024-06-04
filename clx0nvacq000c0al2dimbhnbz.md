---
title: "Pyta Project: Editing pre-commit hook configuration"
datePublished: Tue Jun 04 2024 17:15:54 GMT+0000 (Coordinated Universal Time)
cuid: clx0nvacq000c0al2dimbhnbz
slug: pyta-project-editing-pre-commit-hook-configuration
tags: python, testing, yaml, regular-expressions, pre-commit

---

Pre-commit hooks are checkers that automatically fix code style issues before commiting the code. Instead of reporting the style errors to users (like PythonTA), they normally do not display the specific style rules being used and just directly refactor the code without confirmation from users. However, in PythonTA project, the unit tests are codes that are intended to exhibit certain style errors, and should not be "fixed" by pre-commit hooks. Thus, I need to modify the pre-commit configuration to exclude the test files from pre-commit checking.

### The pre-commit configuration

`.pre-commit-config.yaml` is the configuration file for pre-commit hooks. It specifies the hooks being used and their versions, as well as files to be included and excluded from the checking. A full list of configurations can be found in the documentation ([https://pre-commit.com/#plugins](https://pre-commit.com/#plugins))

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-yaml
      - id: end-of-file-fixer
      - id: trailing-whitespace
  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks:
      - id: isort
  - repo: https://github.com/psf/black-pre-commit-mirror
    rev: 24.3.0
    hooks:
      - id: black
        args: [--safe, --quiet]
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v4.0.0-alpha.8
    hooks:
      - id: prettier
exclude: examples

ci:
  autoupdate_schedule: quarterly
```

In the exisitng configuration, `examples` , the directory that stores the test cases, has been added to `exclude` . However, in my latest modification, files under the directory `tests/fixtures/sample_dir` also need to be ignored by pre-commit hooks. Thus, I need to add this path to `exclude` as well.

However, when I try to assign a list to `exclude` , both `examples` and `sample_dir` are then being not excluded from the checks. Refering to the documentation ([https://pre-commit.com/#plugins](https://pre-commit.com/#plugins)), it turns out `exclude` can only accept a single string, and I can use regular expression to match multiple directories.

### Regular Expression

Regular expression is a sequence of characters that form a search pattern. It contains these metacharacters used to match a specific pattern:

* `[]` : match a set of characters, for example \[a-m\] matches any character from "a" to "m"
    
* `\` : escape character, or matching for specific sequences. Commonly used are \\d (match a digit), \\w (match a letter, digit, or \_), \\s (match for white space)
    
* `.` : match any character
    
* `^` : match the beginning of string
    
* `$` : match the end of string
    
* `*` : match zero or more occurances
    
* `+` : match one or more occurances
    
* `?` : match zero or one occurances
    
* `{}` : indicate the specific number of occurances
    
* `|` : or
    
* `()` : grouping an expression
    

In the pre-commit documentation, there is an example that shows how to match multiple files with regular expression:

```yaml
# ...
    -   id: my-hook
        exclude: |
            (?x)^(
                path/to/file1.py|
                path/to/file2.py|
                path/to/file3.py
            )$
```

The beginning pipe in `exclude: |` indicates a multi-line string. `(?x)` is a verbose flag that allows inline comments in the regular expression. `^` and `$` before and after the brackets matches beginning and end of the string. Finally, inside the brackets different file names are separated with or operator `|` . In summary, this pattern matches files whose full paths that matches one of the paths listed.

In our case, we need to make a slight modification on the regular expression, since we are not matching for specific files but all files under given directories. Thus, the `$` in the end need to be omited so that we are not just matching the specific directory but sub-directories and files within.

```yaml
exclude: |
  (?x)^(
      examples|
      tests/fixtures/sample_dir
  )
```