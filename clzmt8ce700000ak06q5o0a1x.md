---
title: "PyTA Project: Augment CFG Edges with Z3 Constraints"
datePublished: Fri Aug 09 2024 14:36:21 GMT+0000 (Coordinated Universal Time)
cuid: clzmt8ce700000ak06q5o0a1x
slug: pyta-project-augment-cfg-edges-with-z3-constraints
tags: python, graph, pylint, z3, depth-first-search, control-flow-graph

---

Today's task requires a combination of various components of PythonTA introduced in previous articles, including [control flow graph module](https://raineyang.hashnode.dev/pyta-project-depth-first-search-for-control-flow-graph), [Z3 visitor](https://raineyang.hashnode.dev/pyta-project-converting-function-preconditions-to-z3-constraints), and [Z3 expression wrapper](https://raineyang.hashnode.dev/pyta-project-converting-string-expressions-to-z3-constraints). In this task, we will augment each control flow graph edge with a list a Z3 constraints that represents the logical constraints of function arguments for this edge to be reached in the program. New constraints are created from function preconditions specified in the function docstring, `if` conditions, and `while` conditions. For example, considering the CFG below:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1723209583800/6766fc1e-210b-431f-8722-78944b8f001a.png align="center")

In this graph, `x` and `y` are function parameters, and the logical constraint on the starting edge is the function precondition. When the graph reaches an `if` statement or `while` loop, edges after these statements are added with the `if/while` condition (or its negation on the `False` branch). In addition, if a function argument is being reassigned, we would discard every Z3 constraint involving this variable in the edges following the reassignment, as in this case the variable's value becomes indeterminant.

### The Z3Environment Class

For this task, we first need to come up with a way to keep track of logical constraints and reassignment status of variables along an execution path. The `Z3Environment` class is used for this purpose. It has three attributes: `variable_unassigned` tracks whether a parameter has not been reassigned. `variable_type` stores the data type of each function parameter extracted of parameter type annotations. Currently, only `int` , `float` ,`bool`, `str` types and `list/set/tuple` literals can be processed by `ExprWrapper` , the class the converts python types to corresponding Z3 types. `constraints` represents the list of Z3 constraints along the current execution path.

```python
class Z3Environment:
    """Z3 Environment stores the Z3 variables and constraints in the current CFG path

    variable_unassigned:
        A dictionary mapping each variable in the current environment to a boolean indicating
        whether it has been reassigned (False) or remains unassigned (True).

    variable_type:
        A dictionary mapping each variable in the current environment to its data type.

    constraints:
        A list of Z3 constraints in the current environment.
    """

    variable_unassigned: Dict[str, bool]
    variable_type: Dict[str, str]
    constraints: List[ExprRef]

    def __init__(self, variables: Dict[str, ExprRef], constraints: List[ExprRef]) -> None:
        """Initialize the environment with function parameters and preconditions"""
        self.variable_unassigned = {var: True for var in variables.keys()}
        self.variable_type = variables
        self.constraints = constraints.copy()
```

For adding new Z3 constraints, we need the following two methods: `add_constraint` appends a Z3 constraint to the list of constraints, and `parse_constraint` converts an Astroid AST to an equivalent Z3 expression. Later, during the graph traverse, we will need to first call `parse_constraint` to convert the python expression (represented as Astroid AST) to Z3 constraint, and then add it to environment with `add_constraint`

```python
    def add_constraint(self, constraint: ExprRef) -> None:
        """
        Add a new z3 constraint to environment
        """
        self.constraints.append(constraint)

    def parse_constraint(self, node: NodeNG) -> Optional[ExprRef]:
        """
        Parse an Astroid node to a Z3 constraint
        Return the resulting expression
        """
        ew = ExprWrapper(node, self.variable_type)
        try:
            return ew.reduce()
        except (Z3Exception, Z3ParseException):
            return None
```

To handle variable reassignment, the `assign` method is called when reaching a reassignment statement, which marks the status of variable as `False` (not unassigned, which is, being assigned).

```python
    def assign(self, name: str) -> None:
        """Handle a variable assignment statement"""
        if name in self.variable_unassigned:
            self.variable_unassigned[name] = False
```

Finally, putting everything together, the `update_constraints` method is where we generate the constraints for an edge and remove constraints with reassigned variables. It needs a helper function `_get_vars` , which returns the variables included in a given Z3 expression.

```python
    def update_constraints(self) -> List[ExprRef]:
        """
        Returns all z3 constraints in the environments
        Removes constraints with reassigned variables
        """
        updated_constraints = []
        for constraint in self.constraints:
            # discard expressions with reassigned variables
            variables = _get_vars(constraint)
            reassigned = any(
                not self.variable_unassigned.get(variable, False) for variable in variables
            )
            if not reassigned:
                updated_constraints.append(constraint)

        self.constraints = updated_constraints
        return updated_constraints.copy()


def _get_vars(expr: ExprRef) -> Set[str]:
    """
    Retrieve all z3 variables from a z3 expression
    """
    variables = set()

    def traverse(e):
        if is_const(e) and e.decl().kind() == Z3_OP_UNINTERPRETED:
            variables.add(e.decl().name())
        elif hasattr(e, "children"):
            for child in e.children():
                traverse(child)

    traverse(expr)
    return variables
```

### Depth-First Traverse Through CFG

