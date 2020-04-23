Struct Improvements
=====

## Summary
This proposal is an aggregation of several different proposals for `struct` 
performance improvements. The goal being a design which takes into account the
various proposals to create a single overarching feature set for `struct` 
improvements.

## Motivation

* implicit rules have weaknesses
* `out` is assumed to escape even if it doesn't
* The rules are based on the declaration because the compiler can't peek into 
method bodies

## Detailed Design 

### Allow lifetime annotations
The rules for `ref struct` safety are defined in the 
[span-safety document](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/span-safety.md). 
These rules implicitly associate lifetime values to parameters based on their 
declaration:

- `this` cannot escape as `ref` 
- `ref`, `out` or `in` parameters escape as `ref`
- All other parameters cannot escape as `ref`

The language will provide two attributes that allow developers to be explicit
about the lifetime of `this` and `ref`, `out` and `in` parameters. This will 
give the method implementations and uses significantly more flexibility in 
their use cases.

The first is `[EscapesRefThis]`. This annotation when placed on a property or
method of a `ref struct` allows for `this` or any fields on `this` to escape
as `ref` from the member. This will allow for `ref` returning members that don't
require heap allocations.

```cs
ref struct FrugalList<T>
{
    private T _item0;
    private T _item1;
    private T _item2;

    public int Count = 3;

    [EscapesRefThis]
    public ref T this[int index]
    {
        get
        {
            switch (index)
            {
                case 0: return ref _item1;
                case 1: return ref _item2;
                case 1: return ref _item3;
                default: throw null;
            }
        }
    }
}
```

These rules explicitly allow for returning transitive fields in addition to 
normal fields.

```cs
ref struct ListWithDefault<T>
{
    private FrugalList<T> _list;
    private T _default;

    [EscapesRefThis]
    public ref T this[int index]
    {
        get
        {
            if (index >= _list.Count)
            {
                return ref _default;
            }

            return ref _list[index];
        }
    }
}
```

Accordingly the following rules around `ref struct` safety would need to be 
updated:

- The lifetime of `this` would be *ref-safe-to-escape* for the entire method
in the presence of this attribute.
- When calling a member marked `[EscapesRefThis]` the lifetime of `this` must
be considered when calculating the lifetime of a `ref` return value.

Further a member which was marked as `[EscapesRefThis]` would not be eligible
to implement `interface` members. This would hide the lifetime nature of the 
member at the `interface` call site and would lead to incorrect lifetime 
calculations.

The second is `[DoesNotRefEscape]`. This annotation when placed on an `ref`, 
`out` or `in` parameter prevents it from escaping as `ref` from the method. This
means the parameters won't be included in the lifetime calculation for `ref`
returns. Nor will be included when calculating lifetime for `Span<T>` 

BLAH
Got my tongue tied here. The issue isn't just about `ref` escape. It's about 
when there is at least one `ref struct` passed by `ref` and any other
`ref struct` input (by ref or not by ref)

This is going to be hard to write without putting people to sleep
BLAH

*temporary life times need to be maintained for returns*

Restrictions:
- A method marked as `[EscapesRefThis]` can never implement an `interface`
method

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

### Future Consideration 1

## Related Information
### Issues

- https://github.com/dotnet/csharplang/issues/1147
- https://github.com/dotnet/csharplang/issues/992
- https://github.com/dotnet/csharplang/issues/1314
- https://github.com/dotnet/csharplang/issues/2208

### Proposals

- https://github.com/dotnet/csharplang/blob/725763343ad44a9251b03814e6897d87fe553769/proposals/fixed-sized-buffers.md
