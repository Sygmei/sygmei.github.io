---
title: "Python and the phantom parameters, part 2"
date: 2020-05-23T15:34:30-04:00
categories:
  - programming
tags:
  - python
---

```python
@inject_args("a", "b", "c")
def function_which_accept_no_parameters():
    return a + b * c

print(function_which_accept_no_parameters(10, 5, 3))
```

One snippet is worth thousand of words they say.
That's why today, I'm welcoming you with some Python code which I hope will trigger your interest.

Today's challenge is to write a decorator named `inject_args` which will allow us to transform a function which takes no parameters to a function which takes some parameters.

## The decorator

Some of you might already have written a decorator but for those who never did, I will explain how it works.

A decorator in Python can be seen as some syntaxic sugar over a function transformation.

```python
@decorate
def my_function():
    pass

def my_other_function():
    pass

# This is exactly the same thing
my_other_function = decorate(my_other_function)
```

The decorator `decorate` is also a function which takes a function as an argument.
But of course it can become a bit more complicated than that.

In our case, the `inject_args` decorator will look like this :

```python
def inject_args(*arguments):
    def inject_args_function_stage(function):
        def inject_args_call_stage(*args, **kwargs):
            pass
        return inject_args_call_stage
    return inject_args_function_stage

@inject_args("a", "b", "c")
def my_function():
    return a + b * c

my_function(10, 20, 30)
```

***Why is there 3 functions ?***

If you never worked with decorators before (or at least, never wrote one), all of this can seems a bit confusing.

Our decorator needs three stages :
- One stage to get the decorator arguments, this is the top stage named `inject_args`, this is the one we will use directly to decorate our function
- One stage to get the function we decorate, this is the stage named `inject_args_function_stage`
- One stage to proxify the arguments passed to the decorated function, this is the stage named `inject_args_call_stage`

To break it down, let's remember what a decorator really is : a function which transforms a function passed in argument. We could rewrite it like this :
```python
def my_function():
    return a + b * c

#                         decorator     function
my_function = inject_args("a", "b", "c")(my_function)
#           call
my_function(10, 20, 30)
```

Since `inject_args` returns `inject_args_function_stage`, we can directly chain-call with `(my_function)` and that itself will return a function of type `inject_args_call_stage` which will call the underlying function `my_function`.

***Why is `inject_args_function_stage` defined in `inject_args` and `inject_args_call_stage` defined inside `inject_args_function_stage` ?***

That's a valid point, why defining these functions inside each other ? Why not defining them as free functions ?

This is to use one powerful Python feature : closures.

I will not go in details on how closure works because it deserves an article on its own but basically, closures allow functions to capture some elements it needs to properly works.

I will go back on this later.

## Argument injection

Once again, let's take a look at the initial snippet :
```python
def function_which_accept_no_parameters():
    return a + b * c
```

***Why does Python allows us to write a function like this if `a`, `b` and `c` are not defined ?***

Easy answer : Python expects them to exists as global variable when we will call `function_which_accept_no_parameters`.
It's fine if these variables were nowhere to be found when we defined the function, Python only wants them to exist it is time for them to be used.

We can check this assumption by disassembling `function_which_accept_no_parameters` with :
```python
import dis

dis.dis(function_which_accept_no_parameters)
```

which will prints the following :

```
  2           0 LOAD_GLOBAL              0 (a)
              2 LOAD_GLOBAL              1 (b)
              4 LOAD_GLOBAL              2 (c)
              6 BINARY_MULTIPLY
              8 BINARY_ADD
             10 RETURN_VALUE
```

that's quite a lot of information !

***Now you have all these weird `LOAD_GLOBAL`, `BINARY_MULTIPLY`, `BINARY_ADD` keywords, and these numbers, what do they represent ?***

- The first column with the `2` on the left just indicates the line number, line `1` being the `def function_which_accept_no_parameters():`.
- The second column with the 0, 2, ..., 10 represents the index in the bytecode array.
- The third column is the opcode at the given index (0, 2, ..., 10).
- The fourth column (0, 1, 2) is the value in the bytecode at the index + 1 (1, 3, 5) for the (0, 1, 2) values.
- The fifth column is a hint showing what really is the index in the fourth column.

***That's a lot of informations ! Bytecode ? Opcodes ? Why is there no numbers in the fourth column for the last lines ?***

That's a lot of questions, I will answer them one by one.

First of all, here is a little glossary :

- **Bytecode** : Bytecode is a form of instruction set designed for efficient execution by a software interpreter.
- **Opcode** : In computing, an opcode (abbreviated from operation code, also known as instruction code) is the portion of a machine language instruction that specifies the operation to be performed.

As you can see, `LOAD_GLOBAL`, `BINARY_MULTIPLY`, `BINARY_ADD` and `RETURN_VALUE` are here to explain to the Python interpreter, what kind of operations it should do.

When you look at the initial code, you can find these operations :
```python
return a + b * c

# a: LOAD_GLOBAL
# b: LOAD_GLOBAL
# c: LOAD_GLOBAL
# b * c: BINARY_MULTIPLY
# a + (result of b * c): BINARY_ADD
# return: RETURN_VALUE
```

And the reason why there is no numbers in the fourth column for the last lines is that these opcodes don't require what we call : an operand.
- **Operand** : Depending on architecture, the operands may be register values, values in the stack, other memory values

An operand is a value associated to an opcode that will specify on what should the opcode operate.
`BINARY_MULTIPLY`, `BINARY_ADD`, `RETURN_VALUE` don't need such operands because unlike `LOAD_GLOBAL`, they know on what data they will work.

***How do they know on what data they will work if you don't specify it ? I could want to do a + b * c or a * b + c***

That is because the Python interpreter (like many interpreters) is stack-based.

Instead of requiring operands, opcodes like `BINARY_ADD` will require a certain amount of values to be pushed on top of the stack and will consume them.

If we look at `BINARY_ADD` documentation [here](https://docs.python.org/3/library/dis.html#opcode-BINARY_ADD) it says :

```
BINARY_ADD
    Implements TOS = TOS1 + TOS.
```

***What are TOS and TOS1 ?***

TOS means Top Of Stack, basically, the value on the top of the stack, TOS1 is the value just under the TOS, TOS2 is under TOS1 etc..
A stack is basically a list where the last element added will need to be popped out of the list to be able to see the other elements.

We could represent the operation that way :
![](https://raw.githubusercontent.com/Sygmei/sygmei.github.io/master/assets/_posts/phantom_parameters_python/BINARY_ADD.png)