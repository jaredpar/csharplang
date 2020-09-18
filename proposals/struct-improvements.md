Struct Improvements
=====

## Summary
This proposal is an aggregation of several different proposals for `struct` 
performance improvements. The goal being a design which takes into account the
various proposals to create a single overarching feature set for `struct` 
improvements.

## Motivation

* The lack of `ref` fields mean that high performance `struct` innovation is 
limited to the dotnet/runtime repository where they can take advantage of the 
`ByReference<T>` type (which effectively are `ref` fields). It also means such
innovation in the runtime lacks necessary compiler safety in a number of cases.
* Implicit escape safety rules are effective for majority of cases but low level 
code frequently hits the friction points they create1
* `out` is assumed to escape even if it doesn't
* The rules are based on the declaration because the compiler can't peek into 
method bodies

## Detailed Design 
The rules for `ref struct` safety are defined in the 
[span-safety document](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/span-safety.md). 
This document will describe the proposed changes to this document as a result of
this design. Once accepted as an approved feature these changes will be
incorporated into that document.

### Provide ref fields
The language will allow developers to declare `ref` fields inside of a 
`ref struct`. This can be useful for example when encapsulating large 
mutable `struct` instances or defining new high performance types like `Span<T>`
in libraries besides the runtime.

The challenging part about allowing `ref` field declarations comes in defining
the rules such that `Span<T>` can be defined using a `ref` field without 
breaking compatibility with existing code. To understand this lets first
consider the likely changes to `Span<T>` once `ref` fields are supported.

```cs
// This is the eventual definition of Span<T> once we add ref fields into the
// language
struct Span<T>
{
    ref T _field;
    int _length;

    // This constructor does not exist today however it is likely to be added as
    // a part of changing Span<T> to have ref fields. It is a convenient, and
    // safe, way to create a length one span over a stack value that today 
    // requires unsafe code.
    public Span(ref T value)
    {
        ref _field = ref value;
        _length = 1;
    }
}
```

Given this definition there are two related scenarios, by the current 
rules, that we need to account for in our design. Particularly we need to ensure
that we don't reduce the *safe-to-escape* scopes of existing method signatures. 

The first scenario to consider is the existing one of a method having `ref` 
parameters and returning a `Span<T>` value.

```cs
Span<T> ConstructorLikeMethod(ref T value) => ... 
```

The *safe-to-escape* of the return value of `ConstructorLikeMethod` is outside 
the enclosing method. This is irrespective of the arguments used to invoke the 
method. That is because the *safe-to-escape* of the return is the minimum of 
the *safe-to-escape* of the arguments and since none of them are `ref struct` 
it is implicitly safe to return outside the enclosing method.

The second scenario to consider is how to define our safety rules for the 
newly added constructor. These need to specifically take into account that 
this newly added constructor could be used as the implementation detail of 
the above `ConstructorLikeMethod`

```cs
// This should be illegal
Span<T> ConstructorLikeMethod(ref T value) => new Span<T>(ref value);
```

The rules specifically need to ensure such code does not compile because it 
would represent a back compat break. Prior to the introduction of `ref` fields
all calls to `ConstructorLikeMethod` had a return which was *safe-to-escape* to
outside the enclosing method. Clearly this can no longer be true if it is 
implemented by calling our newly defined `Span<T>` constructor.

```cs
Span<int> Example()
{
    int local = 0;
    return ConstructorLikeMethod(ref local);
}
```

This code is completely legal today and **must** remain legal. Hence the 
language must ensure the rules for the constructor do not allow it to be used 
as an implementation detail of methods like `ConstructorLikeMethod`.

This tension between allowing constructors such as `Span(ref T field)` and 
ensuring compatibility with constructor like `ref struct` returning methods is 
a key pivot point in the design of `ref` fields.

To do this we will change the escape rules for a constructor on a `ref struct` 
that directly contains a `ref` field as follows:
- If the constructor contains any `ref struct` parameters then the 
*safe-to-escape* of the return will be the current method scope
- If the constructor contains any `in / out / ref` parameters then the 
*safe-to-escape* of the return will be the current
method scope.
- Else the *safe-to-escape* will be the outside the enclosing method

**NEED TO CHANGE TO CONSTRUCTOR INVOCATION TO CATCH THE THIS CALL**

***EXAMPLES**

The limiting of this rule to `ref struct` that directly contain a `ref field` 
is an important compatibility concern. Consider that the majority of `ref struct`
defined today indirectly contain `Span<T>` references. That mean by extension
they will indirectly contain `ref` fields in the future. Hence it's important
to ensure constructors like the following maintain their current lifetime 
rules. 

```cs
ref struct Example
{
    LargeStruct _largeStruct;
    Span<int> _span;

    public Example(in LargeStruct largeStruct, Span<int> span)
    {
        _largeStruct = largeStruct;
        _span = span;
    }
}
```

