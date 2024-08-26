This is a reupload of [this post](https://discord.com/channels/267624335836053506/1167105029426131014) for the sake of getting it out there.

# Introduction

So you've made a calculator that can add, subtract, multiply or divide two numbers. But can we make it work for more than two numbers? How about a calculator that works for arbitrary expressions like `4*3+9` or `5*(-1+3)`?

[`eval()` might immediately come to mind.](https://discord.com/channels/267624335836053506/1075957447677706292) If `eval()` can already evaluate arbitrary expressions for us, so why reinvent the wheel?

```python
expr = input("Enter expression: ")
result = eval(expr)
print(f"Result: {result}")
```

The answer is: [`eval()` is a bit *too* powerful](https://discord.com/channels/267624335836053506/1149228330147647509). If you were to add something like this to your Discord bot, for example, then users may be able to mess with your bot through this backdoor.

Instead, we could write a full-fledged parser and evaluator like [what this person made](https://discord.com/channels/267624335836053506/1097748826854539365/1109632834131480697). But if you don't mind losing a bit of flexibility, you could use the parser included in the Python standard library: `ast`.

# Using `ast.parse()`

First, we can parse a string into a syntax tree with `ast.parse()`. We can use `ast.dump()` to see what the output looks like:

```python
>>> import ast
>>> ast.parse("1 + 2")
<ast.Module object at 0x000001A1350239D0>
>>> print(ast.dump(_, indent=4))
Module(
    body=[
        Expr(
            value=BinOp(  # We're mostly interested in this part here
                left=Constant(value=1),
                op=Add(),
                right=Constant(value=2)))],
    type_ignores=[])
>>>
```

Note how `1 + 2` gets parsed into a `BinOp` (binary operation) with attributes `left`, `right` and `op` (operator) that store the operator and operands. Now see what happens with a slightly more complicated expression:

```python
>>> ast.parse("1 + 2 * 3")
<ast.Module object at 0x000001A1350238D0>
>>> print(ast.dump(_, indent=4))
Module(
    body=[
        Expr(
            value=BinOp(  # Again, this is the important bit
                left=Constant(value=1),
                op=Add(),
                right=BinOp(
                    left=Constant(value=2),
                    op=Mult(),
                    right=Constant(value=3))))],
    type_ignores=[])
>>>
```

We still have a `BinOp` expression for addition, but this time, the `right` attribute is _another_ `BinOp` expression for multiplication! Note how the `BinOp` for multiplication was deeper down the tree since it has a higher precedence than addition.

# Creating a Visitor

## Base Program

Once we have a syntax tree, we can create a visitor to walk through each node in the tree and evaluate it (if you're interested, you can read a bit more about the visitor pattern [here](https://craftinginterpreters.com/representing-code.html#working-with-trees)(warning: java)).

```python
import ast

class Evaluator(ast.NodeVisitor):
    pass  # We can add methods for visiting nodes here

expr = ast.parse(input("Enter expression: "))
result = Evaluator().visit(expr)
print("Result: {result}")
```

## Getting to the Expression

We can add simple visitor functions for `Module` and `Expr` that just 'unpacks' the expression inside. We can also add a `generic_visit()` method that will be called when we encounter a node we haven't seen yet.

```python
class Evaluator(ast.NodeVisitor):
    def visit_Module(self, node):
        if len(node.body) != 1:  # ensure we're only calculating 1 expression
            raise ValueError("Invalid input")
        return self.visit(node.body[0])

    def visit_Expr(self, node):
        return self.visit(node.value)

    def generic_visit(self, node):
        # Called when we don't have a visitor function for that node
        raise ValueError(f"Invalid node: {node}")
```

## Constants

`Constant` nodes hold literal primitive values, like `1`, `2.5` and `"foo"`. Since we're making a calculator, we can filter non-numeric values to ensure we're only working with numbers:

```python
class Evaluator(ast.NodeVisitor):
    ...
    def visit_Constant(self, node):
        if not isinstance(node.value, int | float):
            raise ValueError("Non-numeric value was found")
        return node.value
```

## Unary Operations

Unary operations apply to 1 operand, and lets us represent negative numbers (`-5` is actually a unary minus and a `5` literal).
Unary operations (`UnaryOp` nodes) have two attributes: `op` (operator) and `operand`.
We could figure out the operation using an if or match statement, but since the operations are also tree nodes, we can add them to our visitor methods and use `self.visit()` to get the operator.
To calculate the value of the operand, we can pass it into `self.visit()` too! And just like recursion, it returns the resulting value _regardless of whether the operand is a `Constant`, `BinaryOp`, or something else_!
Once we have the operation and operand, it's as easy as calling the operation and returning the value:

```python
class Evaluator(ast.NodeVisitor):
    def visit_UAdd(self, _node): return operator.pos
    def visit_USub(self, _node): return operator.neg
    ...
    def visit_UnaryOp(self, node):
        operand = self.visit(node.operand)
        op = self.visit(node.op)
        return op(operand)
```

## Binary Operations

This is more or less the same as unary operators, but with 2 operands rather than 1.

```python
import ast
import operator

class Evaluator(ast.NodeVisitor):
    def visit_Add(self, _node): return operator.add
    def visit_Sub(self, _node): return operator.sub
    # etc. See the final program for the full list
    ...
    def visit_BinOp(self, node):
        left = self.visit(node.left)
        right = self.visit(node.right)
        op = self.visit(node.op)
        return op(left, right)
```

… and that's it! We now have a functional calculator that can evaluate arbitrary numeric expressions.

# Other Features

And that completes the base capabilities of the calculator! But we don't have to stop here. Here are a few more things we could add to our calculator for fun:

## Mathematical Constants

We could add constants like `pi` and `e`, too. Since they will be parsed as `Name` nodes, we can take the identifier and look it up in a dictionary of predefined constants:

```python
import ast
import math
import operator

constant_values = {
    "e": math.e,
    "pi": math.pi,
}

class Evaluator(ast.NodeVisitor):
    ...
    def visit_Name(self, node):
        if value := constant_values.get(node.id):
            return value
        else:
            raise ValueError(f"Invalid name: {node.id}")
```

## Functions

We can add specific functions as well, using the `Call` node. For the sake of simplicity, I will only allow functions that have one input:

```python
functions = {
    "abs": abs,
    "sin": math.sin,
    # etc.
}

class Evaluator(ast.NodeVisitor):
    ...
    def visit_Name(self, node):
        if value := constant_values.get(node.id):
            return value
        elif func := functions.get(node.id):
            return func
        else:
            raise ValueError(f"Invalid constant/function: {node.id}")
    ...
    def visit_Call(self, node):
        func = self.visit(node.func)
        if len(node.args) != 1:
            raise ValueError(f"Functions only take 1 argument")
        arg = self.visit(node.args[0])
        return func(arg)
```

## Implicit Multiplication

Another advantage of using `ast.parse()` and a visitor rather than `eval()` is that it lets us customize some of the syntax. Since expressions like `5(2+3)` or `(1+3)(2+4)` are parsed as function calls but raise a runtime error since integers aren't callable. Instead, we can make our visitor automatically detect such patterns and multiply the values instead:

```python
class Evaluator(ast.NodeVisitor):
    ...
    def visit_Call(self, node):
        left = self.visit(node.func)
        is_function = isinstance(left, Callable)
        if len(node.args) != 1:
            if is_function:
                raise ValueError(f"Functions only take 1 argument")
            else:
                raise ValueError(f"Invalid right hand side of implicit multiplication")
        right = self.visit(node.args[0])
        if is_function:
            return left(right)
        else:
            return left * right
```

# Conclusion

That's how to make a calculator that calculates arbitrary expressions using `ast` to do the parsing for you. And since there's no way to execute arbitrary code on it (I think), it's much safer than `eval()`ing untrusted input. Feel free to adapt this however you like for your own projects!
