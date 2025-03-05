# Appendix A: Essential Functions Support (PowerShell Version)

In this appendix, you'll find some basic PowerShell implementations of various functions described in the book.
Keep in mind that these implementations serve an educational purpose only.

Some concepts (such as currying and applicative functors) are not native to PowerShell, so these examples are minimal emulations.

## always

```powershell
function Always {
    param($a, $b)
    return $a
}
```

## compose

```powershell
function Compose {
    param(
        [ScriptBlock[]]$Functions
    )
    return {
        param($args)
        $result = $args
        for ($i = $Functions.Count - 1; $i -ge 0; $i--) {
            $result = @(& $Functions[$i] @result)
        }
        return $result[0]
    }
}
```

## curry

```powershell
# Note: Currying is not a native concept in PowerShell.
# This is a minimal emulation that creates a closure for partial application.
function Curry {
    param(
        [ScriptBlock]$Function,
        [int]$Arity = $Function.Parameters.Count
    )
    
    function Inner {
        param(
            [object[]]$Args
        )
        if ($Args.Count -lt $Arity) {
            return { param($MoreArgs) 
                & (Inner @($Args + $MoreArgs))
            }
        }
        else {
            return & $Function @Args
        }
    }
    return Inner @()
}
```

## either

```powershell
function Either {
    param(
        [ScriptBlock]$F,
        [ScriptBlock]$G,
        $e
    )
    if ($e.isLeft) {
        return & $F $e.value
    }
    else {
        return & $G $e.value
    }
}
```

## identity

```powershell
function Identity {
    param($x)
    return $x
}
```

## inspect

```powershell
function Inspect {
    param($x)
    if ($x -is [ScriptBlock]) {
        return $x.ToString()
    }
    elseif ($x -is [string]) {
        return "'$x'"
    }
    elseif ($x -is [object]) {
        $props = $x.PSObject.Properties | ForEach-Object { "$($_.Name): $(Inspect $_.Value)" }
        return "{" + ($props -join ', ') + "}"
    }
    else {
        return [string]$x
    }
}
```

## left

```powershell
class Left {
    [bool]$isLeft = $true
    [object]$value
    Left($value) {
        $this.value = $value
    }
}

function Left {
    param($a)
    return [Left]::new($a)
}
```

## liftA2

```powershell
# Note: This is only a simulation. In a functional language,
# liftA2 applies a function in a context (Applicative Functor).
# Here we assume $a1 and $a2 have Map and Ap methods defined.
function LiftA2 {
    param(
        [ScriptBlock]$Fn,
        $a1,
        $a2
    )
    return $a1.Map($Fn).Ap($a2)
}
```

## liftA3

```powershell
# Similar simulation as liftA2.
function LiftA3 {
    param(
        [ScriptBlock]$Fn,
        $a1,
        $a2,
        $a3
    )
    return $a1.Map($Fn).Ap($a2).Ap($a3)
}
```

## maybe

```powershell
function Maybe {
    param(
        $Default,
        [ScriptBlock]$Func,
        $m
    )
    if ($m.isNothing) {
        return $Default
    }
    else {
        return & $Func $m.value
    }
}
```

## nothing

```powershell
class Nothing {
    [bool]$isNothing = $true
    [object]$value = $null
}

function Nothing {
    return [Nothing]::new()
}
```

## reject

```powershell
# In this simulation, Reject throws an error to represent a rejected Task.
function Reject {
    param($a)
    throw $a
}
```

*Additional functions by Christopher Kuech may be appended here to extend functionality as needed.*
````
