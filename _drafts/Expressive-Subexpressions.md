---
layout: post
title: Expressive Subexpressions
date: 2020-02-24
categories: [powershell, code style, good practices]
tags: [subexpression, functional, style]
---

Intro paragraph (this will be used as the synopsis/summary/excerpt on the post listing pages)

## Subexpressions and Anonymous Functions

basic summary of subexpressions and how they behave

so like `;` doesn't terminate the subexpression and it can be multiline

### Types of Subexpressions

two types of subex - @() and $() - and what to call em

#### Regular Subexpressions

how regular subexpressions work and what they're used for

#### Array Subexpressions

how array subexpressions work and what you can use em for

### Inline Scriptblock Invocations

how scriptblocks can be used to invoke things and collect output like subexs

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

$Results = & {
    Do-Things
}

$ImportedResults = . {
    Do-Things -OutVariable OtherVar
}

```
