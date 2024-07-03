---
title: "PyTA Project: Converting Function Preconditions to Z3 Constraints"
datePublished: Wed Jul 03 2024 18:45:39 GMT+0000 (Coordinated Universal Time)
cuid: cly66uezy00000al7aaoz3agj
slug: pyta-project-converting-function-preconditions-to-z3-constraints
tags: python, static-analysis, z3

---

Today's task is to update `ExprWrapper`, a module that converts a python expression to corresponding z3 expression, to support container classes like `list` , `tuple`, and `set`, and `in` operation. In this article, I will first provide a brief overview of z3 library and its use in PythonTA, and then explain my implementation.

### Introduction to Z3 library

Z3 is a theorem prover library that can solve systems of equations. A complete introduction to Z3 can be found at [this documentation](https://microsoft.github.io/z3guide/programming/Z3%20Python%20-%20Readonly/Introduction/). Here I will only cover features relevant to the task, including data types, arithmetic and boolean logic, functions, and solver.

Z3 contains three data types: integer, real number, and boolean value. Unlike float value in python, real numbers in Z3 are precise values, where rational numbers are represented in quotients and irrational numbers are represented as roots of polynomials. For example `Q(1, 3)` defines a real value 1/3, which differs from the float value 1/3:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719964773573/a10bfb60-8ff7-405a-bf19-d19077aa88dd.png align="center")

We can define variables of different data types and use them in equations. The `solve` function is used to find a solution with the given constraints. If the system of equations is unsatisfiable, the function will return "no solution".

```python
x = Int('x')
y = Real('y')
solve(x**2 + y**2 > 3, x**3 + y < 5)  # result: [y = 0, x = -2]
```

Z3 supports boolean operations including `And`, `Or`, `Not`, `Implies`, `If`, and equals (==). Note that `Implies` represents an "if-then" statement, where the second argument is true when the first one is true. If represents am "if-then-else" statement, where the second argument is returned if the condition is true, otherwise the third argument is returned. The code below shows their difference. Note that the return value is `[]` in the second example because the statement is vacuously true when the condition is false.

```python
p = Bool('p')
q = Bool('q')
solve(Implies(True, q))    # [q = True]
solve(Implies(False, q))   # [] 
solve(If(True, p, q))      # [p = True]
solve(If(False, p, q))     # [q = True]

solve(Implies(And(p, q), q))    # [p = True, q = True]
solve(If(Or(p, q), p, q))      # [p = True, q = False]
```

A `Solver` is a general-purpose solver that can store and solve systems of equations. We can add a constraint to the solver with `add` method. `check` returns `sat` if the constraints has a solution, `unsat` if no solution exists, and `unknown` if the solver cannot solve the system. A constraint is called **satisfiable** if there exists a solution, and **valid** if it's true for all values. A constraint F is valid if Not(F) is unsatisfiable. The solution to a system is called a **model**, and is retrieved from `model` method. Note that `model` only returns one solution to the constraints. If we want to get all solutions to the system, we can iteratively call `model` on the Solver and add the solution to the constraints, until all solutions are found and the constraints are unsatisfiable (or, if there are infinitely many solutions, an infinite loop would occur).

```python
s = Solver()
x = Int('x')
s.add(Or(x == 1, x == 2, x == 3))
solutions = []

while s.check() == sat:
    model = s.model()
    x_value = model.eval(x)
    solutions.append(x_value)
    s.add(x != x_value)
```

The Solver maintains a stack of constraints, and we can use `push` method to create a new scope to add additional constraints, and `pop` to descard constraints from the previous `push`. We can consider this as analogous to git branches, where `push` is like creating a new branch from the main branch, and `pop` reverts to the main branch and discards the changes.

```python
x = Int('x')
y = Int('y')

s = Solver()
s.add(x > 10, y == x + 2)
print (s)    # [x > 10, y == x + 2]
print (s.check())    # sat

print ("Create a new scope...")
s.push()    
s.add(y < 11)
print (s)    # [x > 10, y == x + 2, y < 11]
print (s.check())    # unsat

s.pop()
print (s)    # [x > 10, y == x + 2]
print (s.check())    # sat
```

