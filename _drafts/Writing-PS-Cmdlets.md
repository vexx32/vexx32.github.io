---
layout: post
title: Writing PowerShell Cmdlets in C# (And Other Fun Languages)
date: YYYY-MM-DD
categories: [cmdlets, powershell, how-to]
tags: [authoring, powershell, cmdlets, csharp, fsharp, guides]
---

Intro paragraph (this will be used as the synopsis/summary/excerpt on the post listing pages)

## First Heading

Some text.


## Convo with Fred

**🌸Joel🌸 — Today at 12:13 PM**

> @Fred I have a question (non-urgent) for when you have time.
> You're probably one of the most prolific folks that writes PowerShell as compiled cmdlets over script modules.
>
> Why was that the route you chose?
> What are the specific advantages / tradeoffs you're looking at there for the purposes of your modules?

**Fred — Today at 1:18 PM**

> Well, I tend to do so in three scenarios:
>
> + When it's simply faster (e.g. the String module, which is exclusively in-memory processing)
> + When I want my own code not to be in the PS scopes (mostly when I do unspeakable things to scriptblocks users hand me; Invoke-PSFProtectedCommand is a nice example here)
> + When I want to access non-public c# content.
>   I rarely make things non-public unless it's something likely to shoot the unwary explorer so massively in the foot without knowing what hit them (e.g. in the rare instances where I go reflecting into the PowerShell engine - I don't want accidentally exposing something that in error cases crashes the console without being able to debug anything to anybody who can't take that barrier themselves)
>
> That said, the Cmdlets are very much a minority in the total amount of commands I publish.
> Most of the PSFramework Cmdlets are due to performance reasons, as they need to superscale
>
> Oh, I suppose there has also been a fourth category: "Because I felt like it" :stuck_out_tongue_closed_eyes: 
> I'm pretty sure that in some instances I might just as well have written it in script and just felt like doing it in C#