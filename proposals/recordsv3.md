# Records 

## Language Building Blocks

### Constructor Methods

### Init Only

### Validators

### 

```cs
public data class C(int X, int Y)
{
    public C
    {
        if (X + Y > 1000) throw new ArgumentException();
    }
}
```

## Required reading
https://github.com/dotnet/csharplang/blob/master/proposals/recordsv2.md
https://github.com/dotnet/csharplang/blob/master/proposals/records.md