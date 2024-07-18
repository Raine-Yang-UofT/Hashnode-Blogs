---
title: "PyTA Project: Converting String Expressions to Z3 Constraints"
datePublished: Thu Jul 18 2024 18:09:08 GMT+0000 (Coordinated Universal Time)
cuid: clyrl58gg000109jxc43h4r8r
slug: pyta-project-converting-string-expressions-to-z3-constraints
tags: python, string, z3, abstract-syntax-tree

---

This article is a continuation of [the previous task](https://raineyang.hashnode.dev/pyta-project-converting-function-preconditions-to-z3-constraints) , which implements the parsing of container types (`list/set/tuple`) and operators to Z3 constraints. In today's task, we will implement the parsing for string variables and corresponding operators, including equality check, `in/not in` operators, indexing, and slicing.

### String Operations in Z3

In Z3, a string is represented as type `SeqRef` . String literals can be implicitly convered to string type in Z3 and passed into Z3 string methods.The string methods in Z3 we will use are as follows:

* `String(name)`: create a string variable with given `name`
    
* `Contains(string, substring)`: whether `substring` is a part of `string`
    
* `SubString(string, lower, upper)`: returns a substring of `string` from `lower` (inclusive) to `upper` (exclusive) index
    
* `Concat(*string)` : connects the given string arguments to a single string
    
* `Length(string)` : returns the length of a string
    

Here are a few example uses:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721316124096/be94fffe-aaa8-45d3-9621-e78f7161b645.png align="center")

### Parsing `in/not in` operations

Firstly, the parsing of a string variable to a Z3 String is quite straightforward, which we can implement in the same way as parsing other data types in `apply_name` method

```python
    def apply_name(self, name: str) -> z3.ExprRef:
        """
        Set up the appropriate variable representation in Z3 based on name and type.
        If an error is encountered or a case is unconsidered, return None.
        """
        typ = self.types[name]
        type_to_z3 = {
            "int": z3.Int,
            "float": z3.Real,
            "bool": z3.Bool,
            "str": z3.String,
        }
        if typ in type_to_z3:
            x = type_to_z3[typ](name)
        else:
            raise Z3ParseException(f"Unhandled type {typ}.")

        return x
```

To support `in/not in` for strings, we want to first do a little bit of refactoring. In the previous article we have implemented parsing `in/not in` operators on container types. We want to combine that with string parsing in one function so that we are handling `in/not in` binary operators in a single method.

In parsing `in/not in` for container type, we converts the expression as a list of equality checks on whether the variable is equal to (or not equal to) any of the values in the container. However, for `in/not in` on strings we need to use an alternative method, as in this case, we are not checking whether the string variable matches a single character, but any possible substrings. In addition, for expressions like `x in y` where `x` and `y` are both string arguments, we cannot know the composition of the strings beforehand. Thus, we can directly apply the `Contains` method for the parsing:

```python
    def apply_in_op(
        self, left: z3.ExprRef, right: Union[z3.ExprRef, List[z3.ExprRef], str], negate=False
    ) -> z3.ExprRef:
        """
        Apply `in` or `not in` operator on a list or string and return the
        resulting z3 expression. Raise Z3ParseException if the operands
        do not support `in` operator
        """
        if isinstance(right, list):  # container type (list/set/tuple)
            return (
                z3.And(*[left != element for element in right])
                if negate
                else z3.Or(*[left == element for element in right])
            )
        elif isinstance(right, (str, z3.SeqRef)):  # string literal or variable
            return z3.Not(z3.Contains(right, left)) if negate else z3.Contains(right, left)
        else:
            op = "not in" if negate else "in"
            raise Z3ParseException(
                f"Unhandled binary operation {op} with operator types {left} and {right}."
            )
```

### Parsing String Indexing and Slicing

The parsing of string indexing and slicing is an especially tricky case, as Python supports multiple types of indexing and slicing arguments which Z3 does not fully support. Cases need to be considered are as follows:

* positive indexing: `x[1]`
    
* negative indexing: `x[-1]` , which indexes from right to left
    
* positive slicing with fixed lower and upper bound: `x[1:4]`
    
* negative slicing with fixed lower and upper bound: `x[-4:-1]`
    
* slicing with indeterminant upper bound: `x[1:]` , where the upper bound is the end of string
    
* slicing with step length: `x[1:4:2]`
    
* slicing with indeterminant upper bound and step length: `x[1::2]`
    

In the rest of the article, we will go through the implement that accounts for all the cases above (except the last one). Currently, we do not has an easy algorithm to represent the last case in Z3 expression and have to skip this case.

Both indexing and slicing operations are represented as a `Subscript` node in Astroid AST. The `slice` attribute of `Subscript` indicates the specific operation. If it is an indexing with positive index, the `slice` would be a `Const` node whose `value` is the index. A negative indexing is represented as a combination of `UnaryOp` whose `op` should be `-` and a `Const` node. A slicing operation is represented as a `Slice` node, which has attributes `lower` ,`upper`, and `step` . The values of `lower` ,`upper`, and `step` are stored as Astroid nodes in the same way as indexing. If an argumented is neglected, the corresponding entry would be `None`. The parser does not automatically fill in default values.

