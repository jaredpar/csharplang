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

Today `ref` fields accomplished by using the `ByReference<T>` type which the 
runtime treats effective as a `ref` field. This means though that only the 
runtime repository can take full advantage of `ref` field like behavior and 
all uses of it require manual verification of safety. Part of the 
[motivation for this work](https://github.com/dotnet/runtime/issues/32060) is
to remove `ByReference<T>` and use proper `ref` fields in all code bases.

The challenging part about allowing `ref` fields declarations though comes in
defining rules such that `Span<T>` can be defined using `ref` fields without 
breaking compatibility with existing code. To understand the challenges here 
let's first consider the likely changes to `Span<T>` once `ref` fields are 
supported.

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

The constructor defined here presents a problem because it's return values must
necessarily have restricted lifetimes for many inputs. Consider that if a 
local parameter is passed by `ref` into this constructor that the returned 
`Span<T>` must have a *safe-to-escape* scope of the local's declaration scope.

At the same time it is legal to have methods today which take a `ref` parameter 
and return a `Span<T>`.  

```cs
Span<T> CreateSpan(ref T value) => ... 
```
These methods have essentially the same properties as the new `Span<T>`
constructor yet they have very different lifetime rules. The *safe-to-escape* 
of the return value of `CreateSpan` is outside the enclosing method that invokes
it. This is irrespective of the arguments used to invoke the method. That is 
because the *safe-to-escape* of the return is the minimum of the 
*safe-to-escape* of the arguments and since none of them are `ref struct` 
it is implicitly safe to return outside the enclosing method.

The rules must ensure the `Span<T>` constructor properly restricts the 
*safe-to-escape* scope of constructed objects while ensuring calls to methods
like `CreateSpan` remain the same. 

```cs
class C
{
    Span<T> CreateSpan(ref T value) => ...;

    Span<T> Examples<T>(ref T p, T[] array)
    {
        // Legal today, must remain legal 
        return CreateSpan(ref p); 

        // Legal today, must remain legal
        T local = default;
        return CreateSpan(ref local);

        // Legal for any possible value that could be used as an argument
        return CreateSpan(...);

        // Legal today, must remain legal
        return new Span<T>(array, 0, array.Length);

        // Must error because allowing this would necessarily change the 
        // *safe-to-escape* value of the return if it were allowed. This 
        // would break compatibility
        return new Span<T>(ref p);

        // Must error because this would be returning a reference to the
        // stack
        return new Span<T>(ref local);
    }
}
```

This tension between allowing constructors such as `Span(ref T field)` and 
ensuring compatibility with `ref struct` constructor like `CreateSpan` is a
key pivot point in the design of `ref` fields.

To do this we will change the escape rules for a constructor invocation on a 
`ref struct` that directly contains a `ref` field as follows:
- If the constructor contains any `ref struct` parameters then the 
*safe-to-escape* of the return will be the current scope
- If the constructor contains any `in / out / ref` parameters then the 
*safe-to-escape* of the return will be the current scope.
- Else the *safe-to-escape* will be the outside the method scope

Lets examine these rules in the context of samples to better understand their
impact.

```cs
ref struct RS
{
    ref int _field1;
    ref object _field2;

    public RS(object[] array) => ...;
    public RS(ref int i) => ...;
    public RS(ref object o) => ...;

    RS CreateRS(ref i) => ...;

    RS RuleExamples(ref int i, ref object o, object[] array)
    {
        var rs1 = new RS(ref i);

        // Error by bullet 1: the safe-to-escape scope of 'rs1' is the current
        // scope.
        return rs1; 

        var rs2 = new RS(array);

        // Okay by bullet 3: the safe-to-escape scope of 'rs2' is outside the 
        // method scope.
        return rs2; 

        int local = 42;

        // Error by bullet 1: the safe-to-escape scope is the current scope
        return new RS(ref local);
        return new RS(ref o);

        // Okay because rules for method calls have not changed.
        return CreateRS(ref local);
        return CreateRS(ref o);
    }
}
```

It is important to note that for the purposes of the rule above any use of 
constructor chaining via `this` is considered a constructor invocation. The
result of the chained constructor call is considered to be returning to the 
original constructor hence *safe-to-escape* rules come into play. That is 
important in avoiding examples like the following:

```cs
struct RS1
{
    ref int _field;
    public RS1(ref int p) => ...;
}

ref struct RS2
{
    RS1 _field;
    public RS2(RS1 p)
    {
        _field = p;
    }

    public RS2(ref int i)  
        // Error: the *safe-to-escape* return of :this the current method scope
        // but the 'this' parameter has a *safe-to-escape* outside the current
        // method scope
        : this(new RS1(ref i))
    {

    }
}
```

The limiting of the constructor rules to just `ref struct` that directly contain
 `ref` field is another important compatibility concern. Consider that the 
majority of `ref struct` defined today indirectly contain `Span<T>` references. 
That mean by extension they will indirectly contain `ref` fields once `Span<T>`
adopts `ref` fields. Hence it's important to ensure the *safe-to-return* rules
of constructors on these types do not change. That is why the restrictions
must only apply to types that directly contain a `ref` field.

Example of where this comes into play.

```cs
ref struct Container
{
    LargeStruct _largeStruct;
    Span<int> _span;

    public Container(in LargeStruct largeStruct, Span<int> span)
    {
        _largeStruct = largeStruct;
        _span = span;
    }
}
```

Much like the `CreateSpan` example before the *safe-to-escape* return of the 
`Container` constructor is not impacted by the `largeStruct` parameter. If the
new constructor rules were applied to this type then it would break 
compatibility with existing code. The existing rules are also sufficient for 
existing constructors to prevent them from simulating `ref` fields by storing
them into `Span<T>` fields.

```cs
ref struct RS4
{
    Span<int> _span;

    public RS4(Span<int> span)
    {
        // Legal today and the rules for this constructor invocation 
        // remain unchanged
        _span = span;
    }

    public RS4(ref int i)
    {
        // Error. Bullet 1 of the new constructor rules gives this newly created
        // Span<T> a *safe-to-escape* of the current scope. The 'this' parameter
        // though has a *safe-to-escape* outside the current method. Hence this
        // is illegal by assignment rules because it's assigning a smaller scope
        // to a larger one.
        _span = new Span(ref i);
    }

    // Legal today, must remain legal for compat. If the new constructor rules 
    // applied to 'RS4' though this would be illegal. This is why the new 
    // constructor rules have a restriction to directly defining a ref field
    static RS4 CreateContainer(ref int i) => new RS4(ref i);
}
```

The rules for assignment also need to be adjusted to account for `ref` fields.
It is only legal to `ref` assign a `ref` field in the constructor of the 
declaring type. Further the `ref` being assigned must have a 
*ref-safe-to-escape* scope outside the constructor method.

General `ref` assignment of `ref` fields outside the constructor are not allowed
because it would allow for scenarios like the following:

```cs
ref struct SmallSpan
{
    public ref int _field;

    // Notice once again we're back at the same problem as the original 
    // CreateSpan method: a method returning a ref struct and taking a ref
    // parameter
    SmallSpan TrickyRefAssignment(ref int i)
    {
        // *safe-to-escape* is outside the current method
        SmallSpan s = default;

        // The *ref-safe-to-escape* of 'i' is the same as the *safe-to-escape*
        // of 's' hence most assignment rules would allow it.
        ref s._field = ref i;

        return s;
    }

    SmallSpan BadUsage()
    {
        // If allowed this would accidentally smuggle a reference to 'i' to the
        // caller of BadUsage
        int i = 0;
        return TrickyRefAssignment(ref i);
    }
}
```

General `ref` assignment of `ref` fields could be allowed in the cases where we
knew the receiver had a *safe-to-escape* scope that was not outside the current
method scope. That is not a practical sample though hence this special case will
not be allowed. In the future if it becomes easy to declare locals that have 
*safe-to-escape* scopes which are not outside the current method then this 
should be reconsidered.

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

