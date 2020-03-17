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

### Named Constructor Methods
Records need to support initializer style expressions in scenarios other than
standard object construction to support `with` expressions. This though is 
simply adding a special property to a specific named method in a type. Records
though are not alone in wanting to denote methods as being used exclusivel for
construction. Factory methods serve the same purpose in .NET and are common
throughout our framework.

The ability to mark methods as being used exclusively for creation should 
become a standalone feature. Developers will be able to mark members which 
exist soley to create objects with `[ConstructorMethod]` and the language will
give such methods the same flexibility as object constructors. That means 
those methods can now be used in combination with object and collection 
initializers.

```cs
public class Widget
{
    public string Name { get; set; }
    public int Id { get; init; }

    internal Widget() { }
}

public static class WidgetFactory
{
    [ConstructorMethod]
    public static Create() => new Widget();
}

var w = WidgetFactory.Create()
{
    Name = "JaredPar",
    Id = 42;
}
```

Members annotated with `[ConstructorMethod]` have the following restrictions:
- Cannot be a `void` returning method
- The return expression must be a `new` object expression, a member invocation 
where the member is marked as `[ConstructorMethod]`, `null` or `default`.

### Validators
Records need a way to validate all the state be it nominal or positional. This
is true both when the record is initially created and when it is altered via
a `with` expression. Types which seek to participate in object initializers
are no different than records here. Object initializers have always been a
trade off between ease of construction and validation.

Validators are a general solution that records can build upon. Validators are 
a method which is run at the conclusion of object construction that allow types
to validate the final constructed state of the object.

This feature ties in with the init only feature in that both require a more 
precise definition of multi-phase object construction. Validators run at the 
end of construction and once complete all `init` fields and properties become
`readonly`. That means validators get to view the final state of the object
and ensure it meets the contract expected by the type.

The construction sequence of an object will now be:

- `new` 
- Pass through `[ConstructorMethod]` methods
- Process initializers including `init` construction
- Run validator

The final syntax for validators will be decided separately but for the purpose 
of this document it will be the type name withot modifiers or parameter list.

```cs
public class C
{
    public int X { get; set; }
    public int Y { get; init; }
    C
    {
        if (X + Y > 1000) throw new ArgumentException();
    }
}
```

### Covariant Returns
Records need to 


## Embracing Memberwise Clone
This propsal is exploring what records would look like if `With` was truly a 
member wise clone operation. Concretely what if the signutare for `With` on 
type `T` was always the following no matter how many positional or nominal 
members were added:

```cs
public T With();
```

To explore this lets separate out the changes necessary to support the `With` 
consumption to those necessary for `With` implementation.

### With expressions
The generated `With` method for a record will be annotade with the 
`[ConstructorMethod]` attribute. That means a `with` expression is effectively 
beautiful syntax for calling the appropriate `With` method on the object. For 
many records the following will be effectively equivalent. 

```cs
var r = new Record(...);
r with { X = 42 };
r.With() { X = 42 };
```

This also means the rules for initializer expressions are the same as `with` 
expressions. The compiler is simply taking advantage of the
`[ConstructorMethod]` behavior here to treat it as a new object instance on 
which initializer expressions are now legal. 

To support the use of object initializers on records the code generation for 
positional parameters will include a `init` accessor in addition to the `get`
accessor.

```cs
record class Point(int X, int Y);

// generates
class Point {
    public int X { get; init; }
    public int Y { get; init; }

    public Point(int X, int Y) {
      this.X = X;
      this.Y = Y;
    }
}
```

The `init` accessor means that records, and their non-record expanded counter
parts, can play freely with both object initializers and `with` expressions
a like.

```cs
record class Point(int X, int Y); 

var r = new Point(42, 13);
r = r with { X = 13 };
```

There is no special casing for records needed here, it's just a fall out of 
existing C# building block features

### With implementation
The 


### Record Validators


## Considerations

### init only and ConstructorMethod
One important property to keep in mind with `[ConstructorMethod]` is that it 
must not violate the rules of init only and validators. The language will decide
on rules for when object creation completes and `init` members flip to
`readonly` and the validators is run. These rules will eventually be codified in
IL verification rules. 

The `[ConstructorMethod]` feature must work within the bounds of these rules.
That is why it's limited to returning `new` objects. This means it will 
trivially meet the contract because the method does not have any references 
to the object which would be percieved as fully constructed that it could 
pass around even though the caller could still be freely mutating the object.
It's also clear that the caller is responsible for running the validator method
not the callee. 

One potential extension though is that we should allow more flexibility in 
`[ConstructorMethod]` implementations where the return type of the member is 
the same type as the containing type. The implication being that types should
understand their contract and hence it's reasonable for them to create a new 
instance, change internal state via method calls, field changes, etc ... and 
then return the final instance. The type here is responsible for it's own 
contract and hence is only hurting itself if it does illegal things. 

Another potentional extension is to allow `unsafe` to violate normal 
`[ConstructorMethod]` rules here. After all violating these rules is nominally 
an IL safety violation so using `unsafe` as an escape is a sensible plan.

### Covariant returns
This gets even better with covariant returns

## Required reading
https://github.com/dotnet/csharplang/blob/master/proposals/recordsv2.md
https://github.com/dotnet/csharplang/blob/master/proposals/records.md