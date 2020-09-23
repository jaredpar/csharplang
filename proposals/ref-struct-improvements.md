
Ref Struct Struct Improvements
=====

## Summary

## Motivation


## Detailed Design 

### ref struct constraint
***THIS SECTION NEEDS WORK***

### ref struct interfaces
***THIS SECTION NEEDS WORK***

## Considerations

### ref interface and default interface members
The reason a `ref struct` can't inherit a default interface implementation is
that it would be a path to allow for a `ref struct` to be boxed. Consider the 
following example:

```csharp
interface IBoxer 
{
    void Go()
    {
        object o = this;
        Console.WriteLine(o.ToString());
    }
}

ref struct S : IBoxer { }

class Util
{
    void Use<T>(T box)
        where T : ref struct, IBoxer
    {
        box.Go();
    }

    void SneakyBox()
    {
        // Will box S through IBoxer.Go
        Use<S>(default);
    }
}
```

The same implementation of `Go` would not be legal on a `ref struct` as the 
compiler would flag `object o = this` as a boxing operation on a `ref struct`
instance.

The compiler doesn't, and because of reference assemblies can't, consider method
body contents when doing analysis. Hence it must assume the worst that all 
default implemented methods on an `interface` are boxing. This means a `ref 
struct` cannot inherit them or call into them.

## Open Issues

### ref struct only interfaces
Allowing a `ref struct` to implement an standard `interface` definition but not
allowing it to take advantage of default implemented members does de-value 
default implementation a bit as it adds a case where default implementations are
not universally safe.

One potential solution would be to extend the `ref` modifier so it can also 
apply to `interface` declarations and then restrict a `ref struct` to only 
implementing a `ref interface`.

That would neatly solve the default implemented member problem as it the 
compiler would prevent default members on `interface` declarations from doing
any operation that boxed the value.

At the same time though it would further split the hierarchy. It would mean 
that we'd eventually have say `IDisposable` and `IRefDisposable`. 

## Future Considerations