---
layout: page
title: Quick Reference for PowerShell & C# Terms
permalink: /glossary/
---

# Index of Terms

Below is a (probably wildly incomplete) list of common PowerShell and C# terms that you may find
useful.
If a specific term is not present, please feel free to [contact](/about/) me and ask me to amend
this page!

# Common Terminology

## Assembly

A compiled file, usually a `.dll`, that contains a library of (typically) related types.

[Reference]()

```powershell
Add-Type -AssemblyName System.Drawing.Common
```

[Back to Index](#index-of-terms)

## Calculated Property

Typically created with `Select-Object`; a property which is derived from another property or
expression.
These are always static _NoteProperties_.

[Reference]()

```powershell
$Item | Select-Object -Property @{ Name='Thing'; Expression={$_.Other + 4} }
```

[Back to Index](#index-of-terms)

## Cast

A much more permissive type conversion operation than C#'s style of casting.
Many methods of conversion will be attempted when casting in PowerShell; an error will only be
generated if _all_ of them fail.

[PowerShell](https://blogs.msdn.microsoft.com/powershell/2013/06/11/understanding-powershells-type-conversion-magic/) &mdash;
[C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/casting-and-type-conversions)

```powershell
[int]"12.4"
```

[Back to Index](#index-of-terms)

## Class

The definition of a specific reference type, set out in code. The _template_ from which an object of
that specific type is created.
Classes can _inherit_ (aka be created from a template of) other classes.

[C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/classes)

```powershell
class MyClass {
    [string] $Property = 10
}
```

[Back to Index](#index-of-terms)

## Enum

A massively simplified version of a class or struct, defining a set of values that bear their own
named labels.
Each label is mapped to a specific value in the underlying type.

[PowerShell](https://social.technet.microsoft.com/wiki/contents/articles/26436.how-to-create-and-use-enums-in-powershell.aspx) &mdash;
[C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/enumeration-types)

```powershell
enum Mood {
    Bad
    Good
    HouseIsOnFire
}
```

[Back to Index](#index-of-terms)

## Enumerate

To step through a collection of items, one at a time.

[PowerShell]() &mdash;
[C#](https://csharp.net-tutorials.com/control-structures/loops/)

```powershell
$List = 1, 2, 3, 4, 5
foreach ($Item in $List) {
    "I count $Item"
    Start-Sleep -Seconds 1
}
```

[Back to Index](#index-of-terms)

## Object

A single instance of a defined type.
Unlike most shells, PowerShell treats data as objects rather than strings.

[Reference]()

```powershell
$Object = New-Object -TypeName System.Numerics.Complex
$Object = [System.Numerics.Complex]::new()
```

[Back to Index](#index-of-terms)

## Reflection

Use of specialised methods (contained in the `System.Reflection` namespace) to access otherwise
inaccessible properties, values, and methods during runtime.

[PowerShell](https://blog.netspi.com/using-powershell-and-reflection-api-to-invoke-methods-from-net-assemblies/) &mdash;
[C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection)

```powershell
ExampleCode
```

[Back to Index](#index-of-terms)

## Struct

A complex value type, which may have many properties and contained values.
Structs cannot be natively created in PowerShell but are frequently borrowed from the .NET
libraries.

[C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/classes)

```powershell
[DateTime]::Now
```

[Back to Index](#index-of-terms)

## Type

The runtime representation of a class. A `Type` is actually a defined class of object in its own
right, storing metadata about the class and allowing limited direct interaction with the features of
the class itself.

[C#](https://docs.microsoft.com/en-us/dotnet/api/system.type?view=netcore-2.1)

```powershell
$Type = [System.Collections.Hashtable]
```

[Back to Index](#index-of-terms)

## Tokenizer

The core code in a language parser that translates portions of string data in the source code into
usable language tokens of defined types.

[PowerShell](https://geekeefy.wordpress.com/2017/06/07/powershell-tokenization-and-abstract-syntax-tree/)

```powershell
using namespace System.Management.Automation.Language

$Tokens = $null
[Parser]::ParseInput( '{ $v = 1 }', [ref] $Tokens, $null )
$Tokens
```

[Back to Index](#index-of-terms)

## Unroll

Informal term used to describe how many PowerShell language features will automatically enumerate
collections, such as the pipeline, or when accessing properties on a collection of items.

[PowerShell](http://www.nivot.org/blog/post/2012/03/16/PowerShell-30-Now-with-Property-Unrolling!)

```powershell
$FileList.Length
```

[Back to Index](#index-of-terms)

## ScriptProperty

A property whose value is not necessarily fixed and will be calculated every time it is requested.
Usually created using `Add-Member`.

[Reference]()

```powershell
$Object | Add-Member -Type ScriptProperty -Name 'Prop' -Value {$this.Val - 10}
```

[Back to Index](#index-of-terms)

## SubExpression

A general term for a complete expression embedded within another.
In PowerShell, this frequently refers to embedding a complete expression within a string using
`$( )`, which is often known as the `subexpression operator`.
PowerShell string subexpressions are only permitted in double-quoted strings.

[Reference]()

```powershell
"This is a $('string','value','piece of text' | Get-Random) with an embedded subexpression."
```

[Back to Index](#index-of-terms)

## Pipeline

## Command

An instruction to run in PowerShell. Includes functions, scripts, native applications, cmdlets, and aliases.

## Function

## Advanced Function

## Cmdlet

A command written in C# or another .NET language. Normally compiled into an assembly.

## Scope

## Variable

## Splat

## Generics

## Method

## Operators

## Output Streams

## Error

### Terminating Error

### Non-Terminating Error

## Exception

## Dot Sourcing

## Call Operator

## PS Remoting

## Double-Hop

## Profile

# PowerShell Primitives

## Primitive Type

## Integer

## String

## Hashtable

## Array

## Double
