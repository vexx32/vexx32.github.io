---
layout: post
title: Handling PowerShell's Versatile Subexpressions
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
Yes, it's perfectly legal (and quite useful on occasion!) syntax to drop an entire statement block into a subexpression.
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

# First four values -> Test-Connection Output (if using PS Core 6.2.x / 7)
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
    $folders = Get-ChildItem -Directory
    $folders | Get-ChildItem -File
}
```

This has a similar effect to using a subexpression.
However, using a scriptblock creates a new scope &mdash; any new variables or other scoped data will not be available outside the scriptblock.
In the above example, the `$folders` variable becomes unavailable as soon as you step outside the scriptblock once again.

The other option is to dot-source the scriptblock.

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
> Generally speaking, invoking the scriptblock should be the typical case, and dot-sourcing should be an exception to the rule.

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

& $Script # output: 24
```

That is, no matter where you call the scriptblock from, it behaves the same way.
It's hard to see how this is useful here, but it comes in _very_ handy if you ever need to handle events in PowerShell.

Events in C# are pretty easy to work with.
In PowerShell, they can be problematic, since there's no guarantee an event will be invoked in the same context that it was registered in.
It can very easily end up being called from a completely separate thread.
This means that the scriptblock has to be effectively self-contained &mdash; if you need to reference anything that's defined or created outside the scriptblock itself, `GetNewClosure()` comes in very handy.

```powershell
function Get-Collection {
    [CmdletBinding()]
    param() end {
        $collection = [System.Management.Automation.PSDataCollection[int]]::new()
        $sb = {
            if ($null -eq $collection) {
                [Console]::WriteLine("COLLECTION IS NULL OH NO")
                return
            }

            [Console]::WriteLine("Collection length is now $($collection.Count + 1).")
        }.GetNewClosure()

        $collection.add_DataAdding($sb)
        $PSCmdlet.WriteObject($collection, $false)
    }
}

$col = Get-Collection
$col.Add(10)
```

Credit to [Patrick Meinecke](https://twitter.com/SeeminglyScienc) for the lovely example here.

Essentially what's going on here is we're creating a `PSDataCollection`, and then registering a scriptblock to its `DataAdding` event.
This event triggers whenever you add items to the collection.
Here, `GetNewClosure()` allows the scriptblock to reference its own parent collection directly, without fear that the `$collection` variable won't exist when the event fires.

Calling `.Add()` causes the event to fire as the new data is added, and the reference to `$collection` is maintained.
You won't always need this capability, but it's a very handy tool to keep on hand for when you do need it!

## Putting it All Together

I've often found that this kind of thing is best illustrated by example, so I'll try to make this part a bit more on the side of showing than telling.

### Examples

#### Generating a List of Arguments for a Native Command

```powershell
$command = 'docker'
$args = @(
    'rm'
    if ($Force) {
        '-f'
    }
    $Name
)

& $command @args | Out-Null
```

In this example, we're invoking an `rm` command with Docker, and we want to wrap it in a PowerShell function.
We can expose a `-Force` parameter that it respects, and altering the arguments is as simple as adding the `-f` argument or not.
Rather than complicate things by trying to add it in later, or duplicating the code completely and having two paths (one with `-f` and one without), we can just add an `if` statement directly in the subexpression.

Personally, I prefer this kind of methodology over manually manipulating strings any day, especially since that usually leads to `Invoke-Expression`.

#### Creating Match Patterns

```powershell
$DomainPattern = @(
    'domain.com'
    'domain.co.uk'
    'domain.de'
).ForEach{ [regex]::Escape($_) } -join '|'

Get-ExoMailbox -ResultSize Unlimited | Where-Object UserPrincipalName -match $DomainPattern
```

Here, we put together an array of domain names, call `[regex]::Escape()` on each of them so we can use them as literals in a regex expression, and then join on `|` which is a logical OR separator in regex.
The end result is that we can filter by the regular expression and get any of the objects where the `UserPrincipalName` contains any of the designated domain names.
It's a great example of when you can use the multiline syntax for an array subexpression.
Using a multi-line statement is perfectly optional here; the above `$DomainPattern` declaration is identical to the following:

```powershell
$DomainPattern = @('domain.com', 'domain.co.uk', 'domain.de').ForEach{ [regex]::Escape($_) } -join '|'
```

There are a couple of reasons I tend to prefer the multiline syntax:

1. Shorter line lengths tend to be a bit easier for me to read; I prefer to scan primarily down a script rather than both down _and_ across.
1. Breaking the statement over a couple of lines actually makes collaborative coding easier.
   In a line-by-line diff (the sort you'll see with Git and also with pull requests on Github and similar platforms), the multiline version is _much_ easier on the eyes when you're adding or removing items from the array.

#### Collating Output from Multiple Sources

```powershell
$FileList = @(
    Get-ChildItem -Path $Dir1 -File
    Get-ChildItem -Path $Dir2 -File -Recurse
)

foreach ($File in $FileList) {
    Rename-Item -Path $File.FullName -NewName "$($File.Name).old"
}
```

Here we're able to collate the output from _multiple_ discrete commands and put them into a single array.
This example is a little contrived, since you can do a similar thing in _most_ cases just by supplying multiple items to `-Path`.
However, if you ever have a case where you need to search through each path a little differently (using different filters, for example), this can come in handy.

I also quite like that you can effectively **join** two separate pipelines together and continue them in a third pipeline!

```powershell
$(
    Get-ChildItem -Path $Dir1 -File | Where-Object Name -notmatch '\d'
    Get-ChildItem -Path $Dir2 -Recurse -File | Where-Object Length -gt 4kb
) |
    Sort-Object -Property Length |
    Select-Object -Property Name, Length, FullName |
    Export-Csv -Path $CsvFile
```

The syntax can at times be a bit awkward, so feel free to play with it a little until it looks somewhat acceptable to you.
I haven't needed this more than once or twice, but I rather like the idea that I can sort of funnel many things into a single pipeline if I need to.

### Familiarity

I think a lot of users will likely be confused at first if they see a subexpression pushed to its limits.
You can go quite a ways into the weeds with them if you really have a need to.

```powershell
$List = 1..20

$Items = @(
    $Assertions = switch ($List) {
        { $_ % 3 -eq 0 } {
            "$_ is divisible by 3"
        }
        { $_ / 2 -gt 8 } {
            "$_ is more than 16"
        }
        7 {
            "Lucky day!"
        }
        default {
            "Today's not very lucky for you..."
        }
    }

    "Your Lucky Numbers are..."
    $(
        1..50 | Get-Random -Count (Get-Random -Min 1 -Max 10)
    ) -join ', '

    $Assertions | Get-Random

    # Push any errors back to the regular output stream here
    & { Resolve-DnsName google.com } 2>&1
)

$Items -join ' | '
```

Would you ever _need_ to?
Probably not.
Clearly this is quite a contrived example, but it serves to show just how much you can throw into a single subexpression.
Yes, if you wanted to, you could use `$()` the same way `@()` is used here, if you needed to.

Sometimes you'll find the alternative is significantly longer, more convoluted, and quite a bit slower.
There are advantages to be had in being able to contain so many different constructs inside a single statement, from time to time.

I would definitely be wary of trying to encapsulate _too_ many complicated statements into a single subexpression, to be sure.
Knowing it's possible just ensures you have the option open to you, should you ever need it.