***EXAMPLES of Indirect***

The rules for assignment of need to be adjusted when the lvalue is a `ref field`
and the assignment is occurring by reference. Given an assignment from an
(rvalue) expression E1 which has a *ref-safe-to-escape* scope S1 to a `ref` 
field expression E2 with a *safe-to-escape* scope S2, it is an error if S2 
is larger than S1. 

**EXAMPLES OF WHY**

A `ref` field will be emitted into metadata using the `ELEMENT_TYPE_BYREF` 
signature. This is no different than how we emit `ref` locals or `ref` 
arguments. For example `ref int _field` will be emitted as
`ELEMENT_TYPE_BYREF ELEMENT_TYPE_I4`. This will require us to update ECMA335 
to allow this entry but this should be rather straight forward.

Developers can continue to initialize a `ref struct` with a `ref` field using 
the `default` expression in which case all declared `ref` fields will have the 
value `null`. Any attempt to use such fields will result in a
`NullReferenceException` being thrown.

```cs
struct S1 
{
    public ref int Value;
}

S1 local = default;
local.Value.ToString(); // throws NullReferenceException
```

While the C# language pretends that a `ref` cannot be `null` this is legal at the
runtime level and has well defined semantics. Developers who introduce `ref` 
fields into their types need to be aware of this possibility and can use the 
following [runtime helpers](https://github.com/dotnet/runtime/pull/40008) to 
check for `null` in the appropriate cases.

```cs
struct S1 
{
    private ref int Value;

    public int GetValue()
    {
        if (System.Runtime.CompilerServices.Unsafe.IsNullRef(ref Value))
        {
            throw new InvalidOperationException(...);
        }

        return Value;
    }
}
```

Misc Notes:
- A `ref` field can only be declared inside of a `ref struct` 
- A `ref` field cannot be declared `static`

### Provide parameter escape annotations
The rules for calculated the escape scopes of parameters are all inferred from 
their declarations:

- `this` cannot escape as `ref` which restricts it *ref-safe-to-escape* scope.
- `ref`, `out` or `in` parameters escape as `ref` which impact the 
*ref-safe-to-escape* of returns as well as combinations of arguments.

The defaults chosen for these escape scopes are reasonable for the majority of 
programs. However there are many cases where the inferred escape behavior does
not line up with the actual implementation and this creates friction for 
developers.

To remove this friction the language will provide two attributes that allow
developers to be explicit about the lifetime of `this` and `ref`, `out` and 
`in` parameters. These parameters are as follows:

`[RefEscapes]`: when applied to an instance method, instance property or 
instance accessor of a `ref struct` then the `this` parameter will be considered
*ref-safe-to-escape* to outside the enclosing method.
`[RefDoesNotEscape]`: when applied to a `ref`, `out` or `in` parameter then that
parameter will be considered *ref-safe-to-escape* to the top level scope of the 
enclosing method.

The "Parameters" section of the span safety spec will be adjusted to include 
these rules. 

The first attribute, `[RefEscapes]`, allows for greater flexibility in 
`ref struct` definitions as they can begin returning `ref` to their fields. That
allows for definitions like `FrugalList<T>`:

```cs
struct FrugalList<T>
{
    private T _item0;
    private T _item1;
    private T _item2;

    public int Count = 3;

    public ref T this[int index]
    {
        [RefEscapes]
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

This will naturally, by the existing rules in the span safety spec, allow
for returning transitive fields in addition to direct fields.

```cs
ref struct ListWithDefault<T>
{
    private FrugalList<T> _list;
    private T _default;

    public ref T this[int index]
    {
        [RefEscapes]
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

Members which contain the `[RefEscapes]` attribute cannot be used to implement 
interface members. This would hide the lifetime nature of the member at 
the `interface` call site and would lead to incorrect lifetime calculations.

The second attribute, `[RefDoesNotEscape]`, when placed on a `ref`, `out` or
`in` parameter prevents it from escaping as `ref` from the method. This means 
these parameters will not factor into the *ref-safe-to-escape* rules for method
invocation. 

This is helpful to developer as it provides an escape valve when developers 
run into the [method arguments must match](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/span-safety.md#method-arguments-must-match)
lifetime restriction. These attributes let the developer specify that certain
parameters do not escape which means they don't need to be considered for that
calculation.

Together these two attributes will cause the "Parameters" section of the span
safety doc to be updated to the following:

- If the parameter is a `ref`, `out`, or `in` parameter, it is 
*ref-safe-to-escape* outside the enclosing method unless it is attributed with
`[RefDoesNotEscape]` in which case it is *ref-safe-to-escape* to the top-level 
scope of the enclosing method.
- If the parameter is the `this` parameter of a `struct` type, it is 
*ref-safe-to-escape* to the top-level scope of the enclosing method unless the 
method is annotated with `[RefEscapes]` in which case it is *ref-safe-to-escape*
outside the enclosing method.

Misc Notes:
- A member marked as `[RefEscapes]` can not implement an `interface` method.
- The `RefEscapesAttribute` and `RefDoesNotEscapeAttribute` will be defined
in the `System.Runtime.CompilerServices` namespace.


**temporary life times need to be maintained for returns**
**NEED a way to mark a return of a method as "definitely stack allocated**
https://github.com/dotnet/csharplang/issues/1130

### Safe fixed size buffers

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

- https://github.com/dotnet/csharplang/issues/1130
- https://github.com/dotnet/csharplang/issues/1147
- https://github.com/dotnet/csharplang/issues/992
- https://github.com/dotnet/csharplang/issues/1314
- https://github.com/dotnet/csharplang/issues/2208
- https://github.com/dotnet/runtime/issues/32060

### Proposals

- https://github.com/dotnet/csharplang/blob/725763343ad44a9251b03814e6897d87fe553769/proposals/fixed-sized-buffers.md


#### TODO before submitting


https://github.com/dotnet/csharplang/issues/992

Consider downward facing only to fix the issues described in this comment
https://github.com/dotnet/csharplang/issues/992#issuecomment-357045213


Need to solve the difference between the following methods.

``` C#

// This is the logical, and eventual, definition of Span<T> once we add ref
// fields into the language
struct Span<T>
{
    ref T _field;
    int _length;

    // This is a ctor which is illegal today but will be legal, and likely added,
    // once we add ref fields into the language
    public Span(ref T value)
    {
        _field = ref value;
        _length = 1;
    }
}

class Example
{
    // This is a can exist today and the returned Span<T> can never escape the 
    // parameter by ref.
    Span<T> Create(ref T p) { throw null; }

    SmallSpan<T> MethodFactory()
    {
        T value;

        // This local must be "safe-to-escape" as that is a legal use of Span 
        // today
        // https://sharplab.io/#v2:EYLgtghgzgLgpgJwD4AEBMBGAsAKFygZgAJ0iBRADwjAAcAbOIgb1yLaIGUaIA7AHgAqAPiIBhBHAjxBQgBQSAZkQFEAbhDoBXOAEoiAXhE84Ad07d+w2cZMBtALrM1G7UQC+OgNy5W7X2y5ePgBLHhgRAFUoCABzOEpqejhZHX9mNPYiUJgiOgB7AGMNAyIAFjRvHEzMwP5skSgLEvFJeHk4JXyiui8M9hQAdiJG3krMt1w3IA=
        var span1 = Create(ref value);
        return span1; // This is Okay
    }

    SmallSpan<T> Constructor()
    {
        T value;

        // This must *NOT* be "safe-to-escape" as it is possible for the ctor of
        // Span<T> to return the parameter by ref once it has a ref field.
        var span1 = new Span(ref value);
        return span1; // This must be an error
    }
}
```

```cs
ref struct StackLinkedListNode<T>
{
    T _value;
    ref StackLinkedListNode<T> _next;

    public T Value => _value;

    public bool HasNext => !Unsafe.IsNullRef(ref _next);

    [EscapesRefThis]
    public ref StackLinkedListNode<T> Next 
    {
        get
        {
            if (!HasNext)
            {
                throw new InvalidOperationException("No next node");
            }

            return ref _next;
        }
    }

    public StackLinkedListNode(T value)
    {
        _value = value;
    }

    public StackLinkedListNode(T value, ref StackLinkedListNode<T> next)
    {
        _value = value;
        ref _next = ref next;
    }
}
```



#### Notes to delete
Jan: 

My preference would be to emit them as ELEMENT_TYPE_BYREF, no different from
how we emit ref locals or ref arguments.
 
For example, “ref int” would be emitted as “ELEMENT_TYPE_BYREF ELEMENT_TYPE_I4”.

Can check them for null using the Unsafe APIs described here.

https://github.com/dotnet/runtime/pull/40008

This means we can keep a `ref struct` with a `ref` field as defaultable. 

The problem we need to work around is that the span safety rules essentially 
think that the following can't capture:

```
ref struct Example
{
    ref int field;

    Example(ref int field)
}
```

Even though the parameter `field` is ref safe to escape outside the constructor
the `this` parameter is not. That is an explicit rule. Hence this constructor
cannot capture `field` as a `ref`. Can't do this implicitly either cause that would
be a breaking change ... 

Actually this is not a breaking change. We can say that ctors of `ref struct` 
which contain a `ref` field and have a `ref` parameter are downward facing. That
is the default. The type author can work around this by marking the parameter
as `[DoesNotRefEscape]`. This is back compat because `ref` fields are new hence
no one can hit this yet.

