# Pulsar: a static analyzer for the J* language

Pulsar is a static analyzer and linter for the [J*](https://github.com/bamless/jstar) language
written in **J\***.

## Supported features

The analysis of source files is broken down in multiple passes each reporting a specific class of
errors. For now all passes are always enabled, but in the near future they will be toggleable via
command line arguments.

### Syntax analysis pass
Or more simply, the parser. This reports syntax errors and some more semantic ones like assignment 
to a non-lvalue expression. This pass does exactly the same things that the `jstar` command line
interface and the `jstarc` compiler already do, and its main use is to obtain a correctly formed
parse tree on which to apply subsequent passses. If this pass fails with an error, no other pass
will be executed, as we do not support partial parsing of source code.

### Variable resolution pass
This pass is responsible for checking variable declarations and their use. It makes sure that
variables respect scoping rules and that are declared before usage. It also checks for possibly
uninitialize variables. A variable is considered uninitialized if its declaration doesn't have an initializer and its not assigned to a value in all execution paths prior to its usage.
Another check that is performed is the checking of declaration before usage of `static` global
variables.  
Examples:

resolve.jsr:
```lua
var a

if x
    a = 49
else
    print("Don't initialize `a`")
end

print(a)

fun foo()
    print(bar)
end

static var bar = "bar"
```
output:
```
Analyzing resolve.jsr...
File resolve.jsr [line 3]:

if x
   ^
Cannot resolve name 'x'
File resolve.jsr [line 9]:

print(a)
      ^
Variable 'a' (declared at line 1) may be uninitialized
File resolve.jsr [line 12]:

    print(bar)
          ^~~
Static name 'bar' referenced before declaration (line 15)
```

### Unused variables pass
This pass checks that all declared local variables (or static global ones) are actually used in the
program. Global variables are not checked as they form the API of the module and are accessible from
the outside. To make pulsar ignore a specific variable and not report an error, append a trailing 
underscore to its name.  
Examples:

unused.jsr:
```lua
fun test(arg1, arg2)
    var z = "Zed"
    print(arg1)
end

fun stub(arg_)
    raise NotImplementedException()
end
```
output:
```
Analyzing unused.jsr...
File test.jsr [line 1]:
fun test(arg1, arg2)
               ^~~~
Name 'arg2' declared but not used
File unused.jsr [line 2]:

    var z = "Zed"
        ^
Name 'z' declared but not used
```

### Illegal unpacks pass
This pass checks for illegal unpacking variable declarations or assignments.  
Examples:

unpack.jsr:
```lua
var a, b, c = 1, 2
```
output:
```
Analyzing unpack.jsr...
File unpack.jsr [line 1]:
var a, b, c = 1, 2
            ^
Too few values to unpack: expected 3, got 2
File unpack.jsr [line 2]:

a, b, c = "Not a Tuple"
        ^
Right hand side of unpack must be a Tuple or a List
```

### Return check pass
This pass reports non-void functions that don't return a result on all possible execution paths. A
function is considered non-void if it has at least one non-bare return (a return statement with an
expression).  
Examples:

return.jsr:
```lua
fun foo(x)
    if !x
        return false
    end
    print(x)
end
```
output:
```
Analyzing return.jsr...
File return.jsr [line 1]:
fun foo(x)
    ^~~
Non-void function 'foo' doesn't return a value on all execution paths
```

### Unreachable code pass
This pass checks for unreachable statements, for example statements that follow a `return` or a
`raise` that is always executed.  
Example:

unreachable.jsr
```lua
fun foo(x)
    return x
    print(x)
end

fun bar(y)
    try
        y += 2
    ensure
        raise Exception()
    end
    return y
end
```
output:
```
Analyzing unreachable.jsr...
File unreachable.jsr [line 3]:

    print(x)
    ^~~~~
Unreachable statement. Previous statement breaks control unconditionally
File unreachable.jsr [line 12]:

    return y
    ^~~~~~
Unreachable statement. Previous statement breaks control unconditionally
```

## How to use

Pulsar doesn't have any dependency (aside from the obvious one of **J\***) so using it is as simple
as cloning the repository:
```bash
https://github.com/bamless/pulsar.git
```
moving in its directory:
```bash
cd pulsar
```
and executing `pulsar.jsr` with a list of **J\*** files to check:
```bash
jstar pulsar.jsr file1.jsr file2.jsr file3.jsr

# Or more simply on unix-like systems
./pulsar.jsr file1.jsr file2.jsr file3.jsr
```