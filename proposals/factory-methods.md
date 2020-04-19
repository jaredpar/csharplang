Factory Methods
=====

## Summary
This proposal will extend the scenarios in which object initializers can be
used to include type factory methods in addition to constructors. 

## Motivation
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

## Detailed Design

## Questions

## Considerations 