After setting up the Z3 environment, there is another preparation we need to make: getting all the paths along the control flow graph. The standard way to do so is to implement a depth-first search through the graph. I have created an implementation of DFS in a previous article ([https://raineyang.hashnode.dev/pyta-project-depth-first-search-for-control-flow-graph](https://raineyang.hashnode.dev/pyta-project-depth-first-search-for-control-flow-graph)). However, that one traverses through the nodes in the graph, while today we need to get all the edges along each path, which I implement in the code below:

```python
    def _get_paths(self) -> List[List[CFGEdge]]:
        """Get edges that represent paths from start to end node in depth-first order."""
        paths = []

        def _visited(
            edge: CFGEdge, visited_edges: Set[CFGEdge], visited_nodes: Set[CFGBlock]
        ) -> bool:
            return edge in visited_edges or edge.target in visited_nodes

        def _dfs(
            current_edge: CFGEdge,
            current_path: List[CFGEdge],
            visited_edges: Set[CFGEdge],
            visited_nodes: Set[CFGBlock],
        ):
            # note: both visited edges and visited nodes need to be tracked to correctly handle cycles
            if _visited(current_edge, visited_edges, visited_nodes):
                return

            visited_edges.add(current_edge)
            visited_nodes.add(current_edge.source)
            current_path.append(current_edge)

            if current_edge.target == self.end or all(
                _visited(edge, visited_edges, visited_nodes)
                for edge in current_edge.target.successors
            ):
                paths.append(current_path.copy())
            else:
                for edge in current_edge.target.successors:
                    _dfs(edge, current_path, visited_edges, visited_nodes)

            current_path.pop()
            visited_edges.remove(current_edge)

        _dfs(self.start.successors[0], [], set(), set())
        return paths
```

The algorithm is basically the same as the version with nodes. One thing worth noting is that we need to keep track of both visited nodes and edges, and visit an edge only if the edge itself and its target node are not visited, to handle cycles correctly. Considering the case below: in this `while` loop structure, the desired behavior is to form two execution paths, one into the loop and one out of the loop.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1723212337364/bebad1c0-a774-4ac5-b173-9c88b02d5851.png align="center")

However, if we only keep track of unvisited edges, the first traverse will go through both edges in the cycle, leading to the one path below. Thus, we also need to stop at visited nodes to handle cycles.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1723212460307/4d48c9e0-4e16-4394-a3d7-413371f36602.png align="center")

### Add Z3 Constraints to CFG Edges

Now it's time to put everything together in the `update_edge_z3_constraint` method in `ControlFlowGraph` class:

```python
    def update_edge_z3_constraints(self) -> None:
        """Traverse through edges and add Z3 constraints on each edge.

        Constraints are generated from:
        - Function preconditions
        - If conditions
        - While conditions

        Constraints with reassigned variables are not included in subsequent edges."""
```

First, we iterate through the paths generated in `_get_paths` method and create a new `Z3Environment` along each path. An edge with a non-None `condition` indicates a `if` or `while` condition, which we add it (or its negation, if the edge is labeled `False`) to the current `Z3Environment` . Finally, we invoke `update_constraints` to every edge in the path.

```python
        for path_id, path in enumerate(self._get_paths()):
            # starting a new path
            z3_environment = Z3Environment(self._z3_vars, self.precondition_constraints)
            for edge in path:
                # traverse through edge
                if edge.condition is not None:
                    condition_z3_constraint = z3_environment.parse_constraint(edge.condition)
                    if condition_z3_constraint is not None:
                        if edge.label == "True":
                            z3_environment.add_constraint(condition_z3_constraint)
                        elif edge.label == "False":
                            z3_environment.add_constraint(Not(condition_z3_constraint))

                edge.z3_constraints[path_id] = z3_environment.update_constraints()
```

One thing to note about is that we are storing Z3 constraints on each edge in a `Dict[int, List]`, where the key is `path_id` and value is the list of constraints. This is because one edge can be part of multiple paths, each with its own sets of constraints. The `path_id` will be used to distinguish which list of constraints belongs to which path.

```python
                # traverse into target node
                for node in edge.target.statements:
                    if isinstance(node, Assign):
                        # mark reassigned variables
                        for target in node.targets:
                            if isinstance(target, AssignName):
                                z3_environment.assign(target.name)
```

After that, we need to iterate through statements in the node after the edge to determine if there are any arguments being reassigned.

At this point, the main part of the task is completed, but there are still a few minor points worth noting. Firstly, `z3` is an optional dependency in PythonTA, so we should also make sure that the functionality of control flow graph itself it not affected even if z3-related features are not enabled. We need to put the import statements related to z3 in a `try/except` block, which marks the variable `z3_dependency_available` as `False` if an `ImportError` occurs.

```python
try:
    from z3 import Z3_OP_UNINTERPRETED, ExprRef, Not, Z3Exception, is_const

    from ..transforms import ExprWrapper

    z3_dependency_available = True
except ImportError:
    ExprRef = Any
    ExprWrapper = Any
    Not = Any
    Z3Exception = Any
    is_const = Any
    Z3_OP_UNINTERPRETED = Any
    z3_dependency_available = False
```

If z3 is not successfully imported, we would disable the feature `update_edge_z3_constraints`

```python
    def update_edge_z3_constraints(self) -> None:
        """Traverse through edges and add Z3 constraints on each edge.

        Constraints are generated from:
        - Function preconditions
        - If conditions
        - While conditions

        Constraints with reassigned variables are not included in subsequent edges."""
        if not z3_dependency_available:
            return
```

Secondly, we need to invoke `Z3Visitor` at some point to set up initial z3 constraints from function precondition docstrings. Initially I try to directly integrate it into `CFGVisitor` module. However, this turns out to be less optimal as it introduces coupling between two modules that are, in principle, unrelated. Instead, inside `visit_functiondef` method in `CFGVisitor` we would conditionally call `update_edge_z3_constraints` only if the `FunctionDef` node has the `z3_constraints` attribute added by `Z3Visitor`

```python
        if hasattr(func, "z3_constraints"):
            self._current_cfg.precondition_constraints = func.z3_constraints
            self._current_cfg.update_edge_z3_constraints()
```