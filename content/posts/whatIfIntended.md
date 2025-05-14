---
title: "WhatIf It's Working as Intended?"
description: "When developers and users disagree"
date: "2025-05-14"
---

## The Issue

An issue on the [PowerShell GitHub repository](https://github.com/PowerShell/PowerShell/issues/25459) claims that the `Tee-Object` cmdlet exhibits unexpected behavior when `$WhatIfPreference = $true`.

The example provided creates a function that supports the `-WhatIf` parameter. This function calls `Tee-Object -Variable` which, according to the [PowerShell Docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object), 'Saves command output in a ... variable and also sends it down the pipeline.' The example includes console output when calling the function with and without the `-WhatIf` parameter.

```powershell
function Test-Tee {
    [CmdletBinding(SupportsShouldProcess)]
    Param()
    $true | Tee-Object -Variable b # Explicit call to 'Tee-Object' because I'm on a Mac and 'tee' is a built-in system command.
    $b
}

> Test-Tee
True
True

> Test-Tee -WhatIf
True
What if: Performing the operation "Set variable" on target "Name: b Value: True".
```

## The Investigation

### `-WhatIf` and `$WhatIfPreference`

First, I looked at the `-WhatIf` parameter and its functionality. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess?view=powershell-7.5#whatifpreference) and [PowerShell Docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_preference_variables#whatifpreference) explain that passing a supported command the `-WhatIf` parameter or setting `$WhatIfPreference = $true`, sets this preference for the current scope or session, respectively.

It's my understanding that no state change should occur when `$WhatIfPreference` is set to `$true`.

Next, I looked at the the issue's reported behavior, `What if: Performing the operation "Set variable" on target "Name: b Value: True".`. I wondered what the output would be if we tee a file instead of a variable.

```powershell
function Test-Tee {
    [CmdletBinding(SupportsShouldProcess)]
    Param()
    $true | Tee-Object -FilePath (Join-Path -Path $env:HOME -ChildPath 'test.txt')
    $b
}
Test-Tee -WhatIf
What if: Performing the operation "Output to File" on target...
```

As I expected, the `What if:...` message was printed to the console. It's interesting though, the messages that are output. These messages are exactly the same as if we called `Set-Variable` or `Out-File`.

```powershell
Out-File -Path (Join-Path -Path $env:HOME -ChildPath 'test.txt') -InputObject $true -WhatIf
Set-Variable -Name b -Value $true -WhatIf

What if: Performing the operation "Output to File" on target...
What if: Performing the operation "Set variable" on target "Name: b Value: True".
```

### PowerShell Source

I suspected that `Tee-Object` directly calls `Out-File` or `Set-Variable`. I forked the PowerShell repository and went digging. The code in question lives in `Tee-Object.cs`.

```csharp
// src/Microsoft.PowerShell.Commands.Utility/commands/utility/Tee-Object.cs
protected override void BeginProcessing()
{
    _commandWrapper = new CommandWrapper();
    if (string.Equals(ParameterSetName, "File", StringComparison.OrdinalIgnoreCase))
    {
        _commandWrapper.Initialize(Context, "out-file", typeof(OutFileCommand));
        ...
    }
    else if (string.Equals(ParameterSetName, "LiteralFile", StringComparison.OrdinalIgnoreCase))
    {
        _commandWrapper.Initialize(Context, "out-file", typeof(OutFileCommand));
        ...
    }
    else
    {
        // variable parameter set
        _commandWrapper.Initialize(Context, "set-variable", typeof(SetVariableCommand));
        ...
    }
}
```

Sure enough, we can see that, depending on which ParameterSet is active, `Out-File` or `Set-Variable` are being called, passing through a `Context`, which includes the `$WhatIfPreference`.

`Tee-Object` itself does not declare `SupportsShouldProcess = $true` in its attributes. However, by calling `Out-File` or `Set-Variable` and passing the current Context, it allows these underlying cmdlets (which do support ShouldProcess) to detect and respond to the `$WhatIfPreference` present in that context.

## The Findings

`Tee-Object` appears to be working as intended following PowerShell's designed behaviors around state altering commands.

When calling `Test-Tee -WhatIf`, `$WhatIfPreference` is set to `$true` for this function and any command it spawns through its scope. This is different from calling `Tee-Object -WhatIf`, which, as shown in the issue produces the expected `Tee-Object: A parameter cannot be found that matches parameter name 'WhatIf'.` error.

### The Script

I wanted to see if I could write supported PowerShell that would write to the pipeline and set a variable's value while `$WhatIfPreference = $true`.

```powershell
function Test-Tee {
    [CmdletBinding(SupportsShouldProcess)]
    Param()
    Write-Output $true -OutVariable b
    $b
}

> Test-Tee
True
True

> Test-Tee -WhatIf
True
True
```

This example does just that. By using `Write-Output`, a non-state changing command, and using the CommonParameter `-OutVariable`, we can achieve the intended behavior.

## The Better Question

`Set-Variable` clearly and accurately respects the `$WhatIfPreference`. As expected, direct variable assignment (`$var = val`) allows the variable to be set regardless of the `$WhatIfPreference` value. What's interesting though is that `-OutVariable` also allows the variable to be set. This raises another very interesting questions that should be discussed with a larger audience.

If direct variable assignment and `-OutVariable` both allow setting the value of a variable when `$WhatIfPreference = $true`, why does `Set-Variable` implement `SupportsShouldProcess`?