### PythonTA use of Z3

One feature PythonTA is planning to incorporate is to apply Z3 library detect logical errors statically. For example, in the code below, since the function precondition indicates `x > 0` , the `else` branch should not be executed if the function is called correctly. Thus, the code has a logical error.

```python
def f(x: int) -> int:
    """Precondition: x > 0"""
    if x > 0:
        return 1
    else:
        return 0  # This is unreachable based on the precondition
```

The module `z3_visitor.py` is used to convert function preconditions defined in the function docstring to a list of z3 constraints, and store these constraints as an attribute `z3_constraints` in the `FunctionDef` node. It registers the method `set_function_def_z3_constraints` to Astroid as a transform, which will be invoked whenever a `FunctionDef` node is being visited.

```python
class Z3Visitor:
    """
    The class responsible for visiting astroid nodes (currently only FunctionDef nodes),
    parsing preconditions, and converting them to z3 expressions to be appended in the
    z3_constraints attribute of the node.
    """

    def __init__(self):
        """Return a TransformVisitor that sets an environment for every node."""
        visitor = TransformVisitor()
        # Register transforms
        visitor.register_transform(nodes.FunctionDef, self.set_function_def_z3_constraints)
        self.visitor = visitor

    def set_function_def_z3_constraints(self, node: nodes.FunctionDef):
        # Parse types
        types = {}
        annotations = node.args.annotations
        arguments = node.args.args
        for ann, arg in zip(annotations, arguments):
            if ann is None:
                continue
            # TODO: what to do about subscripts ex. Set[int], List[Set[int]], ...
            inferred = ann.inferred()
            if len(inferred) > 0 and inferred[0] is not Uninferable:
                if isinstance(inferred[0], nodes.ClassDef):
                    types[arg.name] = inferred[0].name
        # Parse preconditions
        preconditions = parse_assertions(node, parse_token="Precondition")
        # Get z3 constraints
        z3_constraints = []
        for pre in preconditions:
            pre = astroid.parse(pre).body[0]
            ew = ExprWrapper(pre, types)
            try:
                transformed = ew.reduce()
            except (Z3Exception, Z3ParseException):
                transformed = None
            if transformed is not None:
                z3_constraints.append(transformed)
        # Set z3 constraints
        node.z3_constraints = z3_constraints
        return node
```

`set_function_def_z3_constraints` first parses the function arguments by inferring from the arguments' type annotations. Then, it retrieves the part of function docstring under "Preconsition" as the function preconditions. Finally, it passes the argument types and preconditions to `ExprWrapper` and invokes its `reduce()` method to get the corresponding z3 constraints of the preconditions.

`ExprWrapper` is the class that converts Astroid nodes to Z3 constraints, and is the class we will be working on. Currently, the `reduce` method in `ExprWrapper` parses data types `int`, `float`, and `bool` , operators including boolean operations (`and, or, not`), unary operation (`not`), binary operations (`+, -, , /, **, <=, >=, <, >, ==`), and function names. To handle nested expressions, each parsing method recursively calls `reduce` on the expression components, where a single name or constant value are the base case.

```python
    def reduce(self, node: astroid.NodeNG = None) -> z3.ExprRef:
        """
        Convert astroid node to z3 expression and return it.
        If an error is encountered or a case is not considered, return None.
        """
        if node is None:
            node = self.node

        if isinstance(node, nodes.BoolOp):
            node = self.parse_bool_op(node)
        elif isinstance(node, nodes.UnaryOp):
            node = self.parse_unary_op(node)
        elif isinstance(node, nodes.Compare):
            node = self.parse_compare(node)
        elif isinstance(node, nodes.BinOp):
            node = self.parse_bin_op(node)
        elif isinstance(node, nodes.Const):
            node = node.value
        elif isinstance(node, nodes.Name):
            node = self.apply_name(node.name)
        else:
            raise Z3ParseException(f"Unhandled node type {type(node)}.")

        return node
```

### Extending ExprWrapper to support container classes

To complete our task, we need to add a parse method for list/set/tuple, and extend the method `apply_bin_op` to support `in` and `not in` operations. Note that `not in` should be interpreted as a single expression, not as a combination of `not` and `in` , as it is represented as a single binop node. We can interpret `in` and `not in` as follows:

