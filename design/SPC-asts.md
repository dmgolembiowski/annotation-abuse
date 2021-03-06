# SPC-asts
The `inrange` macro shall read the annotations of the class variables in the class definition, and generate properties with the same names as the annotated class variables. Annotations for class variables are stored in the `MyClass.__annotations__` dictionary. If this class attribute does not exist, it means that there are no annotated class variables. An exception shall be raised if there are no annotations.

For the class definition
```python
class MyClass:
    foo: "0 < foo < 10"
```
the macro should generate a class equivalent to
```python
class MyClass:
    foo: "0 < foo < 10"

    def __init__(self):
        self._foo = None

    @property
    def foo(self):
        return self._foo

    @foo.setter
    def foo(self, value):
        if (value > 0) and (value < 10):
            self._foo = value
```

The process for constructing the class is as follows:
- Collect the class variables with annotations.
- Parse each annotation into an AST.
- Extract the endpoints of each range.
- Construct each getter AST.
- Construct each setter AST.
- Convert the getter/setter ASTs to functions.
- Construct a property from the getter and setter functions.
- Add all new properties to the class.
- Construct an `__init__` method AST.
- Modify the `__init__` AST to create the instance variables that back the properties.
- Compile the `__init__` AST to a function.
- Bind the `__init__` function to the class.

## [[.item]]
The information about each class variable that has been selected for processing shall be bundled into an instance of the `MacroItem` class. The `MacroItem` class shall store the following pieces of data:
- The name of the class variable.
- The original annotation string.
- The numeric value of the range's lower bound.
- The numeric value of the range's upper bound.
- The getter function.
- The setter function.
- The AST of the initialization statement.

## [[.decorator]]
The decorator shall only be applied to a class definition. A `MacroError` shall be raised if the decorator is applied to an object for which `type(my_object)` is not `type`.

The decorator should follow this sequence of events:
- Check the type of the argument.
- Check that annotations exist.
- Return the new class definition.

### Unit Tests
Invalid inputs:
- [[.tst-rejects_funcs]]: Test that the decorator raises a `MacroError` when applied to a function definition.
- [[.tst-rejects_methods]]: Test that the decorator raises a `MacroError` when applied to a method definition.
- [[.tst-no_annotations]]: Test that applying the decorator to a class with no annotations raises a `MacroError`.

## [[.collect]]
For each `field: annotation` in `cls.__annotations__` the processor shall:
- Skip `field` if `annotation` is not a string.
- Construct a `MacroItem` from `field` and `annotation`.
- Add the item to `self._items`.

### Unit Tests
Invalid inputs:
- [[.tst-no_strings]]: Test that a `MacroError` is raised if none of the annotations are strings.
Valid inputs:
- [[.tst-mixed_strings]]: Test that class variables with string annotations are collected.

## [[.parse]]: Parse annotations into ASTs
The annotation should be passed straight to `ast.parse`, which will return an `ast.Module`. The `ast.Compare` node shall be extracted from the module AST.

If the string is properly formatted, the module's `body` field will contain a single item: an `ast.Expr` node. The expression's `value` field should be an `ast.Compare` node.

### Unit Tests
Invalid inputs:
- [[.tst-not_comparison]]: Test that a `MacroError` is raised when nodes not of type `ast.Compare` are found in the parsed annotation.
Valid inputs:
- [[.tst-comparison]]: Test that the processor correctly extracts the `ast.Compare` node when it is present.

## [[.extract]]: Extract range endpoints
An expression of the form `x < y < 5` will produce the following node:
```
Compare(left=Name(id='x', ctx=ast.Load()),
        ops=[
            ast.Lt(),
            ast.Lt(),
        ],
        comparators=[
            ast.Name(id='y', ctx=ast.Load()),
            ast.Num(n=5),
        ],
)
```
Note that although you would typically read this expression as "y is greater than x and less than 5", the node stores this as "x is less than y, which is less than 5".

A value that appears with a negative sign (`-`) in front of it will be wrapped in an `ast.UnaryOp` node, like so:
```
ast.UnaryOp(op=ast.USub(), operand=ast.Num(n=5))
```

`InRangeProcessor` shall extract endpoints that are positive or negative numbers.

### Unit Tests
Valid inputs:
- [[.tst-valid_ints]]: Test that valid integer ranges are extracted.
- [[.tst-valid_floats]]: Test that valid floating point ranges are extracted.
Invalid inputs:
- [[.tst-rejects_inf_nan]]: Test that `NaN` and `inf` are rejected in ranges.
Semantics:
- [[.tst-order]]: Test that ranges are rejected when the left literal is greater than the right literal.
- [[.tst-equal]]: Test that ranges are rejected when the two literals are equal.

## [[.ast2func]]
There shall be a method `ast_to_func` that converts an AST into a function. The method shall take the AST and the name of the function as arguments.

### Unit Tests
Valid inputs:
- [[.tst-func-roundtrip]]: Test that a function converted to an AST and back again still works.

## [[.getter]]:
The getter function for the property will have the name `var_getter`, where `var` is the name of the corresponding class variable. The getter shall return the attribute `self._var`. The getter function shall be stored in the `MacroItem` for the corresponding class variable.

## [[.setter]]
The setter function for the property will have the name `var_setter`, where `var` is the name of the corresponding class variable. The setter shall only set the value of `self._var` when the provided value is within the range specified in the annotation. The setter function shall be stored in the `MacroItem` for the corresponding class variable.

### Unit Tests
Invalid inputs:
- [[.tst-outside_range]]: Test that the setter rejects values not in the specified range.
Valid inputs:
- [[.tst-in_range]]: Test that the setter accepts values in the specified range.

## [[.property]]
A property shall be constructed from the getter and setter functions stored in the `MacroItem` instance.

## [[.initast]]: Construct `__init__` AST
When no `__init__` is included with the class definition, the processor shall construct an AST equivalent to
```
def __init__(self):
    super().__init__()
```
The processor shall have a static method named `InRangeProcessor._make_empty_init_ast()` that produces this AST.

## [[.statements]]: Add initializations to `__init__`
A statement of the form `self._var = None` shall be appended to the `__init__` AST for each class variable selected for processing. These statements initialize the instance attributes that store the data used by the generated properties.

For a class definition
```
class MyClass:
    foo: "0 < foo < 1"
    bar: "1 < bar < 2"
```
the following statements should be added to `__init__`:
```
self._foo = None
self._bar = None
```

### Unit Tests
Basic function:
- [[.tst-init_stmts]]: Test that the backing instance variables are created.

## [[.bind]]: Bind `__init__` to class
To bind the `__init__` function to the class you execute the following line:
```python
setattr(self.cls, "__init__", init_func.__get__(self.cls))
```
