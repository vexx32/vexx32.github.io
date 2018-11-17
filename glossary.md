---
layout: page
title: Glossary of PowerShell & Related Terms
permalink: /glossary/
---

# Table of Contents

* [Terms From C# and .NET](#terms-from-C-and-NET)
* [PowerShell Terms](#powershell-terms)

# Terms From C# and .NET

| Term | Definition | Example | Reference Link |
|:----:|:-----------|:--------|:--------------:|
| Assembly   | A compiled file, usually a `.dll`, that contains a library of (typically) related types. | `System.Drawing.Common.dll` |
| Cast       | In C#, this is a tightly-defined operation which converts one type into another which can be overloaded by defining `op_Explicit()` in a class. | `var n = (uint)x;` |
| Class      | The definition of a specific type, set out in code. The _template_ from which an object of that specific type is created. Classes can _inherit_ (aka be created by automatically copying from) other classes. | `class MyClass { ... }` |
| Enum       | A massively simplified version of a class or struct, defining a set of values that bear their own named labels. Each label is mapped to a specific value in the underlying type. | `enum Mood { Bad, Good, HouseIsOnFire }` | [Link](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/enumeration-types) |
| Object     | A single instance of a defined type. | `$Object = New-Object -TypeName System.Numerics.Complex` |
| Reflection | Use of specialised methods (contained in the `System.Reflection` namespace) to access otherwise inaccessible properties, values, and methods during runtime. | |
| Type       | The runtime representation of a class. A `Type` is actually a defined type of object in its own right, storing metadata about the class and allowing limited direct interaction with the features of the class itself. | `[System.Collections.Hashtable]` |

# PowerShell Terms

| Term | Definition | Example | Reference Link |
|:----:|:-----------|:--------|:--------------:|
| Cast | In PowerShell, this operation is much more loosely-defined and cannot be easily overloaded. Many methods of conversion will be attempted when casting in PowerShell, and only after _all_ of them have failed will you receive an error.  | `[int]"12.4"` |
enumerate - To loop through each item in a collection
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
