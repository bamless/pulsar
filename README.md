# Pulsar: a static analyzer for the J* language

Pulsar is a static analyzer and linter for the [J*](https://github.com/bamless/jstar) language
written in **J\***.

## Supported features

The analysis of source files is broken down into multiple passes each reporting a specific class of
errors. Each pass can be disabled at will, and the behaviour of some of them can be changed by
passing extra command line options. To list all available options simply pass the `-h` flag to
pulsar.

### Syntax analysis pass
Or more simply, the parser. This reports syntax errors and some more semantic ones like assignment 
to a non-lvalue expression. This pass does exactly the same things that the **J\*** runtime parser
already does, and its main use is to obtain a correctly formed parse tree on which to apply 
subsequent passses. If this pass fails with an error, no other pass will be executed, as we do not
support partial parsing of source code. Obviously, this pass can't be disabled.

### Semantic checking pass
This pass performs the same checks that the **J\*** runtime compiler already does, such as unpacking
declarations/assignments consistency, usage of break/continue inside loops and checking that return
statements are inside functions. This pass can't be disabled, as source files that don't pass these
checks are not considered valid **J\*** source files.

### Variable resolution pass
This pass is responsible for checking variable declarations and their use. It makes sure that
variables respect scoping rules and that are declared before usage. It also checks for possibly
uninitialized variables. A variable is considered uninitialized if its declaration doesn't have an 
initializer and its not assigned a value in all execution paths prior to its usage. Another check 
that is performed is the checking of declaration before usage of `static` global variables.  
This pass can be completely disabled by passing the `-v, --no-variable-resolution` option.  
By passing the `-g, --no-redefined-globals` this pass will not check for redefinition of global
variables, as they are technically allowed by the language.  
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
This pass can be disabled by passing the `-u, --no-unused` command line option.  
Examples:

unused.jsr:
```lua
fun test(arg1, arg2)
    var foo = "bar"
    print(arg1)
end

// pulsar will not warn for the unused `arg_` as its name ends with a trailing underscore
fun stub(arg_)
    raise NotImplementedException()
end
```
output:
```
Analyzing unused.jsr...
File unused.jsr [line 1]:
fun test(arg1, arg2)
               ^~~~
Name 'arg2' declared but not used
File unused.jsr [line 2]:

    var foo = "bar"
        ^~~
Name 'foo' declared but not used
```

### Return check pass
This pass reports non-void functions that don't return a result on all possible execution paths. A
function is considered non-void if it has at least one non-bare return (a return statement with an
expression).  
This pass can be disabled by passing the `-r, --no-check-returns` option.  
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
This pass can be disabled by passing the `-U, --no-unreachable` option.  
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

## Acces checking pass
This pass checks for and enforces naming conventions for private attributes/methods and constant
variables. Specifically, an attribute/method starting with an underscore (`_`) is considered private
to the current class and can't be accessed by anything but `this`. Variables are instead considered
constants if their names are composed only of capital letters and underscores. For constant
attribute variables, we do not warn if they are assigned to in the constructor, as otherwise there
won't be a way to initialize them.  
This pass can be disabled by passing the `-A, --no-access-check` option.  
Example:

access.jsr
```lua
var PI = 3.141592

class Foo
    fun new(bar)
        this._bar = bar
    end

    fun getBar()
        return this._bar
    end
end

var foo = Foo("bar")

print(foo._bar)
print(foo.getBar())

PI = 3.15
print(PI)
```
output:
```
Analyzing access.jsr...
File access.jsr [line 15]:
print(foo._bar)
          ^~~~
Accessing supposedly private attribute '_bar'
File access.jsr [line 18]:
PI = 3.15
^~
Assigning to supposedly constant name PI
```

## How to use

Pulsar doesn't have any dependency (aside from the obvious one of **J\***) so using it is as simple
as cloning the repository:
```bash
git clone https://github.com/bamless/pulsar.git
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