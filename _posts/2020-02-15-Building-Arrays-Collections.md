---
layout: post
title: Building Arrays and Collections in PowerShell
date: 2020-02-15
categories: [powershell, style, good practices]
tags: [array, collection, list, powershell, best practices, good practices]
---

Well, it's certainly been a while since I posted on here.
Trying to get back into the swing of things for 2020 now that I'm mostly settled in at my new job.
So, let's take a look at a classic problem that never fails to confuse folks just getting into PowerShell.
I've also seen plenty of more experienced folks trip over it quite a lot, so here we are.

## Contents

- [Contents](#contents)
- [Background Information](#background-information)
  - [Background - Collections in .NET](#background---collections-in-net)
  - [ICollection](#icollection)
- [Collections in PowerShell](#collections-in-powershell)
  - [Common Patterns](#common-patterns)
    - [Use of `+=` to Build Arrays](#use-of--to-build-arrays)
      - [What's Wrong with this Picture](#whats-wrong-with-this-picture)
    - [Using an Expandable List](#using-an-expandable-list)
      - [Why Use a List](#why-use-a-list)
    - [Using the Pipeline](#using-the-pipeline)
  - [Working With Collections](#working-with-collections)
    - [`New-Object` v.s. `[item]::new()` v.s. `[item]@()`](#new-object-vs-itemnew-vs-item)
    - [Adding Many Items to an Existing Collection](#adding-many-items-to-an-existing-collection)
- [General Recommendations](#general-recommendations)

## Background Information

The term `collection` is often bandied about, so let's define it concretely before we get started, shall we?
In general parlance, a _collection_ is generally considered to be a pretty broad term.
In its simplest usage, it just refers to a set of individual items.
Oftentimes, these items may be related to one another in some way.

### Background - Collections in .NET

In the .NET ecosystem, the term `collection` has a pretty well-defined meaning.
A "collection" is any type of object that implements the `ICollection` interface.
Seems a bit circular, right?
Let's look at what that actually means and take a quick refresher on what an `interface` actually is in .NET for those of you who're not familiar.
From the C# Programming Guide's [page on interfaces][csharp-interfaces], we get this definition:

> An interface contains definitions for a group of related functionalities that a non-abstract class or a struct must implement.
>
> By using interfaces, you can, for example, include behavior from multiple sources in a class.

That's a bit of a wordy definition, but it does the job.
In practical terms, an interface is a way of defining a required set of properties or methods that must be defined by any other type that wants to inherit from or _implement_ this interface.
It's a way of being able to define standard ways of interacting with more than one kind of object.

### ICollection

[`ICollection`][icollection] is a fairly simple interface that has a few demands:

- Methods
  - CopyTo(Array array, int index)
- Properties
  - Count
  - IsSynchronized
  - SyncRoot

Not a whole lot, huh?
So for a type to be an `ICollection`, it needs to guarantee these things:

1. It has a `CopyTo()` method that you can use to copy its items into an array of your choosing starting from the requested index.
1. It has a `Count` property that will give you the current number of items in that collection.
1. It has the `IsSynchronized` and `SyncRoot` properties.
   For collections that utilise this functionality, these can be used to determine whether a collection's state is synchronised across threads.
   One example of a collection that allows you to create synchronised versions, you can check out [Hashtable][hashtable] and its static method [Synchronized()][hashtable-sync].

Fairly straightforward on the whole, I suppose.
Interfaces can get quite complex, and there are two main extensions of `ICollection` in use in .NET Core: [`IList`][ilist] and [`IDictionary`][idictionary]
`IList` types represent a "flat" collection, where it's simply an ordered list of items.
On the other side of things, `IDictionary` represents a less-strictly-ordered set of key/value pairs where a given key (often a string such as a name) is used to access the relevant item directly.

## Collections in PowerShell

The main type of collection in use in PowerShell is the humble `Array`, at least in terms of what is often available directly from a script.
Behind the murky curtain, PowerShell actually utilises the `System.Collections.ArrayList` type quite heavily in its pipeline processor.

Partially as a result of that, arrays (and more specifically `object[]`) are the backbone of PowerShell scripting.
In the vast majority of cases, you're going to be handling arrays unless you deliberately choose to use another type of collection to handle output.
Let's take a look at some fairly common methods of handling collections of items in PowerShell, shall we?

### Common Patterns

#### Use of `+=` to Build Arrays

```powershell
$array = @()
foreach ($value in 1..5) {
    $array += $value
}
```

Seems pretty simple and sound, at least at first glance.
So... spot anything amiss?

The pattern here is "create the collection object, and then add each item to it one at a time".
This works quite well for some collection types, but not so well for others.
In fact, for arrays specifically it's technically not _possible_ to do this &mdash; arrays are actually fixed-size collections.
If you call `$array.Add(10)` you'll get an error stating as much.
But this still works, somehow, and it's all thanks to some ✨ PowerShell magic! ✨

> ℹ **Behind the Scenes**
>
> PowerShell's `+` and `+=` operators are designed to work with arrays in a relatively unusual way.
> When you try to add items to an array like this, what _actually_ happens goes something like this:
>
> 1. PowerShell checks the size of the collection in `$array` and the number of items being added to it (in this case, just one each time).
> 1. PowerShell creates a _completely different_ array of the correct size, and
> 1. The original array is copied into this new array, along with the new item(s).
>
> This is also why it's perfectly possible to join two arrays together with the `+` or `+=` operators.

##### What's Wrong with this Picture

The missing piece here, and "what's wrong" in some ways is that this operation can become prohibitively expensive.
For smaller arrays, it's not a concern at all.
The main problem is that we won't always know ahead of time how big our collection might become.

If `$array` is quite large (say somewhere in the realm of 5,000-10,000 items), then generating new arrays every single time you add just a couple of items becomes **very** expensive to do.
A new array, even if it's empty, requires allocation of memory so that the array has enough space to exist in a single region of memory.
Adding a single item to a 10,000 item array means reserving a new block of memory, enough to hold _another_ 10,001 items, then copying everything across.

Notice that I didn't mention anything about deleting the original array.
This is because PowerShell can't necessarily assume you're _definitely_ wanting to completely remove the original array.
You might have another variable storing a reference to that same array in a completely different scope, after all.
Checking for that every time you want to add things to an array would **also** be prohibitively expensive.

So what happens, then?
The original array will sit around in memory until the .NET garbage collection routines realize that it's not needed by anything anymore.
In some cases, that can mean it sits around for quite a while before that memory is freed up again.

For smaller collections, of course, this isn't really an issue at all.
However, this is a very easy pattern to fall into, so it can come back to bite you when you least expect it.
Let's look at a few other ways to approach this task.

#### Using an Expandable List

.NET contains quite a few types of lists.
If you look at the types that implement `IList` you'll see there are quite a few available options.
Oftentimes older blog posts or code examples will have [`ArrayList`][arraylist] as their collection of choice.
Personally, I've never liked using ArrayList directly.
As I mentioned above, PowerShell uses it behind the scenes extensively &mdash; but I'd be willing to bet that if they had the time available, more than a few people on the PowerShell team would _love_ to replace it with something more modern.
I know I would; ArrayList comes from the very old days of .NET, hailing all the way back to .NET v1 if I recall correctly.

In fact, the documentation page for ArrayList warns against its continued use:

> ℹ **Important**
>
> We don't recommend that you use the ArrayList class for new development.
> Instead, we recommend that you use the generic `List<T>` class.
> The ArrayList class is designed to hold heterogeneous collections of objects.
> However, it does not always offer the best performance.
> Instead, we recommend the following:
>
> - For a heterogeneous collection of objects, use the `List<Object>` (in C#) or `List(Of Object)` (in Visual Basic) type.
> - For a homogeneous collection of objects, use the `List<T>` class.
>   See Performance Considerations in the `List<T>` reference topic for a discussion of the relative performance of these classes.
>   See Non-generic collections shouldn't be used on GitHub for general information on the use of generic instead of non-generic collection types.

So, for completeness' sake, I'll include an example of what using an ArrayList looks like &mdash; but I wouldn't recommend ever using it.

```powershell
# There are multiple ways to create the original ArrayList, but this is one of the simplest and easiest to use.
[System.Collections.ArrayList]$list = @()
foreach ($value in 1..5) {
    # Redirection to null is necessary as ArrayList outputs an index number for each added item.
    $list.Add($value) > $null
}
```

So, let's say we follow that recommendation from the ArrayList documentation, and want to use a generic List instead.
What even is [`List<T>`][list-1] anyway?
`List<T>` is a **generic** type, which in simple terms just means it has a bit of a fluid definition.
You can define a `List<T>` in terms of what type of object you want it to work with or (in the case of List, at least) store, where the `T` is the type of object you're storing in it.
For a list comprised entirely of numbers, we might use `List<int>` to store them.

Why would we use this over ArrayList?
In PowerShell there isn't a massive difference here, but for me personally it comes down to a few things.

1. You can create a List that only stores a specific type, which means you can catch mistakes earlier.
   Adding other types of items to that list will either have them converted to the List's designated type, or will throw an error stating that it was the wrong type.
1. Using the `Add()` method on `List<T>` doesn't create incidental output like `ArrayList`'s `Add()` method does, so you have less chance to miss things and pollute your output stream.
1. Given the assertion in the _documentation_, no less, that ArrayList is recommended against, I would assume two things:
   - There is the possibility at any point that the .NET / .NET Core team will eventually deprecate it completely and possibly even remove it from .NET at that point or afterwards.
   - No further work is being done on `ArrayList`, and if there are any improvements to be had you will likely only find them in `List<T>`.

Given that, I'd choose to use `List<T>` instead of ArrayList in every single case.
When using it, there's one question you need to ask before using it:

> What kind of objects am I storing in this list?

If you know ahead of time that you plan on storing more than one type of item in the list, you have two choices:

1. Use a common parent type or interface, for example `System.Exception` is the parent type of any Exception-typed object in .NET, so using that base type allows you to store _any_ other Exception object.
1. Use the `object` type, which is **the** parent type for everything in .NET.

In our previous examples, we're simply storing numbers.
Let's see how that looks with a generic List, shall we?

```powershell
[System.Collections.Generic.List[int]]$list = @()
foreach ($value in 1..5) {
    # No redirection needed here, List<T> does not emit data from Add()
    $list.Add($value)
}
```

That certainly looks a little tidier.
The long type name is a bit much at times, but at least in PowerShell 5.1 and up, you can use `using namespace System.Collections.Generic` to let you just use `[List[int]]` if you prefer.

> 📝 **Note**
>
> While the .NET documentation frequently uses the C# syntax `List<T>`, when we get to PowerShell we actually use square brackets to indicate the generic type parameter instead.
> `List<T>` instead becomes `[List[T]]`, and in practical usage `T` is replaced with another type name.
> This is purely a syntactical difference between C# and PowerShell &mdash; in both cases we're referring to the same .NET type.
> In F# you'd write `List<'T>` and in Visual Basic it looks instead like `List(Of T)`.

##### Why Use a List

Lists are fantastic for cases where you don't know the collection size ahead of time.
If you knew the size ahead of time, you can simply pre-allocate an array of the correct type and size anyway:

```powershell
$array = [int[]]::new(5)
for ($index = 0; $index -lt 5; $index++) {
    $array[$index] = $index * 2
}
```

In PowerShell, they're best used in cases where you need to be adding to the collection from multiple places in your code, **and** you want the order of items to stay the same as the order you put them into the collection.
They're also the most effective way to build multiple collections in a single loop statement, if that's something you need to do.

#### Using the Pipeline

I mentioned earlier that PowerShell already heavily uses `ArrayList` in its pipeline processor.
We can actually take advantage of this without ever needing to use ArrayList ourselves directly.
This style of building collections looks a lot more like coding in a functional language than the other options, so it may confuse some folks who aren't accustomed to coding this way.
However, it's perfectly valid and quite flexible in nature.

The above examples coded this way would look a little more like this:

```powershell
$array = foreach ($value in 1..5) {
    # This value is not stored anywhere and becomes part of the "output" from the loop statement.
    $value + 4
}
```

If you're not familiar with this kind of pattern, you're probably wondering how on Earth that even works.
In PowerShell, unlike most .NET languages (and indeed many other programming languages in general), allows you to assign a variable to the "result" of a statement, regardless of whether that statement is a keyword-based loop or a complete pipeline.
Any uncaptured output will then be funneled into that variable and stored as `[object[]]` (an array of objects) in the variable.

Because this utilises the built-in PowerShell pipeline (which, remember, uses ArrayList to handle output), there is no additional overhead of creating another type of collection here.
We're just using what PowerShell has always been ready to provide for us.
This is also one of the simplest ways to keep your code highly compatible across PowerShell versions with minimal effort.

> ℹ **Compatibility**
>
> This pattern works with _all_ released versions of PowerShell, even back to version **1**.
> For compatibility, it's the absolute best option.
> And thankfully, it's one of the most effective options as well!

This pattern works with all PowerShell control-flow statements, which include `switch`, `foreach`, `for`, `do..until`, `do..while`, `while`, and `try..catch..finally` (and there's probably a couple I forgot to mention, but will still work as well!)
It also works seamlessly with pipelines, no matter how lengthy they get:

```powershell
$pictures = Get-ChildItem -Filter *.png -Recurse -File |
    Where-Object Length -gt 1MB |
    Select-Object -Property Name, Length, FullName
```

### Working With Collections

#### `New-Object` v.s. `[item]::new()` v.s. `[item]@()`

This is more of a general point to make about creating new objects in PowerShell, but if you're not aware it bears mention here too.
`New-Object` is, not to put too fine a point on it, quite slow.
At least compared to alternatives.

Syntax like `$list = [System.Collections.Generic.List[int]]::new()` is fairly new, first available in PowerShell 5.1.
If compatibility is your main concern, `::new()` is not the first choice, to be sure.
However, `New-Object` should _still_ be the last choice, generally being reserved for COM objects and similar otherwise difficult to construct items.

Syntax `$list = [System.Collections.Generic.List[int]]@()` or `[System.Collections.Generic.List[int]] $list = @()` will work all the way back to version 2 of PowerShell.
For a compromise between compatibility and speed I'd go for one of those.

#### Adding Many Items to an Existing Collection

If you have a List collection and want to add multiple items to it quickly, rather than using any kind of loop you can actually add them in directly.
`List<T>` has an `AddRange()` method that allows you to add multiple items at once:

```powershell
$List = [System.Collections.Generic.List[object]]@(1..5)
$List.AddRange(5..10)

# Note - for strongly-typed lists you will need to cast the array you're adding to the relevant array type explicitly:

$List = [System.Collections.Generic.List[int]]@(1..5)
$List.AddRange([int[]](5..10))
```

## General Recommendations

Taking all the above into account, we can establish some broad recommendations when handling collections of items in PowerShell.

1. If you _can_ simply assign the statement to a variable to build your collection, **do so**.
   It's the most compatible approach and one of the simplest ways to handle it.
1. If for some reason you cannot assign the statement or need to build multiple collections in the same loop, use a `List<T>`.
1. If you have a scenario where you need to add multiple items to a collection, use a `List<T>` and make short work of it with `AddRange()`.

I went into this post thinking it'd be short and sweet, but I really can't help but explain things to the *n*th degree, can I? 😂

That's all I have for now, I think.
If you have any questions or comments, feel free to leave a comment below.

I'm also keeping an eye out for some more content to cover to get back into regular blog posts, so if there's something I haven't already covered that you're needing some explanation on and think I can help, give me a shout on Twitter or one of the community channels.
Have a great day! 😊

<!-- Reference Links -->
[csharp-interfaces]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/interfaces/
[icollection]: https://docs.microsoft.com/en-us/dotnet/api/system.collections.icollection?view=netcore-3.1
[hashtable]: https://docs.microsoft.com/en-us/dotnet/api/system.collections.hashtable?view=netcore-3.1
[hashtable-sync]: https://docs.microsoft.com/en-us/dotnet/api/system.collections.hashtable.synchronized?view=netcore-3.1
[ilist]: https://docs.microsoft.com/en-us/dotnet/api/system.collections.ilist?view=netcore-3.1
[idictionary]: https://docs.microsoft.com/en-us/dotnet/api/system.collections.idictionary?view=netcore-3.1
[arraylist]: https://docs.microsoft.com/en-us/dotnet/api/system.collections.arraylist?view=netcore-3.1
[list-1]: https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=netcore-3.1
