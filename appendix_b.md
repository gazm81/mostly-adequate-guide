# Appendix B: Algebraic Structures Support (PowerShell Version)

In this appendix, you'll find some basic PowerShell implementations of various algebraic
structures described in the book. Keep in mind that these implementations may not be the fastest or the
most efficient out there; they *solely serve an educational purpose*.

Some methods also refer to functions defined in [Appendix A](./appendix_a.md).

## Compose

```powershell
# Emulating Compose by combining two functions F and G.
function New-Compose {
    param(
        [ScriptBlock]$F,
        [ScriptBlock]$G,
        $x
    )
    $value = & $F (& $G $x)
    return [PSCustomObject]@{
        Value = $value
        Map = {
            param($fn)
            # Map transforms the inner value.
            New-Compose -F $F -G $G -x (& $fn $this.Value)
        }
        Ap = {
            param($applyObj)
            # Ap applies a function contained in another structure.
            $newVal = ($applyObj.Value | ForEach-Object { $_($this.Value) })
            New-Compose -F $F -G $G -x $newVal
        }
    }
}
```

## Either

```powershell
class Either {
    [object]$Value
    Either([object]$x) {
        $this.Value = $x
    }
    static [Either] Of([object]$x) {
        # By convention, Either.Of creates a Right instance.
        return [Right]::new($x)
    }
}
```

#### Left

```powershell
class Left : Either {
    [bool]$IsLeft = $true
    [bool]$IsRight = $false
    Left([object]$x) : base($x) { }
    
    static [Left] Of([object]$x) {
        throw "`Of` called on class Left (value) instead of Either (type)"
    }
    [string] ToString() {
        return "Left($($this.Value))"
    }
    Left Map([ScriptBlock]$fn) { return $this }
    Left Ap([object]$other) { return $this }
    Left Chain([ScriptBlock]$fn) { return $this }
    Left Join() { return $this }
    Left Sequence([ScriptBlock]$of) { return & $of $this }
    Left Traverse([ScriptBlock]$of, [ScriptBlock]$fn) { return & $of $this }
}
```

#### Right

```powershell
class Right : Either {
    [bool]$IsLeft = $false
    [bool]$IsRight = $true
    Right([object]$x) : base($x) { }
    
    static [Right] Of([object]$x) {
        throw "`Of` called on class Right (value) instead of Either (type)"
    }
    [string] ToString() {
        return "Right($($this.Value))"
    }
    [Right] Map([ScriptBlock]$fn) {
        $result = & $fn $this.Value
        return [Either]::Of($result)
    }
    [Right] Ap([object]$f) {
        return $f.Map({ $this.Value })
    }
    [object] Chain([ScriptBlock]$fn) {
        return & $fn $this.Value
    }
    [object] Join() { return $this.Value }
    [object] Sequence([ScriptBlock]$of) { return $this.Traverse($of, { param($x) $x }) }
    [object] Traverse([ScriptBlock]$of, [ScriptBlock]$fn) {
        $mapped = & $fn $this.Value
        return $mapped.Map({ [Either]::Of($_) })
    }
}
```

## Identity

```powershell
class Identity {
    [object]$Value
    Identity([object]$x) { $this.Value = $x }
    
    static [Identity] Of([object]$x) { return [Identity]::new($x) }
    
    [string] ToString() { return "Identity($($this.Value))" }
    
    [Identity] Map([ScriptBlock]$fn) {
        $result = & $fn $this.Value
        return [Identity]::Of($result)
    }
    [object] Ap([Identity]$applyObj) {
        return $applyObj.Map({ $this.Value })
    }
    [object] Chain([ScriptBlock]$fn) {
        return ($this.Map($fn)).Join()
    }
    [object] Join() { return $this.Value }
    [object] Sequence([ScriptBlock]$of) { return $this.Traverse($of, { param($x) $x }) }
    [object] Traverse([ScriptBlock]$of, [ScriptBlock]$fn) {
        $mapped = & $fn $this.Value
        return $mapped.Map({ [Identity]::Of($_) })
    }
}
```

## IO

```powershell
class IO {
    [ScriptBlock]$UnsafePerformIO
    IO([ScriptBlock]$fn) { $this.UnsafePerformIO = $fn }
    
    static [IO] Of([object]$x) { return [IO]::new({ return $x }) }
    
    [string] ToString() { return "IO(?)" }
    
    [IO] Map([ScriptBlock]$fn) {
        return [IO]::new({
            $result = & $this.UnsafePerformIO
            return & $fn $result
        })
    }
    [IO] Ap([IO]$f) {
        return $this.Chain({ param($fn) $f.Map($fn) })
    }
    [IO] Chain([ScriptBlock]$fn) { return $this.Map($fn).Join() }
    [IO] Join() { return [IO]::new({ & (& $this.UnsafePerformIO).UnsafePerformIO }) }
}
```

