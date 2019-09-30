---
layout: post
title: Messing Around with Validation Attributes on Variables
date: YYYY-MM-DD
categories: [placeholder]
tags: [placeholder]
---

```powershell
[ValidateScript(
    {
        if ($_ -eq 0) {
            $true
        }
        else {
            exit $_
        }
    }
)][int]$ExitCode = 0
```

Intro paragraph (this will be used as the synopsis/summary/excerpt on the post listing pages)

# First Heading

Some text.