To simplify our implementation, we can first create a helper function that converts an Astroid node representing either positive or negative numbers to a number literal:

```python
    def _parse_number_literal(self, node) -> Optional:
        """
        If the subtree from `node` represent a number literal, return the value
        Otherwise, return None
        """
        # positive number
        if isinstance(node, nodes.Const) and isinstance(node.value, (int, float)):
            return node.value
        # negative number
        elif (
            isinstance(node, nodes.UnaryOp)
            and node.op == "-"
            and isinstance(node.operand, nodes.Const)
            and isinstance(node.operand.value, (int, float))
        ):
            return -node.operand.value
        else:
            return None
```

For indexing operation, although according to its documentation `z3.SubString` does not seem to support negative indexes, during my testing it seems that this method does parse negative indexes without errors. Thus, we can directly translate an indexing operation to `z3.SubString`

```python
# handle indexing
index = self._parse_number_literal(slice)
if isinstance(index, int):
    return z3.SubString(value, index, index)
```

For slicing operation, we first need to set empty arguments as their default values. Empty `lower`, `upper` , and `step` should be set to 0, `Length(x)` , 1, respectively.

```python
# handle slicing
elif isinstance(slice, nodes.Slice):
    lower = 0 if slice.lower is None else self._parse_number_literal(slice.lower)
    upper = (
        z3.Length(value)
        if slice.upper is None
        else self._parse_number_literal(slice.upper)
    )
    step = 1 if slice.step is None else self._parse_number_literal(slice.step)
```

A slicing operation with step 1 can be directly translated to a `SubString` . However, `SubString` does not support the "step length" argument. We can loop through the index range with given step length and get each partition with `SubString` , and then connect them to a single expression with `Concat`. However, this approach would not work if the upper bound is indeterminant (set to the length of string), since we cannot use a `z3.Length` expression as loop range. This problem remains unsolved and is reported as an issue in PythonTA ([https://github.com/pyta-uoft/pyta/issues/1065](https://github.com/pyta-uoft/pyta/issues/1065))

```python
if (
    isinstance(lower, int)
    and isinstance(upper, (int, z3.ArithRef))
    and isinstance(step, int)
):

    if step == 1:
        return z3.SubString(value, lower, upper)
    else:
        # unhandled case: the upper bound is indeterminant
        if upper == z3.Length(value):
            raise Z3ParseException(
                "Unable to convert a slicing operation with a non-unit step length and an indeterminant upper bound"
            )

            return z3.Concat(
                *(z3.SubString(value, i, i) for i in range(lower, upper, step))
            )
        else:
            raise Z3ParseException(f"Invalid slice {slice}")
```

The complete code is shown below (including raising errors for invalid values):

```python
    def _parse_number_literal(self, node) -> Optional:
        """
        If the subtree from `node` represent a number literal, return the value
        Otherwise, return None
        """
        # positive number
        if isinstance(node, nodes.Const) and isinstance(node.value, (int, float)):
            return node.value
        # negative number
        elif (
            isinstance(node, nodes.UnaryOp)
            and node.op == "-"
            and isinstance(node.operand, nodes.Const)
            and isinstance(node.operand.value, (int, float))
        ):
            return -node.operand.value
        else:
            return None

    def parse_subscript_op(self, node: nodes.Subscript) -> z3.ExprRef:
        """
        Convert an astroid Subscript node to z3 expression.
        This method only supports string values and integer literal (both positive and negative) indexes
        """
        value = self.reduce(node.value)
        if isinstance(value, z3.SeqRef):
            slice = node.slice

            # handle indexing
            index = self._parse_number_literal(slice)
            if isinstance(index, int):
                return z3.SubString(value, index, index)

            # handle slicing
            elif isinstance(slice, nodes.Slice):
                lower = 0 if slice.lower is None else self._parse_number_literal(slice.lower)
                upper = (
                    z3.Length(value)
                    if slice.upper is None
                    else self._parse_number_literal(slice.upper)
                )
                step = 1 if slice.step is None else self._parse_number_literal(slice.step)

                if (
                    isinstance(lower, int)
                    and isinstance(upper, (int, z3.ArithRef))
                    and isinstance(step, int)
                ):

                    if step == 1:
                        return z3.SubString(value, lower, upper)
                    else:
                        # unhandled case: the upper bound is indeterminant
                        if upper == z3.Length(value):
                            raise Z3ParseException(
                                "Unable to convert a slicing operation with a non-unit step length and an indeterminant upper bound"
                            )

                        return z3.Concat(
                            *(z3.SubString(value, i, i) for i in range(lower, upper, step))
                        )
                else:
                    raise Z3ParseException(f"Invalid slice {slice}")
            else:
                raise Z3ParseException(f"Invalid index {slice}")

        else:
            raise Z3ParseException(f"Unhandled subscript operand type {value}")
```