## List

```powershell
class ListStructure {
    [object[]]$Values
    ListStructure([object[]]$xs) { $this.Values = $xs }
    
    static [ListStructure] Of([object]$x) { return [ListStructure]::new(@($x)) }
    
    [string] ToString() { return "List(" + ($this.Values -join ", ") + ")" }
    
    [ListStructure] Concat([ListStructure]$other) {
        return [ListStructure]::new($this.Values + $other.Values)
    }
    [ListStructure] Map([ScriptBlock]$fn) {
        $mapped = $this.Values | ForEach-Object { & $fn $_ }
        return [ListStructure]::new($mapped)
    }
    [ListStructure] Sequence([ScriptBlock]$of) { return $this.Traverse($of, { param($x) $x }) }
    [ListStructure] Traverse([ScriptBlock]$of, [ScriptBlock]$fn) {
        $result = @()
        foreach ($a in $this.Values) {
            $result += (& $fn $a)
        }
        return [ListStructure]::new($result)
    }
}
```

## Map Structure

```powershell
class MapStructure {
    [hashtable]$Value
    MapStructure([hashtable]$x) { $this.Value = $x }
    
    static [MapStructure] Of([hashtable]$x) { return [MapStructure]::new($x) }
    
    [string] ToString() {
        $entries = foreach ($key in $this.Value.Keys) { "$key: $($this.Value[$key])" }
        return "Map(" + ($entries -join ", ") + ")"
    }
    MapStructure Insert([string]$k, $v) {
        $newHash = $this.Value.Clone()
        $newHash[$k] = $v
        return [MapStructure]::Of($newHash)
    }
    [MapStructure] Map([ScriptBlock]$fn) {
        $newHash = @{}
        foreach ($key in $this.Value.Keys) {
            $newHash[$key] = & $fn $this.Value[$key]
        }
        return [MapStructure]::Of($newHash)
    }
    [MapStructure] Sequence([ScriptBlock]$of) { return $this.Traverse($of, { param($x) $x }) }
    [MapStructure] Traverse([ScriptBlock]$of, [ScriptBlock]$fn) {
        $result = @{}
        foreach ($key in $this.Value.Keys) {
            $result[$key] = & $fn $this.Value[$key]
        }
        return [MapStructure]::Of($result)
    }
}
```

## Maybe

```powershell
class Maybe {
    [object]$Value
    Maybe([object]$x) { $this.Value = $x }
    
    [bool] get_IsNothing() {
        return ($this.Value -eq $null)
    }
    [bool] get_IsJust() { return -not $this.IsNothing }
    
    static [Maybe] Of([object]$x) { return [Maybe]::new($x) }
    
    [string] ToString() {
        if ($this.IsNothing) { return "Nothing" }
        else { return "Just($($this.Value))" }
    }
    [Maybe] Map([ScriptBlock]$fn) {
        if ($this.IsNothing) { return $this }
        else { return [Maybe]::Of(& $fn $this.Value) }
    }
    [Maybe] Ap([Maybe]$other) {
        if ($this.IsNothing) { return $this }
        else { return $other.Map({ $this.Value }) }
    }
    [Maybe] Chain([ScriptBlock]$fn) { return ($this.Map($fn)).Join() }
    [object] Join() { if ($this.IsNothing) { return $this } else { return $this.Value } }
    [Maybe] Sequence([ScriptBlock]$of) { return $this.Traverse($of, { param($x) $x }) }
    [Maybe] Traverse([ScriptBlock]$of, [ScriptBlock]$fn) {
        if ($this.IsNothing) { return & $of $this }
        else { 
            $result = & $fn $this.Value
            return $result.Map({ [Maybe]::Of($_) })
        }
    }
}
```

## Task

```powershell
class Task {
    [ScriptBlock]$Fork
    Task([ScriptBlock]$fork) { $this.Fork = $fork }
    
    static [Task] Rejected([object]$x) {
        return [Task]::new({ param($reject, $resolve) $reject $x })
    }
    static [Task] Of([object]$x) {
        return [Task]::new({ param($reject, $resolve) $resolve $x })
    }
    [string] ToString() { return "Task(?)" }
    [Task] Map([ScriptBlock]$fn) {
        return [Task]::new({
            param($reject, $resolve)
            & $this.Fork $reject ([ScriptBlock]::Create("param(`$x) & $fn `$x"))
        })
    }
    [Task] Ap([Task]$f) { return $this.Chain({ param($fn) $f.Map($fn) }) }
    [Task] Chain([ScriptBlock]$fn) {
        return [Task]::new({
            param($reject, $resolve)
            & $this.Fork $reject { param($x) & (& $fn $x).Fork $reject $resolve }
        })
    }
    [Task] Join() { return $this.Chain({ param($x) $x }) }
}
```
