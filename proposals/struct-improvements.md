Struct Improvements
=====

## Summary
This proposal is an aggregation of several different proposals for `struct` 
performance improvements. The goal being a design which takes into account the
various proposals to create a single overarching feature set for `struct` 
improvements.

## Motivation

## Detailed Design 

### EscapesThis and DoesNotEscape

### Expanding ref safe to escape

This is about allowing a `struct` to return a `ref` to it's field in a limited 
set of circumstances.

Rules: https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/span-safety.md

### Safe fixed size buffers

### ref fields

My preference would be to emit them as ELEMENT_TYPE_BYREF, no different from
how we emit ref locals or ref arguments.
 
For example, “ref int” would be emitted as “ELEMENT_TYPE_BYREF ELEMENT_TYPE_I4”.

### Length one Span<T>

This is about allowing the following:

```cs
int i = 42;
Span<int> s = ref i;
```

https://github.com/dotnet/csharplang/issues/992

Consider downward facing only to fix the issues described in this comment
https://github.com/dotnet/csharplang/issues/992#issuecomment-357045213

### ref struct constraint

### ref struct interfaces

Detailed restrictions
* 

## Considerations

### Keywords vs. attributes
This design calls for using attributes to annotate the new lifetime rules for 
`ref` and `ref-struct` members. This also could've been done just as easily with
contextual keywords. For instance: `scoped` and `escapes` could have been 
used instead of `DoesNotEscape` and `EscapesThis`.

Keywords, even the contextual ones, have a much heavier weight in the language
than attributes. The use cases these features solve, while very valuable, 
impact a small number of developers. Consider that only a fraction of 
high end developers are defining `ref struct` instances and then consider that 
only a fraction of those developers will be using these new lifetime features.
That doesn't seem to justify adding a new contextual keyword to the language.

This does mean that program correctness will be defined in terms of attributes
though. That is a bit of a gray area for the language side of things but an 
established pattern for the runtime. 

### ref interface and default interface members
The reason a `ref struct` can't inherit a default interface implementation is
that it would be a path to allow for a `ref struct` to be boxed. Consider the 
following example:

```csharp
interface IBoxer {
    void Go() {
        object o = this;
        Console.WriteLine(o.ToString());
    }
}

ref struct S : IBoxer {

}

class Util {
    void Use<T>(T box)
        where T : ref struct, IBoxer {

        box.Go();
    }

    void SneakyBox() {
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

### Future Consideration 1

## Related Information
### Issues

- https://github.com/dotnet/csharplang/issues/1147
- https://github.com/dotnet/csharplang/issues/992
- https://github.com/dotnet/csharplang/issues/1314
- https://github.com/dotnet/csharplang/issues/2208

### Proposals

- https://github.com/dotnet/csharplang/blob/725763343ad44a9251b03814e6897d87fe553769/proposals/fixed-sized-buffers.md
