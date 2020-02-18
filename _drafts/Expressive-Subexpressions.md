---
layout: post
title: Expressive Subexpressions
date: 2020-02-24
categories: [powershell, code style, good practices]
tags: [subexpression, functional, style]
---

I've always been a fan of using subexpressions to tidy up my code.
They tend to help me collect and store output where I need it, without requiring that I add complex loops and delve into .NET collection types.
The pattern seems to be a bit of a rarity among a lot of other scripters I see, so I thought I'd write a post on subexpressions and all the ways you can use them!

I'll also be covering use of scriptblocks as anonymous functions &mdash; invoking them immediately as you define them &mdash; since that's rather similar to how subexpressions work.

## Subexpressions and Anonymous Functions

Subexpressions are kind of like an inline function that always gets invoked immediately.
You define a set of commands that all get invoked one after another, and then PowerShell processes all the output and either stores or outputs the results.
The only thing you **can't** do with them is store the commands inside and execute them later.
A subexpression is always evaluated at the point in the script where it's declared.

True anonymous functions in PowerShell are also available in the form of inline scriptblock invocations.
If you choose, you _can_ actually store the scriptblock itself &mdash; the function definition, essentially &mdash; and invoke it multiple times.

Something to note that's shared between all subexpressions (as well as scriptblocks) is that the end of a line does **not** terminate the subexpression.
Only the matching closing parenthesis closes off the subexpression.
This actually includes explicit line endings with `;` as well, and this is why `( )` is not a true subexpression:

```powershell
# This line will throw a parse error; the expression ends before the closing parenthesis
$Values = ( Get-Item -Path $Path; Get-ChildItem -Path $Path )

# Conversely, this line is perfectly fine; the subexpression allows the use of line endings within it
$Values = $( Get-Item -Path $Path; Get-ChildItem -Path $Path )
```

Let's take a closer look at the types of subexpression PowerShell has available.

### Types of Subexpressions