```python
a in lst -> a == lst[0] or a == lst[1] or ... or a == lst[n]
            any(a == e for e in lst) # written in Python expression

a not in lst -> a != lst[0] and a != lst[1] and ... and a != lst[n]
                all(a != e for e in lst) # written in Python expression
```

The code below shows the expanded `apply_bin_op` method. `z3.Or` and `z3.And` can take any number of parameters, so we construct the expression with a comprehension and use `*` to break the list to a sequence of arguments.

```python
    def apply_bin_op(
        self, left: z3.ExprRef, op: str, right: Union[z3.ExprRef, List[z3.ExprRef]]
    ) -> z3.ExprRef:
        """Given left, right, op, apply the binary operation."""
        try:
            if op == "+":
                return left + right
            elif op == "-":
                return left - right
            elif op == "*":
                return left * right
            elif op == "/":
                return left / right
            elif op == "**":
                return left**right
            elif op == "==":
                return left == right
            elif op == "<=":
                return left <= right
            elif op == ">=":
                return left >= right
            elif op == "<":
                return left < right
            elif op == ">":
                return left > right
            elif op == "in":
                return z3.Or(*[left == element for element in right])
            elif op == "not in":
                return z3.And(*[left != element for element in right])
            else:
                raise Z3ParseException(f"Unhandled binary operation {op}.")
        except TypeError:
            raise Z3ParseException(f"Operation {op} incompatible with types {left} and {right}.")
```

We also need a method to convert a List/Set/Tuple AST node to a list of Z3 expressions of the container elements. Here we directly store the Z3 expressions in a regular Python `list`. Though Z3 provides expressions like `Array` to store a list of elements, according to its documentation `Array` should only be used to handle unbounded or very large lists due to its inefficiency, and would be an overkill for small lists with fixed elements.

```python
    def parse_container_op(
        self, node: Union[nodes.List, nodes.Set, nodes.Tuple]
    ) -> List[z3.ExprRef]:
        """Convert an astroid List, Set, Tuple node to a list of z3 expressions."""
        return [self.reduce(element) for element in node.elts]
```

Finally, we need to include `parse_container_op` to the `reduce` method.

```python
    def reduce(self, node: astroid.NodeNG = None) -> z3.ExprRef:
        """
        Convert astroid node to z3 expression and return it.
        If an error is encountered or a case is not considered, return None.
        """
        if node is None:
            node = self.node

        if isinstance(node, nodes.BoolOp):
            node = self.parse_bool_op(node)
        elif isinstance(node, nodes.UnaryOp):
            node = self.parse_unary_op(node)
        elif isinstance(node, nodes.Compare):
            node = self.parse_compare(node)
        elif isinstance(node, nodes.BinOp):
            node = self.parse_bin_op(node)
        elif isinstance(node, nodes.Const):
            node = node.value
        elif isinstance(node, nodes.Name):
            node = self.apply_name(node.name)
        elif isinstance(node, (nodes.List, nodes.Tuple, nodes.Set)):
            node = self.parse_container_op(node)
        else:
            raise Z3ParseException(f"Unhandled node type {type(node)}.")

        return node
```

After completing the update, we need to write a unit test in `test_z3_visitor` to test the visitor's performance on container types.

```python
def test_container_constraints():
    z3v = Z3Visitor()
    code = """
    def f(x: int):
        '''
        Preconditions:
            - x in [1, 2, 3]
            - x not in {4, 5, 6, 7, 8}
            - x in (1, 2)
        '''
        pass
    """
    mod = z3v.visitor.visit(astroid.parse(code))
    function_def = mod.body[0]
    # Declare variables
    x = z3.Int("x")
    # Construct expected
    expected = [
        z3.Or(x == 1, x == 2, x == 3),
        z3.And(x != 4, x != 5, x != 6, x != 7, x != 8),
        z3.Or(x == 1, x == 2),
    ]
    actual = function_def.z3_constraints
    assert actual != []
    for e, a in zip(expected, actual):
        solver = z3.Solver()
        solver.add(e == a)
        assert solver.check() == z3.sat
```