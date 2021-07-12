---
permalink: /lint
title: Linting
toc: true
---

Luau comes with a set of linting passes, that help make sure that the code is correct and consistent. Unlike the type checker, that models the behavior of the code thoroughly and points toward type mismatches that are likely to result in runtime errors, the linter is more opinionated and produces warnings that can often be safely ignored, although it's recommended to keep the code clean of the warnings.

Linter produces many different types of warnings; many of these are enabled by default, and can be suppressed by declaring `--!nolint NAME` at the top of the file. In dire situations `--!nolint` at the top of the file can be used to completely disable all warnings (note that the type checker is still active, and requires a separate `--!nocheck` declaration).

The rest of this page documents all warnings produced by the linter; each warning has a name and a numeric code, the latter is used when displaying warnings.

## UnknownGlobal (1)

By default, variables in Luau are global (this is inherited from Lua 5.x and can't be changed because of backwards compatibility). This means that typos in identifiers are invisible to the parser, and often break at runtime. For this reason, the linter considers all globals that aren't part of the builtin global table and aren't explicitly defined in the script "unknown":

```lua
local displayName = "Roblox"

-- Unknown global 'displaName'
print(displaName)
```

Note that injecting globals via `setfenv` can produce this warning in correct code; global injection is incompatible with type checking and has performance implications so we recommend against it and in favor of using `require` with correctly scoped identifiers.

## DeprecatedGlobal (2)

Some global names exist for compatibility but their use is discouraged. This mostly affects globals introduced by Roblox, and since they can have problematic behavior or can break in the future, this warning highlights their uses:

```lua
-- Global 'ypcall' is deprecated, use 'pcall' instead
ypcall(function()
    print("hello")
end)
```

## GlobalUsedAsLocal (3)

The UnknownGlobal lint can catch typos in globals that are read, but can't catch them in globals that are assigned to. Because of this, and to discourage the use of globals in general, linter detects cases when a global is only used in one function and can be safely converted to a local variable. Note that in some cases this requires declaring the local variable in the beginning of the function instead of where it's being assigned to.

```lua
local function testFunc(a)
    if a < 5 then
        -- Global 'b' is only used in the enclosing function; consider changing it to local
        b = 1
    else
        b = 2
    end
    print(b)
end
```

## LocalShadow (4)

In Luau, it is valid to shadow locals and globals with a local variable, including doing it in the same function. This can result in subtle bugs, since the shadowing may not be obvious to the reader. This warning detects cases where local variables shadow other local variables in the same function, or global variables used in the script; for more cases of detected shadowing see `LocalShadowPedantic`.

```lua
local function foo()
    for i=1,10 do
        -- Variable 'i' shadows previous declaration at line 2
        for i=1,10 do
            print(i)
        end
    end
end
```

## SameLineStatement (5)

Luau doesn't require the use of semicolons and doesn't automatically insert them at line breaks. When used wisely this results in code that is easy to read and understand, however it can cause subtle issues and hard to understand code when abused by using many different statements on the same line. This warning highlights cases where code should either be broken into multiple lines, or use `;` as a visual guide.

```lua
-- A new statement is on the same line; add semi-colon on previous statement to silence
if b < 0 then local a = b + 1 print(a, b) end
```

## MultiLineStatement (6)

An opposite problem is having statements that span multiple lines. This is good for readability when the code is indented properly, but when it's not it results in code that's hard to understand, as its easy to confuse the next line for a separate statement.

```lua
-- Statement spans multiple lines; use indentation to silence
print(math.max(1,
math.min(2, 3)))
```

## LocalUnused (7)

This warning is one of the few warnings that highlight unused variables. Local variable declarations that aren't used may indicate a bug in the code (for example, there could be a typo in the code that uses the wrong variable) or redundant code that is no longer necessary (for example, calling a function to get its result and never using this result). This warning warns about locals that aren't used; if the locals are not used intentionally they can be prefixed with `_` to silence the warning:

```lua
local x = 1
-- Variable 'y' is never used; prefix with '_' to silence
local y = 2
print(x, x)
```

## FunctionUnused (8)

While unused local variables could be useful for debugging, unused functions usually contain dead code that was meant to be removed. Unused functions clutter code, can be a result of typos similar to local variables, and can mislead the reader into thinking that some functionality is supported.

```lua
-- Function 'bar' is never used; prefix with '_' to silence
local function bar()
end

local function foo()
end

return foo()
```

## ImportUnused (9)

In Luau, there's no first-class module system that's part of the syntax, but `require` function acts as an import statement. When a local is initialized with a `require` result, and the local is unused, this is classified as "unused import". Removing unused imports improves code quality because it makes it obvious what the dependencies of the code are:

```lua
-- Import 'Roact' is never used; prefix with '_' to silence
local Roact = require(game.Packages.Roact)
```

## BuiltinGlobalWrite (10)

While the sandboxing model of Luau prevents overwriting built-in globals such as `table` for the entire program, it's still valid to reassign these globals - this results in "global shadowing", where the script's global table contains a custom version of `table` after writing to it. This is problematic because it disables some optimizations, and can result in misleading code. When shadowing built-in globals, use locals instead.

```lua
-- Built-in global 'math' is overwritten here; consider using a local or changing the name
math = {}
```

## PlaceholderRead (11)

`_` variable name is commonly used as a placeholder to discard function results. The linter follows this convention and doesn't warn about the use of `_` in various cases where a different name would cause a warning otherwise. To make sure the placeholder is only used to write values to it, this warning detects the cases where it's read instead:

```lua
local _ = 5
-- Placeholder value '_' is read here; consider using a named variable
return _
```

## UnreachableCode (12)

In some cases the linter can detect code that is never executed, because all execution paths through the function exit the function or the loop before reaching it. Such code is misleading because it's not always obvious to the reader that it never runs, and as such it should be removed.

```lua
function cbrt(v)
    if v >= 0 then
        return v ^ 1/3
    else
        error('cbrt expects a non-negative argument')
    end
    -- Unreachable code (previous statement always returns)
    return 0
end
```

## UnknownType (13)

Luau provides several functions to get the value type as a string (`type`, `typeof`), and some Roblox APIs expose class names through string arguments (`Instance.new`). This warning detects incorrect use of the type names by checking the string literals used in type comparisons and function calls.

```lua
-- Unknown type 'String' (expected primitive type)
if type(v) == "String" then
    print("v is a string")
end
```

## ForRange (14)

When using a numeric for, it's possible to make various mistakes when specifying the for bounds. For example, to iterate through the table backwards, it's important to specify the negative step size. This warning detects several cases where the numeric for only runs for 0 or 1 iterations, or when the step doesn't divide the size of the range evenly.

```lua
-- For loop should iterate backwards; did you forget to specify -1 as step?
for i=#t,1 do
end
```

## UnbalancedAssignment (15)

Assignment statements and local variable declarations in Luau support multiple variables on the left side and multiple values on the right side. The number of values doesn't need to match; when the right side has more values, the extra values are discarded, and then the left side has more variables the extra variables are set to `nil`. However, this can result in subtle bugs where a value is omitted mistakenly. This warning warns about cases like this; if the last expression on the right hand side returns multiple values, the warning is not emitted.

```lua
-- Assigning 2 values to 3 variables initializes extra variables with nil; add 'nil' to value list to silence
local x, y, z = 1, 2
```

## ImplicitReturn (16)

In Luau, there's a subtle difference between returning no values from a function and returning `nil`. In many contexts these are equivalent, but when the results are passed to a variadic function (perhaps implicitly), the difference can be observed - for example, `print(foo())` prints nothing if `foo` returns no values, and `nil` if it returns `nil`.

To help write code that has consistent behavior, linter warns about cases when a function implicitly returns no values, if there are cases when it explicitly returns a result. For code like this it's recommended to use explicit `return` or `return nil` at the end of the function (these have different semantics, so the correct version to use depends on the function):

```lua
local function find(t, expected)
    for k,v in pairs(t) do
        if k == expected then
            return v
        end
    end
    -- Function 'find' can implicitly return no values even though there's an explicit return at line 4; add explicit return to silence
end
```

## LocalShadowPedantic (17)

This warning extends `LocalShadow` by also warning about cases where a local variable shadows a local variable with the same name from a parent function, or when it shadows a builtin global. This warning tends to be noisy and as such is disabled by default.

## FormatString (18)

Luau has several library functions that expect a format string that specifies the behavior for the function. These format strings follow a specific syntax that depends on the question; mistakes in these strings can lead to runtime errors or unexpected behavior of the code.

To help make sure that the strings used for these functions are correct, linter checks calls to `string.format`, `string.pack`, `string.packsize`, `string.unpack`, `string.match`, `string.gmatch`, `string.find`, `string.gsub` and `os.date` and issues warnings when the call uses a literal string with an incorrect format:

```lua
-- Invalid match pattern: invalid capture reference, must refer to a closed capture
local cap = string.match(s, "(%d)%2")

-- Invalid format string: unfinished format specifier
local str = ("%d %"):format(1, 2)
```

 Note that with the exception of `string.format` this only works when the function is called via the library, not via the method call (so prefer `string.match(s, "pattern")` to `s:match("pattern")`).

## TableLiteral (19)

Table literals are often used in Luau to create objects or specify data. The syntax for table literals is rich but invites mistakes, for example it allows duplicate keys or redundant index specification for values already present in the list form. This warning is emitted for cases where some entries in the table literal are ignored at runtime as they're duplicate:

```lua
print({
    first = 1,
    second = 2,
    first = 3, -- Table field 'first' is a duplicate; previously defined at line 2
})
```

## UninitializedLocal (20)

It's easy to forget to initialize a local variable and then proceed to use it regardless. This can happen due to a typo, or sometimes it can happen because the original variable definition is shadowed. When a local variable doesn't have an initial value and is used without ever being assigned to, this warning is emitted:

```lua
local foo

if foo then -- Variable 'foo' defined at line 1 is never initialized or assigned; initialize with 'nil' to silence
	print(foo)
end
```

## DuplicateFunction (21)

This warning is emitted when a script defines two functions with the same name in the same scope.

The warning is not produced when the functions are defined in different scopes because this is much more likely to be intentional.

```lua
function foo() end
function foo() end -- Duplicate function definition: 'foo' also defined on line 1

-- OK: the functions are not defined in the same scope.
if x then
    function bar() end
else
    function bar() end
end
```

## DeprecatedApi (22)

This warning is emitted when a script accesses a method or field that is marked as deprecated. Use of deprecated APIs is discouraged since they may have performance or correctness issues, may result in unexpected behavior, and could be removed in the future.

```lua
function getSize(i: Instance)
    return i.DataCost -- Member 'Instance.DataCost' is deprecated
end
```

## TableOperations (23)

To manipulate array-like tables, Luau provides a set of functions as part of the standard `table` library. To help use these functions correctly, this warning is emitted when one of these functions is used incorrectly or can be adjusted to be more performant.

```lua
local t = {}

table.insert(t, 0, 42) -- table.insert uses index 0 but arrays are 1-based; did you mean 1 instead?

table.insert(t, #t+1, 42) -- table.insert will append the value to the table; consider removing the second argument for efficiency
```
## DuplicateCondition (24)

When checking multiple conditions via `and/or` or `if/elseif`, a copy & paste error may result in checking the same condition redundantly. This almost always indicates a bug, so a warning is emitted when use of a duplicate condition is detected.

```lua
assert(self._adorns[normID1] and self._adorns[normID1]) -- Condition has already been checked on column 8
```
