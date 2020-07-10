---
layout: post
title: "Verify Your Module's Help with Pester v5"
date: 2020-07-08
categories: [powershell, testing, modules, help]
tags: [Pester, PowerShell, modules, BuildHelpers, "comment-based help", cbh]
---

If anyone's not yet aware, [Pester](https://github.com/pester/pester) recently released a new major version: 5.0.
This comes with a slew of breaking changes and some fancy new functionality, and some pretty solid performance improvements to boot.
A user in the PowerShell [Discord](https://aka.ms/psdiscord) server recently came across some tests written for Pester v4 that they wanted to refactor for use in Pester v5.
I decided to take a thorough look, and it proved to be quite a bit more complicated than I'd initially expected, mainly due to the way the tests were written.

## What Worked in Pester v4

The original code we'll be hacking away at comes fom [François-Xavier Cat (aka lazywinadmin)](https://lazywinadmin.com/2016/05/using-pester-to-test-your-comment-based.html) and looks like this:

```powershell
Describe "AdsiPS Module" -Tags "Module" {

    # Import Module
    #import-module C:\Test\AdsiPS\AdsiPS.psd1

    #$FunctionsList = (get-command -Module ADSIPS).Name
    $FunctionsList = (get-command -Module ADSIPS | Where-Object -FilterScript { $_.CommandType -eq 'Function' }).Name

    FOREACH ($Function in $FunctionsList)
    {
        # Retrieve the Help of the function
        $Help = Get-Help -Name $Function -Full

        $Notes = ($Help.alertSet.alert.text -split '\n')

        # Parse the function using AST
        $AST = [System.Management.Automation.Language.Parser]::ParseInput((Get-Content function:$Function), [ref]$null, [ref]$null)

        Context "$Function - Help"{

            It "Synopsis"{ $help.Synopsis | Should not BeNullOrEmpty }
            It "Description"{ $help.Description | Should not BeNullOrEmpty }
            It "Notes - Author" { $Notes[0].trim() | Should Be "Francois-Xavier Cat" }
            It "Notes - Site" { $Notes[1].trim() | Should Be "Lazywinadmin.com" }
            It "Notes - Twitter" { $Notes[2].trim() | Should Be "@lazywinadmin" }
            It "Notes - Github" { $Notes[3].trim() | Should Be "github.com/lazywinadmin" }

            # Get the parameters declared in the Comment Based Help
            $RiskMitigationParameters = 'Whatif', 'Confirm'
            $HelpParameters = $help.parameters.parameter | Where-Object name -NotIn $RiskMitigationParameters

            # Get the parameters declared in the AST PARAM() Block
            $ASTParameters = $ast.ParamBlock.Parameters.Name.variablepath.userpath

            $FunctionsList = (get-command -Module $ModuleName | Where-Object -FilterScript { $_.CommandType -eq 'Function' }).Name



            It "Parameter - Compare Count Help/AST" {
                $HelpParameters.name.count -eq $ASTParameters.count | Should Be $true
            }

            # Parameter Description
            If (-not [String]::IsNullOrEmpty($ASTParameters)) # IF ASTParameters are found
            {
                $HelpParameters | ForEach-Object {
                    It "Parameter $($_.Name) - Should contains description"{
                        $_.description | Should not BeNullOrEmpty
                    }
                }

            }

            # Examples
            it "Example - Count should be greater than 0"{
                $Help.examples.example.code.count | Should BeGreaterthan 0
            }

            # Examples - Remarks (small description that comes with the example)
            foreach ($Example in $Help.examples.example)
            {
                it "Example - Remarks on $($Example.Title)"{
                    $Example.remarks | Should not BeNullOrEmpty
                }
            }
        }
    }
}
```

There's a lot going on here, but there are a few things I'd specifically like to point out that will cause us some particular difficulty.
First on the list is the very frequent shifting of scope and shared information between `Describe`/`Context` blocks and the `It` blocks which make up the actual tests.
This _can_ be an issue in Pester v4 as well, but you'll generally only notice that if the code being used generates errors.

For a concrete example, look at how `$Help` is used in the above code -- it's created in a `Describe` block and used again in both `Context` and `It` blocks.
It's unlikely, but if that `Get-Help` call generates an error, the entire `Describe` test would fail, halting all subsequent tests.
With the way these tests are set up, whichever functions have yet to be tested would simple _not get tested_.
That isn't a huge issue in this specific case, but it's definitely something to keep in mind.

When writing Pester tests, you generally want to follow a couple very broad rules (among others):

1. As much code as possible should be _inside_ `It` or a `BeforeAll`/`BeforeEach`/`AfterAll`/`AfterEach` block.
   This ensures that when errors occur, they register as a test failure without necessarily breaking the rest of the tests completely.
1. Code used to generate test cases should utilize `-TestCases` wherever possible.
   This lets you offload all the looping logic to Pester itself, and tends to make tests run a little more smoothly.

While you can (yes, even in v5) utilise a standard loop to generate `It` test cases, you will very quickly run into issues doing that in v5.
Part of that is due to the split behaviour of Discovery and Run in Pester v5, so let's cover that real quick.

### Testing Phases in Pester v5

In Pester v5, tests happen in two _very_ distinct phases.
First, there's a `Discovery` phase:

> During `Discovery`, code in the main body of the test file, as well as code in `Describe` and `Context` blocks, is executed.
> This allows you to generate test cases, iteratively create `It` blocks directly, and gives Pester a chance to evaluate its test plan ahead of time.

Second, there's the `Run` phase:

> During the `Run` phase, only code **inside** an `It` block will be run.
> This _does_ include `It` blocks and test cases that were generated by code run during `Discovery`, but variables created during `Discovery` are typically not available during `Run`.
> For that reason, I'd recommend moving all your test case generation to proper `-TestCases` usage first &mdash; it's the only supported way to make data from `Discovery` available during `Run`.

## Limitations of `-TestCases`

At the time of writing, it's not yet possible replicate fully what the above code is doing by using `-TestCases`.
If you examine the code, you can see it's actually generating `Context` blocks.
This is still doable in Pester v5, but passing data from there into the `Run` phase can get a bit tricky.
However, we'll do our best!

## Prepare for Pester v5

The initial code is working from an assumption that all public functions for our module are contained in individual function files under the `Public` folder in the module.
We can break away from this assumption by simply querying the imported module for its exported commands, which will include compiled cmdlets as well.

The other side of things is that in order to pass data from the Discovery phase to the test runs, we'll need to utilize `-TestCases` quite heavily.
This could be done purely with `-TestCases`, by including the command names and all other information as part of the test case.
Doing things that way works quite well, but you will lose the ability to have each command in its own Context block.
If that doesn't bother you, feel free to go that route!
Personally, I appreciate the extra clarity of having a complete block for each command.

Below is a rewritten version of the above script, which should be fully compatible with Pester v5.
There are some design changes here and there, but the vast majority of the above script has been retained.
It includes a couple of variables for a module name, which should let you substitute any module name in to test against that module.

You'll notice I have a habit of placing code explicitly in regions marked `Discovery`.
This is purely something I do for clarity; Pester doesn't require it, and won't recognise it differently to anything else.
I'm hopeful that at some point in future Pester will provide an explicit `BeforeDiscovery` block for this purpose, but I'm unsure as to when that's likely to be implemented.

```powershell
#region Discovery

$ModuleName = 'MyModule'

#endregion Discovery

BeforeAll {
    $ModuleName = 'MyModule'
    Import-Module $ModuleName
}

Describe "$ModuleName Sanity Tests - Help Content" -Tags 'Module' {

    #region Discovery

    # The module will need to be imported during Discovery since we're using it to generate test cases / Context blocks
    Import-Module $ModuleName

    $ShouldProcessParameters = 'WhatIf', 'Confirm'

    # Generate command list for generating Context / TestCases
    $Module = Get-Module $ModuleName
    $CommandList = @(
        $Module.ExportedFunctions.Keys
        $Module.ExportedCmdlets.Keys
    )

    #endregion Discovery

    foreach ($Command in $CommandList) {
        Context "$Command - Help Content" {

            #region Discovery

            $Help = @{ Help = Get-Help -Name $Command -Full | Select-Object -Property * }
            $Parameters = Get-Help -Name $Command -Parameter * -ErrorAction Ignore |
                Where-Object { $_.Name -and $_.Name -notin $ShouldProcessParameters } |
                ForEach-Object {
                    @{
                        Name        = $_.name
                        Description = $_.Description.Text
                    }
                }
            $Ast = @{
                # Ast will be $null if the command is a compiled cmdlet
                Ast        = (Get-Content -Path "function:/$Command" -ErrorAction Ignore).Ast
                Parameters = $Parameters
            }
            $Examples = $Help.Help.Examples.Example | ForEach-Object { @{ Example = $_ } }

            #endregion Discovery

            It "has help content for $Command" -TestCases $Help {
                $Help | Should -Not -BeNullOrEmpty
            }

            It "contains a synopsis for $Command" -TestCases $Help {
                $Help.Synopsis | Should -Not -BeNullOrEmpty
            }

            It "contains a description for $Command" -TestCases $Help {
                $Help.Description | Should -Not -BeNullOrEmpty
            }

            It "lists the function author in the Notes section for $Command" -TestCases $Help {
                $Notes = $Help.AlertSet.Alert.Text -split '\n'
                $Notes[0].Trim() | Should -BeLike "Author: *"
            }

            # This will be skipped for compiled commands ($Ast.Ast will be $null)
            It "has a help entry for all parameters of $Command" -TestCases $Ast -Skip:(-not ($Parameters -and $Ast.Ast)) {
                @($Parameters).Count | Should -Be $Ast.Body.ParamBlock.Parameters.Count -Because 'the number of parameters in the help should match the number in the function script'
            }

            It "has a description for $Command parameter -<Name>" -TestCases $Parameters -Skip:(-not $Parameters) {
                $Description | Should -Not -BeNullOrEmpty -Because "parameter $Name should have a description"
            }

            It "has at least one usage example for $Command" -TestCases $Help {
                $Help.Examples.Example.Code.Count | Should -BeGreaterOrEqual 1
            }

            It "lists a description for $Command example: <Title>" -TestCases $Examples {
                $Example.Remarks | Should -Not -BeNullOrEmpty -Because "example $($Example.Title) should have a description!"
            }
        }
    }
}
```
