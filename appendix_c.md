# Appendix C: Pointfree Utilities (PowerShell Version)

In this appendix, you'll find pointfree versions of some classic functions described in the book.  
All of the following functions are available in the global context. Keep in mind that these implementations  
may not be the fastest or the most efficient; they *solely serve an educational purpose*.

Note that these functions refer to the Curry and Compose functions defined in [Appendix A](./appendix_a.md).

## add

```powershell
# add :: Number -> Number -> Number
$add = Curry({
    param($a, $b)
    $a + $b
})
```

## append

```powershell
# append :: String -> String -> String
# Defined as the flipped concat.
$append = $flip $concat
```

## chain

```powershell
# chain :: Monad m => (a -> m b) -> m a -> m b
$chain = Curry({
    param($fn, $m)
    $m.chain($fn)
})
```

## concat

```powershell
# concat :: String -> String -> String
$concat = Curry({
    param($a, $b)
    $a + $b
})
```

## eq

```powershell
# eq :: Eq a => a -> a -> Boolean
$eq = Curry({
    param($a, $b)
    $a -eq $b
})
```

## filter

```powershell
# filter :: (a -> Boolean) -> [a] -> [a]
$filter = Curry({
    param($fn, $xs)
    $xs | Where-Object { & $fn $_ }
})
```

## flip

```powershell
# flip :: (a -> b -> c) -> b -> a -> c
$flip = Curry({
    param($fn, $a, $b)
    & $fn $b $a
})
```

## forEach

```powershell
# forEach :: (a -> ()) -> [a] -> ()
$forEach = Curry({
    param($fn, $xs)
    $xs | ForEach-Object { & $fn $_ }
})
```

## head

```powershell
# head :: [a] -> a
$head = {
    param($xs)
    $xs[0]
}
```

## intercalate

```powershell
# intercalate :: String -> [String] -> String
$intercalate = Curry({
    param($sep, $xs)
    $xs -join $sep
})
```

## join

```powershell
# join :: Monad m => m (m a) -> m a
$join = {
    param($m)
    $m.join()
}
```

## last

```powershell
# last :: [a] -> a
$last = {
    param($xs)
    $xs[$xs.Count - 1]
}
```

## map

```powershell
# map :: Functor f => (a -> b) -> f a -> f b
$map = Curry({
    param($fn, $f)
    $f | ForEach-Object { & $fn $_ }
})
```

## match

```powershell
# match :: RegExp -> String -> Boolean
$match = Curry({
    param($re, $str)
    $str -match $re
})
```

## prop

```powershell
# prop :: String -> Object -> a
$prop = Curry({
    param($p, $obj)
    # Assumes $obj is a hashtable or PSObject with a property $p.
    $obj[$p]
})
```

## reduce

```powershell
# reduce :: (b -> a -> b) -> b -> [a] -> b
$reduce = Curry({
    param($fn, $zero, $xs)
    $acc = $zero
    foreach ($x in $xs) {
        $acc = & $fn $acc $x
    }
    $acc
})
```

## replace

```powershell
# replace :: RegExp -> String -> String -> String
$replace = Curry({
    param($pattern, $rpl, $str)
    $str -replace $pattern, $rpl
})
```

## reverse

```powershell
# reverse :: [a] -> [a]
$reverse = {
    param($x)
    if ($x -is [array]) {
        $copy = @($x)
        [Array]::Reverse($copy)
        $copy
    }
    else {
        $arr = [char[]]$x
        [Array]::Reverse($arr)
        -join $arr
    }
}
```

## safeHead

```powershell
# safeHead :: [a] -> Maybe a
$safeHead = {
    param($xs)
    & Compose -Functions @({ Maybe.Of }, $head) $xs
}
```

## safeLast

```powershell
# safeLast :: [a] -> Maybe a
$safeLast = {
    param($xs)
    & Compose -Functions @({ Maybe.Of }, $last) $xs
}
```

## safeProp

```powershell
# safeProp :: String -> Object -> Maybe a
$safeProp = Curry({
    param($p, $obj)
    (& Compose -Functions @({ Maybe.Of }, { param($o) & $prop $p $o })) $obj
})
```

## sequence

```powershell
# sequence :: (Applicative f, Traversable t) => (a -> f a) -> t (f a) -> f (t a)
$sequence = Curry({
    param($of, $f)
    & $f.sequence($of)
})
```

## sortBy

```powershell
# sortBy :: Ord b => (a -> b) -> [a] -> [a]
$sortBy = Curry({
    param($fn, $xs)
    $xs | Sort-Object -Property { & $fn $_ }
})
```

## split

```powershell
# split :: String -> String -> [String]
$split = Curry({
    param($sep, $str)
    $str -split $sep
})
```

## take

```powershell
# take :: Number -> [a] -> [a]
$take = Curry({
    param($n, $xs)
    $xs[0..($n - 1)]
})
```

## toLowerCase

```powershell
# toLowerCase :: String -> String
$toLowerCase = {
    param($s)
    $s.ToLower()
}
```

## toString

```powershell
# toString :: a -> String
$toString = {
    param($x)
    [string]$x
}
```

## toUpperCase

```powershell
# toUpperCase :: String -> String
$toUpperCase = {
    param($s)
    $s.ToUpper()
}
```

## traverse

```powershell
# traverse :: (Applicative f, Traversable t) => (a -> f a) -> (a -> f b) -> t a -> f (t b)
$traverse = Curry({
    param($of, $fn, $f)
    & $f.traverse($of, $fn)
})
```

## unsafePerformIO

```powershell
# unsafePerformIO :: IO a -> a
$unsafePerformIO = {
    param($io)
    & $io.unsafePerformIO()
}
```
