---
layout: page
title: Glossary of PowerShell & Related Terms
permalink: /terms/
---

# Table of Contents

* [C# and .NET Terms](#c-and-net-terms)
* [PowerShell Terms](#powershell-terms)

# C# and .NET Terms

Reflection

Assembly - A collection of types

Class - The recipe which defines a type. The class describes how the members of type. For example, the class describes the properties, and methods of a type.
Object - An instance of a type. Everything in PS is an object (because I can't think of any cases where this is not so).
Type - Created from a class. Describes the shape and behaviour of the object.

# PowerShell Terms

Cast - In PowerShell, this is an attempt to convert an object from one type to another. For example, a string can be "cast" into an integer: [Int]'1'
enum - Introduced as a keyword in PowerShell 5.0. A set of constant values of a specific type used to define a simple set of possible values for a specific purpose with lexical representations.
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
