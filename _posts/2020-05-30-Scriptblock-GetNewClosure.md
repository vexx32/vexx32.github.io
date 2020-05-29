---
layout: post
title: "ScriptBlocks and GetNewClosure()"
date: 2020-05-30
categories: [powershell, scripting, tips]
tags: [scriptblock]
---

Before we even begin, let's just take a moment to remember something very important.
In PowerShell, _everything_ is an object.
Although you'll normally see scriptblocks used to declare functions, or perhaps as a command parameter, and so on&hellip; they are themselves just an object.

That means you can do funny things like just arbitrarily assigning them to variables so you can use them later:

```powershell
$path = 'C:\'

$myScript = {
    Get-ChildItem -Path $path
}

& $myScript

$path = 'C:\Users'
& $myScript
```

If you run that code, you'll some some interesting results.
The `$path` variable is never defined in the scriptblock, so it actually retrieves the variable from the parent scope when it's invoked.
Handy!
Well, sometimes, anyway.
This is how scriptblocks tend to work for the most part, but it's not always what you want.

## Capturing State

Let's take a look at how we might want to process a JSON blob into a more usable PowerShell object.
Special thanks to user @MTG on the [PowerShell Slack](https://aka.ms/psslack) workspace for the fantastic test case here.
Our JSON blob looks a bit like this:

```json
{
    "name":  "server1",
    "id":  28361,
    "active":  true,
    "items":  [
        {
            "itemId":  71418,
            "itemValue":  "dk09230",
            "fieldId":  320,
            "fieldName":  "servicetag",
            "fieldDescription":  ""
        },
        {
            "itemId":  71318,
            "itemValue":  "city",
            "fieldId":  300,
            "fieldName":  "location",
            "fieldDescription":  ""
        }
    ]
}
```

So, we're looking to get out a flat object, which contains the following properties:

- `name`
- `id`
- `active`
- All the "items" in `items` as their own individual properties, where `fieldName` is the property name, and `itemValue` is the property value.

There's probably a decent number of ways to handle this, but it's difficult to handle it well if you have a handful of these JSON objects, and everything has a different set of `items` that you need as individual properties.
Without hard-coding handling for each type of `item` you might happen across in the real world, how do you (reasonably neatly) handle the variant properties you might be getting back here?

There are probably quite a few rabbit holes we could go down to help sort this out, and I'm sure somewhere in there we'd have some folks using `Invoke-Expression` sooner or later.
I'd rather not, so let's take a bit of a look at what scriptblocks can do for us.

Returning to our earlier PowerShell example, let's see how we can modify the scriptblock behaviour a bit to make things work for us.

## `GetNewClosure()`

```powershell
$path = 'C:\'

$myScript = {
    Get-ChildItem -Path $path
}.GetNewClosure()

& $myScript

$path = 'C:\Users'
& $myScript
```

If you try to run this code, you'll something a bit interesting.
Unlike the first example, where changing the value of `$path` before invoking the scriptblock the second time changes the output, this scriptblock does the exact same things both times.
Ignoring the changed value of `$path` _completely_ and just using the original value that it had when the scriptblock was created.
More properly, the value as it was when the `GetNewClosure()` method was used to create a copy of the scriptblock (we're discarding the original un-enclosed copy of the scriptblock in this instance).

That doesn't look super useful here, but it will help us, since it allows us to use the same code to create a collection of scriptblocks that behave slightly differently.
Here's how I ended up framing the code:

```powershell
$object = $json | ConvertFrom-Json

$Properties = @(
    'Name'
    'Id'
    'Active'
    foreach ($field in $object.items) {
        @{ Name = $field.fieldName; Expression = { $field.itemValue }.GetNewClosure() }
    }
)

$object | Select-Object -Property $Properties
```

To explain a little... If you weren't aware, `Select-Object` takes strings, scriptblocks, and/or hashtables as input for `-Property`, and I'm building the collection of properties ahead of time here.
That lets me use slightly neater syntax, but if you really wanted to you could pass that array literal `@( ... )` directly to the command as well.
First, we simply add the direct properties we want from the original object as strings.
Then, using a `foreach` loop I can build the hashtables necessary to add a calculated property to the final object for every `item` that was attached to the original object.

The `GetNewClosure()` here lets us use the value of `$field` in the scriptblock and have it refer to the correct `item` from the original object by capturing the state of `$field` when it was created and always using that.
This is important, because without it, `Select-Object` will just invoke that scriptblock as-is, and it will always be referring to the same `$field` value for each of the properties, which is not what we want.

There might be other ways to handle this, but probably not many ways that are quite this elegant.
The only one that really comes to mind is perhaps looping over `$object.items` and _repeatedly_ calling `Select-Object`&hellip; which is much less readable, and probably a lot slower overall.

It's not often I see a good use case for it, but I'm always glad I'm aware of it when I need it!