Without lumping in anonymous functions as a kind of subexpression (which they _sort of_ are, but we'll look at that in a minute), there are two types of subexpression in PowerShell:

| Syntax | Name                | Example                                                |
| :----: | :------------------ | :----------------------------------------------------- |
| `$( )` | Subexpression       | `$String = "This script has $(Get-Random) functions."` |
| `@( )` | Array Subexpression | `$Values = @(1, 2, 3, 4)`                              |

#### Regular Subexpressions

I think most folks will typically see subexpressions used to insert data into a string, just like in the above example.
This is a great use for it, and it's a function exclusive to this type of subexpression.

But it's far from _all_ you can do with it.
Notice I mentioned a moment ago that you can have line breaks in subexpressions.
That opens up a few interesting possibilities here, especially because each line is considered a totally separate statement:

```powershell
$SomeValue = 4
$Values = $(
    Test-Connection 1.1.1.1

    if ($SomeValue -lt 10) {
        10
    }
    else {
        15
    }

    1..20
)
```

What are you looking at me like that for?
Yes, it's perfectly legal (and quite useful!) syntax to drop an entire statement block into a subexpression.
Ifs, switches, loops, you name it.
Pretty much any language statement is a legal entry in a subexpression, and though I'd advise against it you _can_ do odd things like embedding an entire if statement into a string.
More to the point, though, what kind of result would you expect to find in `$Values` when all's said and done?

It might not be immediately obvious if you're not accustomed to seeing this, but you'll get a single flat array stored in `$Values` when the subexpression completes.
Each statement within the subexpression executes in order and submits its output to the pipeline.

1. For `Test-Connection` (at least in PowerShell 6.2+) this means that 4 `PingStatus` objects are output and subsequently stored into `$Values`.
1. The `if`/`else` statement is evaluated, and the value `10` is output and subsequently stored.
1. The expression `1..20` is evaluated, and an array with values `1` through `20` are output.
   Note that this output is still over a pipeline, so the array is ultimately split up again and sent to output one item at a time.
   This will become important later.

So, our end result, if we'd written it all out manually, would look a bit like this:

```powershell
# Final array size: 25
$Values = [object[]]::new(25)

# First four values -> Test-Connection Output (PS Core / 7)
$Values[0..3] = Test-Connection 1.1.1.1

# Fifth value -> result of the if statement
$Values[4] = if ($SomeValue -lt 10) { 10 } else { 15 }

# Remaining values -> contents of array
$Values[5..24] = 1..20
```

Notice that it doesn't matter here whether we drop a scalar value or an array value into the subexpression.
As long as we leave a line-break between our expressions, each array is unwrapped as it goes over the pipeline, and then all the results are tossed into a single array at the end.
This is part of the essentials of pipeline behaviour, but I'm sure quite a few folks won't expect to see any pipeline behaviour arise from the above example.

> 💡 **Remember**
>
> Every statement in PowerShell that doesn't itself store or redirect output elsewhere uses the standard output pipeline.
> This includes statements within subexpressions.

And yes, before you ask... all of this _still_ applies when you have a subexpression inside a string.
The only additional thing to note in that case is that you're always going to end up converting to a string value at the end.
This can occasionally cause some unintended behaviour, since the behaviour when converting an array to a string depends on the value of `$OFS`.
Usually `$OFS` is unset, and a space is used to separate the items in the array.
if you set `$OFS` to, say, a comma, this behaviour changes.

```powershell
PS> "The values are $(1..10)."
The values are 1 2 3 4 5 6 7 8 9 10

PS> $OFS = ','
PS> "The values are $(1..10)."
The values are 1,2,3,4,5,6,7,8,9,10.
```

Just one little detail to keep in mind. 🙂

#### Array Subexpressions

Array subexpressions work **almost** exactly the same was as regular subexpressions.
They have two key differences:

1. You cannot directly insert an array subexpression into a string.
1. The result of an array subexpression is always (as the name implies) an array.
   (Specifically, it will always be an `[object[]]` array.)

> 📝 **Note**
>
> Array subexpressions will only wrap the output in an array if the output is not already an array.
> In other words, it guarantees the output will ultimately be an `[object[]]` array.
> If the result is a single item, it will be boxed into an `[object[]]` array, and if it's an array of another type, it will box each item and hand back an `object[]` array.
> In most cases, PowerShell will already have boxed the output as `[object[]]` due to its pipeline processing.

In every other way, they work identically to standard subexpressions.
You'll most often see these used as an "explicit" variant of defining an array.

```powershell
$array = @( 1, 2, 3, 4, 5 )
```

While this can be used for clarity, it is in effect somewhat redundant.
The `,` comma operator is what's actually creating the array in this instance.
The subexpression operator simply ensures the result is in fact an array.

### Inline Scriptblock Invocations

Scriptblocks can be defined and used directly in a script, much like a subexpression.
Unlike subexpressions, however, scriptblocks are usually used to store code &mdash; they won't be invoked unless you use one of the invocation operators.
A scriptblock, as the name implies, is just a block of script.
Any valid PowerShell code is valid here, with only a scant few exceptions.

```powershell
# `&` or `.` must be used to invoke the scriptblock
$Results = & {
    Get-ChildItem
    10..100
}
```

This has a similar effect to using a subexpression.
However, using a scriptblock creates a new scope &mdash; any new variables or other scoped data will not be available outside the scriptblock.

That is, unless you dot-source the scriptblock.

```powershell
$Results = . {
    $folders = Get-ChildItem -Directory
    $folders | Get-ChildItem -File
}
```

In this instance, both the `$Results` variable and the `$folders` variable would become available in the same scope after the scriptblock finishes executing.

> ℹ **Note**
>
> Dot-sourcing a scriptblock is more expensive in terms of performance than simply invoking it.
> Generally speaking, invoking the scriptblock should be the typical case, and dot-sourcing generally should be an exception to the rule.

#### `{}.GetNewClosure()`

Something to note about scriptblocks is that you can store them for later invocation, like an impromptu function.
Those of you coming from a C# background are probably already familiar with lambda expressions &mdash; this is something similar.

```powershell
$Value = 12
$Script = {
    $Value + 12
}

$Value = & $Script
$Value # output: 24

& $Script # output: 36
```

As you can see, although the scriptblock is indeed a stored _action_, its effect still depends on the code around it.
In some cases, you won't want this context-sensitive behaviour as much.
For these cases, you can call `.GetNewClosure()` on the scriptblock to effectively tie it to the context where it was defined.

If we apply that to the above example, you can see it in action:

```powershell
$Value = 12
$Script = {
    $Value + 12
}.GetNewClosure()

$Value = & $Script
$Value # output: 24

& $Script # output: 36
```

## Expressive Coding with Subexpressions and Scriptblock Invocations

some nice examples of code that combine them

maybe trivial / contrived examples for funsies

```powershell

$Values = @(
    Command-Name
    $value
    $array
    'raw data'
    @{ structured = data }

    $(
        Other-SubExpressions
    )
)

```
