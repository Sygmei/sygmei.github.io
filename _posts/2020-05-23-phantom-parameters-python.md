---
title: "Python and the phantom parameters, part 2"
date: 2020-05-23T15:34:30-04:00
categories:
  - programming
tags:
  - python
---

One snippet is worth thousand of words they say.
That's why today, I'm welcoming you with some Python code which I hope will trigger your interest.

```python
@inject_args("a", "b", "c")
def function_which_accept_no_parameters():
    return a + b * c

print(function_which_accept_no_parameters(10, 5, 3))
```

Today's challenge is to write a decorator named `inject_args` which will allow us to transform a function which takes no parameters to a function which takes some parameters.

Of course if we try to run the function right now, we will get the following messge :
```python
>>> function_which_accept_no_parameters(10, 20, 30)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: function_which_accept_no_parameters() takes 0 positional arguments but 3 were given
```

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

## Understanding Python bytecode

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

See ? There is three `LOAD_GLOBAL` wandering there !

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

![BINARY_ADD stack schema](https://raw.githubusercontent.com/Sygmei/sygmei.github.io/master/_posts/phantom_parameters_python/BINARY_ADD.png)

## The secret behind Python functions

Okay, all of this bytecode stuff is cool but that does not help us with our argument injection problem.. or does it ?

Another thing I want to show you before starting to write some code is how Python functions are made.

We can check this using the `dir()` function :

```python
def function_which_accept_no_parameters():
    return a + b * c

dir(function_which_accept_no_parameters)
```

will output :
```
['__annotations__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
```

That's a lot of things ! I could go and try to explain what does each of these elements but that is not the point of this post. Let's concentrate on the the juicy part, the `__code__` attribute.

Once again let's inspect it :
```python
dir(function_which_accept_no_parameters.__code__)
```

has the following output :
```python
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'co_argcount', 'co_cellvars', 'co_code', 'co_consts', 'co_filename', 'co_firstlineno', 'co_flags', 'co_freevars', 'co_kwonlyargcount', 'co_lnotab', 'co_name', 'co_names', 'co_nlocals', 'co_posonlyargcount', 'co_stacksize', 'co_varnames', 'replace']
```

Let's ignore the dunder attributes / methods and let's concentrate on the `co_xxxx` attributes.

```python
for item in dir(function_which_accept_no_parameters.__code__):
    if item.startswith("co_"):
        print(item, getattr(function_which_accept_no_parameters.__code__, item))
```

will output :
```
co_argcount 0
co_cellvars ()
co_code b't\x00t\x01t\x02\x14\x00\x17\x00S\x00'
co_consts (None,)
co_filename <stdin>
co_firstlineno 1
co_flags 67
co_freevars ()
co_kwonlyargcount 0
co_lnotab b'\x00\x01'
co_name function_which_accept_no_parameters
co_names ('a', 'b', 'c')
co_nlocals 0
co_posonlyargcount 0
co_stacksize 3
co_varnames ()
```

again, a lot of information ! Before anything, let's try to compare this output with the one of a function with correct parameters :

```python
def function_with_parameters(a, b, c):
    return a + b * c

for item in dir(function_with_parameters.__code__):
    if item.startswith("co_"):
        print(item, getattr(function_with_parameters.__code__, item))
```

```
co_argcount 3
co_cellvars ()
co_code b'|\x00|\x01|\x02\x14\x00\x17\x00S\x00'
co_consts (None,)
co_filename <stdin>
co_firstlineno 1
co_flags 67
co_freevars ()
co_kwonlyargcount 0
co_lnotab b'\x00\x01'
co_name function_with_parameters
co_names ()
co_nlocals 3
co_posonlyargcount 0
co_stacksize 3
co_varnames ('a', 'b', 'c')
```

Here is a table reporting all the differences
| Attribute name | `function_which_accept_no_parameters` | `function_with_parameters` |
| -------------- | ------------------------------------- | -------------------------- |
| co_argcount | 0 | 3 |
| co_code | b't\x00t\x01t\x02\x14\x00\x17\x00S\x00' | b'\|\x00\|\x01\|\x02\x14\x00\x17\x00S\x00' |
| co_name | function_which_accept_no_parameters | function_with_parameters |
| co_names | ('a', 'b', 'c') | () |
| co_nlocals | 0 | 3 |
| co_varnames | () | ('a', 'b', 'c') |

This is pretty interesting and gives a clearer idea on how Python works with function arguments.
Here is a definition of the different arguments from the [Python documentation](https://docs.python.org/3/library/inspect.html#types-and-members).

- co_argcount : number of arguments (not including keyword only arguments, * or ** args)
- co_code : string of raw compiled bytecode
- co_name : name with which this code object was defined
- co_names : tuple of names of local variables
- co_nlocals : number of local variables
- co_varnames : tuple of names of arguments and local variables

`co_name` value obviously differs since the two functions don't have the same name so that is not really interesting in our case.

The `co_argcount` attribute is pretty self-explanatory here since `function_which_accept_no_parameters` takes 0 arguments and `function_with_parameters` takes 3.

For the `co_names` attribute, you might have noticed the "tuple of names of local variables". Why is there global variables in there then ? I did a little [research](https://github.com/python/cpython/pull/2743) and it seems that there is bug in the documentation. For out first function it contains three elements : `('a', 'b', 'c')`, these are the names of the "global" variables used in `function_which_accept_no_parameters`. For `function_with_parameters` the tuple is empty since they are local variables.

For the `co_nlocals` attribute, it seems logical that `function_which_accept_no_parameters` has 0 local variables (they are all considered global variables) and `function_with_parameters` has 3 since arguments are indeed local variables.

Finally, the `co_varnames` is pretty much the `co_names` situation but mirrored. `function_with_parameters` has a tuple with three values `('a', 'b', 'c')` which are the names of the local variables and `function_which_accept_no_parameters` has an empty tuple which means, no local variables.

If you are following, you probably noted that I forgot one attribute : `co_code`, but this one requires a dedicated section.

## The co_code and its co_mplexity

`b't\x00t\x01t\x02\x14\x00\x17\x00S\x00'` ***?*** `b'|\x00|\x01|\x02\x14\x00\x17\x00S\x00'` ***?*** ***What is this cryptic alien language ?***

Check the first character of these values : `b`. It means we are dealing with `bytes`. For readability's sake, I'll transform these `bytes` to an integer array with the following code :

```python
bytecode = function_which_accept_no_parameters.__code__.co_code
bytecode = list(no_args_bytecode)
print(bytecode)
```

which will output the following value :
```python
[116, 0, 116, 1, 116, 2, 20, 0, 23, 0, 83, 0]
```
and for `function_with_parameters` :
```python
[124, 0, 124, 1, 124, 2, 20, 0, 23, 0, 83, 0]
```

Doing that makes it easy to spot the difference between the bytecode of our two functions.
Everything is perfectly identical except for the three occurences of the `116` values which becomes `124`.

Truth is, opcodes are actually encoded as numbers, it is way more efficient for the computer to have opcodes as numbers than opcodes as string.
We can access the association opcode name to number using the `dis.opmap` variable.

Before checking what is the opcode, let's study the structure of this bytecode array. Remember what I said before ? Opcodes are on even indexes while operands are on odd indexes :

```python
  # opcode  # operand
[ 116,      0,
  116,      1,
  116,      2,
  20,       0,
  23,       0,
  83,       0]
```

To be honest, I lied before :
>
    And the reason why there is no numbers in the fourth column for the last lines is that these opcodes don't require what we call : an operand.

As you can see, all opcodes actually do have operands, but for the opcodes that don't require them operands are just for padding purpose.
This behaviour was different prior to Python 3.6, here is the same bytecode in Python 3.5 :
```python
[116, 0, 0, 116, 1, 0, 116, 2, 0, 20, 23, 83]
```
***Okay, I understand that opcodes were not padded before but why is there another trailing `0` after each `116` operand ?***

Since each element of the bytecode is a "byte", it means the maximum value is 255. For opcodes this is not a problem since the opcode maximum value is `163` as of Python 3.8 but for operands you might need higher values.
Imagine you want to do a `LOAD_GLOBAL` on the 1000th "name", you will have to do `[116, 232, 3]` which means "Do a `LOAD_GLOBAL (116)` on the `3 * 256 + 232` element.

***So how does Python 3.8 achieve this if there is only one byte left for the operand ? Are we limited to 256 values ?***

Nope ! Python 3 uses another opcode named `EXTENDED_ARG` which will make the next opcode's operand a 16-bit word instead of a 8-bit one. `EXTENDED_ARG` operand will become the value of the first 8-bit of the 16-bit word and the next opcode's operand will be the last 8-bit. You can chain two `EXTENDED_ARG` or more if a 16-bit word is not enough.
So for Python 3.8, loading the 1000th global becomes :
```python
[144, 3, 116, 232]
```

**Back on topic !**

By now we should have a good idea on what we should patch in order for our function to accept three arguments :
- Replace the `116` in the `co_code` to `124`
- Replace the `0` in `co_argcount` to `3`
- Put an empty tuple in the `co_names` attribute
- Replace the `0` in `co_nlocals` to `3`
- Put the tuple `('a', 'b', 'c')` in `co_varnames`

You might have noticed that there was a method named `replace` when we did the `dir(function_which_accept_no_parameters.__code__)` earlier.
Let's use that to test if it works !
```python
bytecode = function_which_accept_no_parameters.__code__.co_code
function_which_accept_no_parameters.__code__ = function_which_accept_no_parameters.__code__.replace(
    co_argcount=3,
    co_varnames=("a", "b", "c"),
    co_nlocals=3,
    co_names=(),
    co_code=bytes([byte if byte != 116 else 124 for byte in bytecode])
)
```

So far, no errors, let's test our function now :
```python
>>> function_which_accept_no_parameters(10, 20, 30)
610
```
Yay ! Good result !
Only problem is that our values are hard-coded right now, we have our proof-of-concept but let's transform it to a fully fledged prototype !

## Putting the pieces together

Here is the decorator we wrote earlier :
```python
def inject_args(*arguments):
    def inject_args_function_stage(function):
        def inject_args_call_stage(*args, **kwargs):
            pass
        return inject_args_call_stage
    return inject_args_function_stage
```
Let's do a list again on what we need to change in the `__code__` object without hardcoding values :
- Replace the `116` in the `co_code` to `124` when `co_names[operand]` is in `arguments`
- Replace the `x` in `co_argcount` to `x + len(arguments)`
- Remove elements in `co_names` that are also in `arguments`
- Replace the `y` in `co_nlocals` to `y + len(arguments)`
- Append the list `arguments` to the tuple `co_varnames`

Before writing the actual code, I will need a few helper functions :
```python
def patch_sequence(bytecode, sequence_source, sequence_dest):
    sequence_source = "".join([chr(seq_item) for seq_item in sequence_source])
    sequence_dest = "".join([chr(seq_item) for seq_item in sequence_dest])
    return bytecode.replace(sequence_source, sequence_dest)
```

This function will allow us to easily patch a bytecode sequence. This is useful because now we can't blindly patch all `LOAD_GLOBAL` to `LOAD_FAST` since there might be other "real" globals in our function.