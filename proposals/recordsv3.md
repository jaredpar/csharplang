# Records 

## Language Building Blocks

As the records discussions have progressed it's becoming clear there are a
number of building block features that will come out of this effort. As LINQ
produced expression lambdas and extension methods, records will produce several
other features that compose together to form records. This is a brief overview
of how I envision these particular features.

This is not meant to be a full spec'ing of the below features but rather an
overview to give enough context to allow readers to see how they fit into the 
records feature.

### Init Only
Records have increased the pressure for C# to blend flexible object construction
while still supporting effectively immutable data. This support shouldn't be 
limited to records though but rather be an underlying feature of C# that records
simply build on top of. That feature is init only semantics.

The init only feature allows for types to be non-mutable but have more flexible 
creation semantics. Specifically it aims to let the creators of objects to 
set `readonly` fields or `get` only properties at the time of creation and then
once object creation has completed let the `readonly` and `get` only semantics 
take effect. 

This feature impacts only the immediate creation of the object. Once creation
is considered complete by the language, and underlying verification rules, the 
members marked as init only become `readonly`. The semantics at this point would
be indistinguishable from an object created in earlier versions of C#.

The final syntax for init only will be decided separately but for the purpose
of this document the following syntax will be used:

```cs
class C
{
    // A field which can be initialized by creators but will become 
    // readonly after construction is completed
    public init string Name;

    // An auto-implemented property which can be initialized by creators
    // but will become a get-only property after construction is completed
    public string Value { get; init; }
}
```

As noted the init only fields and properties will remain mutable until 
constrution completes. The construction of an object will be considered 
complete at the conclusion of all initializer expressions. That includes 
object, collection and `with` initializers. This means all init only members
can participate freely in initializer expressions and let developers blend 
immutability with simple construction semantics.

```cs
struct Point
{
    public init int X;
    public init int Y;
}

var p = new Point() { X = 13; Y = 42 };
```

### Name Constructor Methods



Members attributed with `[ConstructorMethod]` are free to return types that are
different than the containing type of the attributed member. It is common 
practice for generic types to have non-generic types that serve as factory 
members. 

The `[ConstructorMethod]` attribute is only considered valid when the return 
type of the method / property it's placed on has an implicit reference 
conversion to the containing type of the method / property. The intent here is
that types should own which methods are considered constructor.


init only is a problem here ... how to enforce this????

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