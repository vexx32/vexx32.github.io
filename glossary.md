---
layout: page
title: Quick Reference for PowerShell & C# Terms
permalink: /glossary/
---

| Term       | Definition | Example | Reference Link |
|:----------:|:-----------|:--------|:--------------:|
| Assembly              | A compiled file, usually a `.dll`, that contains a library of (typically) related types. | `System.Drawing.Common.dll` | [Link]() |
| Calculated Property   | Typically created with `Select-Object`; a property which is derived from another property or expression. These are always static _NoteProperties_. | `$Item`<code> &#124; </code>`Select-Object -Property @{ Name='Thing'; Expression={$_.Other + 4} }` | [Link]() |
| Cast                  | A much more permissive type conversion operation than C#'s style of casting. Many methods of conversion will be attempted when casting in PowerShell; an error will only be generated if _all_ of them fail.| `[int]"12.4"` | [PowerShell](https://blogs.msdn.microsoft.com/powershell/2013/06/11/understanding-powershells-type-conversion-magic/) &#13; [C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/casting-and-type-conversions) |
| Class                 | The definition of a specific reference type, set out in code. The _template_ from which an object of that specific type is created. Classes can _inherit_ (aka be created from a template of) other classes. | `class MyClass { ... }` | [Link]() |(https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/classes) |
| Enum                  | A massively simplified version of a class or struct, defining a set of values that bear their own named labels. Each label is mapped to a specific value in the underlying type. | `enum Mood { Bad, Good, HouseIsOnFire }` | [PowerShell](https://social.technet.microsoft.com/wiki/contents/articles/26436.how-to-create-and-use-enums-in-powershell.aspx) &#13; [C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/enumeration-types) |
| Enumerate             | To step through a collection of items, one at a time. | `foreach ($Item in $List) { ... }` | [PowerShell]() &#13; [C#](https://csharp.net-tutorials.com/control-structures/loops/) |
| Object                | A single instance of a defined type. | `$Object = New-Object -TypeName System.Numerics.Complex` | [Link]() |
| Reflection            | Use of specialised methods (contained in the `System.Reflection` namespace) to access otherwise inaccessible properties, values, and methods during runtime. |`ExampleCode` | [PowerShell](https://blog.netspi.com/using-powershell-and-reflection-api-to-invoke-methods-from-net-assemblies/) &#13; [C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection) |
| Struct                | A complex value type, which may have many properties and contained values. | `[DateTime]::Now` | [Link]() |
| Type                  | The runtime representation of a class. A `Type` is actually a defined type of object in its own right, storing metadata about the class and allowing limited direct interaction with the features of the class itself. | `[System.Collections.Hashtable]` | [C#](https://docs.microsoft.com/en-us/dotnet/api/system.type?view=netcore-2.1) |
| Tokenizer             | The core code in a language parser that translates portions of string data in the source code into usable language tokens of defined types. | `using namespace System.Management.Automation.Language; $Tokens = $null; [Parser]::ParseInput('{ $v = 1 }',[ref]$Tokens,$null); $Tokens` | [PowerShell](https://geekeefy.wordpress.com/2017/06/07/powershell-tokenization-and-abstract-syntax-tree/) |
| Unroll                | Informal term used to describe how many PowerShell language features will automatically enumerate collections, such as the pipeline or when accessing properties on a collection of items. | `$FileList.Length` | [PowerShell](http://www.nivot.org/blog/post/2012/03/16/PowerShell-30-Now-with-Property-Unrolling!) |
| ScriptProperty        | A property whose value is not necessarily fixed and will be calculated every time it is requested. Usually created using `Add-Member`. | `$Object`<code> &#124; </code>`Add-Member -Type ScriptProperty -Name 'Prop' -Value {$this.Val - 10}` | [Link]() |
& | . | ,
SubExpression

Pipeline -
Command - An instruction to run in PowerShell. Includes functions, scripts, native applications, cmdlets, and aliases.
Function -
Advanced Function -
Cmdlet - A command written in C# or another .NET language. Normally compiled into an assembly.
Scope
Variable
Splat
Generics
Method
Operators
Output Streams
Error - Terminating Error -
Error - Non-terminating Error -
Exception -


PS Remoting
Double-Hop
Profile

## PowerShell Primitive Types (whats a primitive)

Int -
String -
Hashtable -
Array -
Double -
