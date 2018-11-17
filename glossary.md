---
layout: page
title: Glossary of PowerShell & Related Terms
permalink: /glossary/
---

| Term       | Definition | Example | Reference Link |
|:----------:|:-----------|:--------|:--------------:|
| Assembly   | A compiled file, usually a `.dll`, that contains a library of (typically) related types. | `System.Drawing.Common.dll` | [Link]() |
| Cast       | A much more permissive type conversion operation than C#'s style of casting. Many methods of conversion will be attempted when casting in PowerShell; an error will only be generated if _all_ of them fail.| [int]"12.4"` | [PowerShell](https://blogs.msdn.microsoft.com/powershell/2013/06/11/understanding-powershells-type-conversion-magic/) [C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/casting-and-type-conversions) |
| Class      | The definition of a specific reference type, set out in code. The _template_ from which an object of that specific type is created. Classes can _inherit_ (aka be created from a template of) other classes. | `class MyClass { ... }` | [Link]() |(https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/classes) |
| Enum       | A massively simplified version of a class or struct, defining a set of values that bear their own named labels. Each label is mapped to a specific value in the underlying type. | `enum Mood { Bad, Good, HouseIsOnFire }` | [PowerShell](https://social.technet.microsoft.com/wiki/contents/articles/26436.how-to-create-and-use-enums-in-powershell.aspx) [C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/enumeration-types) |
| Object     | A single instance of a defined type. | `$Object = New-Object -TypeName System.Numerics.Complex` | [Link]() |
| Reflection | Use of specialised methods (contained in the `System.Reflection` namespace) to access otherwise inaccessible properties, values, and methods during runtime. |`ExampleCode` | [Link()] |
| Struct     | A complex value type, which may have many properties and contained values. | `[DateTime]::Now` | [Link]() |
| Type       | The runtime representation of a class. A `Type` is actually a defined type of object in its own right, storing metadata about the class and allowing limited direct interaction with the features of the class itself. | `[System.Collections.Hashtable]` | [C#](https://docs.microsoft.com/en-us/dotnet/api/system.type?view=netcore-2.1)
| Enumerate  | To step through a collection of items, one at a time. | `foreach ($Item in $List) { ... }` | [Link]() |
Tokenizer - include brief example / concept
Unroll - Automatically tranform a collection into its composite items as individual objects. Typically used to refer to member unrolling. For example the AddressFamily property of two IPAddress objects: ([IPAddress[]]('1.2.3.4', '::1')).AddressFamily
lexicon
Calculated Property - Typically used with Select-Object, a property which is derived from another value or expression.
ScriptProperty - A value which is calculated when the property is evaluated. ScriptProperties
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
