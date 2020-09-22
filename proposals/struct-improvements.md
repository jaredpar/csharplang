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
* The defaults chosen for these escape scopes are reasonable for the majority of 
programs. However there are many cases where the inferred escape behavior does
not line up with the actual implementation and this creates friction for 
developers.

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
Span<T> CreateSpan<T>(ref T value) => ... 
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
    Span<T> CreateSpan<T>(ref T value) => ...;

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
    ref int _field;

    public RS(object[] array)
    {
        ...
    }
    public RS(ref int i)
    {
        ref _field = ref i;
    }

    RS CreateRS(ref i) => ...;

    RS RuleExamples(ref int i, object[] array)
    {
        var rs1 = new RS(ref i);

        // ERROR by bullet 1: the safe-to-escape scope of 'rs1' is the current
        // scope.
        return rs1; 

        var rs2 = new RS(array);

        // Okay by bullet 3: the safe-to-escape scope of 'rs2' is outside the 
        // method scope.
        return rs2; 

        int local = 42;

        // ERROR by bullet 1: the safe-to-escape scope is the current scope
        return new RS(ref local);
        return new RS(ref i);

        // Okay because rules for method calls have not changed.
        return CreateRS(ref local);
        return CreateRS(ref i);
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
    public RS1(ref int p)
    {
        ref _field = ref p;
    }
}

ref struct RS2
{
    RS1 _field;
    public RS2(RS1 p)
    {
        _field = p;
    }

    public RS2(ref int i)  
        // ERROR: the *safe-to-escape* return of :this the current method scope
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
        // ERROR. Bullet 1 of the new constructor rules gives this newly created
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

This design also requires that the rules for field lifetimes be expanded as the
rules today simply don't account for them. It's important to note that our 
expansion of the rules here is not defining new behavior but rather accounting 
for behavior that has long existed. The safety rules around using `ref struct` 
fully acknowledge and account for the possibility that `ref struct` will 
contain `ref` state. Whether that was implemented as `ByReference<T>` or `ref`
fields is immaterial. 

As a part of allowing `ref` fields though we must define their rules such that
they fit into the existing consumption rules for `ref struct`. Specifically this
must account for the fact that it's legal *today* for a `ref struct` to 
return it's `ref` state as `ref` to the consumer. This is in fact how the 
indexer for `Span<T>` is implemented today. 

To understand the proposed changes it's helpful to review the existing rules
for method invocation around *ref-safe-to-escape* and how they account for 
a `ref struct` exposing `ref` state today:

> An lvalue resulting from a ref-returning method invocation e1.M(e2, ...) is *ref-safe-to-escape* the smallest of the following scopes:
> 1. The entire enclosing method
> 2. The *ref-safe-to-escape* of all ref and out argument expressions (excluding the receiver)
> 3. For each in parameter of the method, if there is a corresponding expression that is an lvalue, its *ref-safe-to-escape*, otherwise the nearest enclosing scope
> 4. the *safe-to-escape* of all argument expressions (including the receiver)

The fourth item provides the critical safety point around a `ref struct` 
exposing `ref` state to callers. When the `ref` state stored in a `ref struct` 
refers to the stack then the *safe-to-escape* scope for that `ref struct` will 
be at most the scope which defines the state being referred to. Hence limiting
the *ref-safe-to-escape* of invocations of a `ref struct` to the 
*safe-to-escape* scope of the receiver ensures the lifetimes are correct.

Consider as an example the indexer on `Span<T>` which is returning `ref` fields
by `ref` today. The fourth item here is what provides the safety here:

```cs
ref int Examples()
{
    Span<int> s1 = stackalloc int[5];
    // ERROR: illegal because the *safe-to-escape* scope of `s1` is the current
    // method scope hence that limits the *ref-safe-to-escape" to the current
    // method scope as well.
    return ref s1[0];

    // SUCCESS: legal because the *safe-to-escape* scope of `s2` is outside
    // the current method scope hence the *ref-safe-to-escape* is as well
    Span<int> s2 = default;
    return ref s2[0];
}
```

As such the *ref-safe-to-escape* rules for fields will be adjusted to the 
following:

> An lvalue designating a reference to a field, e.F, is *ref-safe-to-escape* (by reference) as follows:
> - If `F` is a `ref` field and `e` is `this`, it is *ref-safe-to-escape* from the enclosing method.
> - Else if `F` is a `ref` field it's *ref-safe-to-escape* scope is the *safe-to-escape* scope of `e`.
> - Else if `e` is of a reference type, it is *ref-safe-to-escape* from the enclosing method.
> - Else it's *ref-safe-to-escape* is taken from the *ref-safe-to-escape* of `e`.

This explicitly allows for `ref` fields being returned as `ref` from a 
`ref struct` but not normal fields (that will be covered later). 

```cs
ref struct RS
{
    ref int _refField;
    int _field;

    // Okay: this falls into bullet one above. 
    public ref int Prop1 => ref _refField;

    // ERROR: This is bullet four above and the *ref-safe-to-escape* of `this`
    // in a `struct` is the current method scope.
    public ref int Prop2 => ref _field;

    public RS(int[] array)
    {
        ref _refField = ref array[0];
    }

    public RS(ref int i)
    {
        ref _refField = ref i;
    }

    public RS CreateRS() => ...;

    public ref int M1(RS rs)
    {
        ref int local1 = ref rs.Prop1;

        // Okay: this falls into bullet two above and the *safe-to-escape* of
        // `rs` is outside the current method scope. Hence the *ref-safe-to-escape*
        // of `local1` is outside the current method scope.
        return ref local;

        // Okay: this falls into bullet two above and the *safe-to-escape* of
        // `rs` is outside the current method scope. Hence the *ref-safe-to-escape*
        // of `local1` is outside the current method scope.
        //
        // In fact in this scenario you can guarantee that the value returned
        // from Prop1 must exist on the heap. 
        RS local2 = CreateRS();
        return ref local2.Prop1;

        // ERROR: the *safe-to-escape* of `local4` here is the current method 
        // scope by the revised constructor rules. This falls into bullet two 
        // above and hence based on that allowed scope.
        int local3 = 42;
        var local4 = new RS(ref local3);
        return ref local4.Prop1;

    }
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
not be allowed. In the future if it becomes easy to [declare locals](https://github.com/dotnet/csharplang/discussions/1130)
that have *safe-to-escape* scopes which are not outside the current method then
this should be reconsidered.

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
- A `ref` field can only be `ref` assigned in the constructor of the declaring
type.

### Provide struct this escape annotation
The rules for the scope of `this` in a `struct` limit the *ref-safe-to-escape*
scope to the current method. That means neither `this`, nor any of its fields
can return by reference to the caller.

```cs
struct S
{
    int _field;
    // Error: this, and hence _field, can't return by ref
    public ref int Prop => ref _field;
}
```

There is nothing inherently wrong with a `struct` escaping `this` by reference.
Instead the justification for this rule is that it strikes a balance between the 
usability of `struct` and `interfaces`. If a `struct` could escape `this` by 
reference then it would significantly reduce the use of `ref` returns in 
interfaces.

```cs
interface I1
{
    ref int Prop { get; }
}

struct S1 : I1
{
    int _field;
    public ref int Prop => _ref field;

    // When T is a struct type, like S1 this would end up returning a reference
    // to the parameter
    static ref int M<T>(T p) where T : I1 => ref p.Prop;
}
```

The justification here is reasonable but it also introduces unnecessary
friction on `struct` members that don't participate in interface invocations. 

To remove this friction the language will provide the attribute `[RefEscapes]`.
When this attribute is applied to an instance method, instance property or 
instance accessor of a `struct` or `ref struct` then the `this` parameter will
be considered *ref-safe-to-escape* outside the enclosing method.

This allows for greater flexibility in `struct` definitions as they can begin 
returning `ref` to their fields. That allows for types like `FrugalList<T>`:

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
struct ListWithDefault<T>
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

To account for this change the "Parameters" section of the span safety document
will be updated to include the following:

- If the parameter is the `this` parameter of a `struct` type, it is 
*ref-safe-to-escape* to the top scope of the enclosing method unless the 
method is annotated with `[RefEscapes]` in which case it is *ref-safe-to-escape*
outside the enclosing method.

Misc Notes:
- A member marked as `[RefEscapes]` can not implement an `interface` method.
- The `RefEscapesAttribute` will be defined in the 
`System.Runtime.CompilerServices` namespace.

### Safe fixed size buffers
The language will relax the restrictions on fixed sized arrays such that the 
can be declared in safe code and the element type can be managed or unmanaged. 
This will make types like the following legal:

```cs
internal struct CharBuffer
{
    internal fixed char Data[128];
}
```

These declarations, much like their `unsafe` counter parts, will define a 
sequence of `N` elements in the containing type. 

For each `fixed` declaration in a type where the element type is `T` the 
language will generate a corresponding `get` only indexer method whose return
type is `ref T`. The indexer will be annotated with the `[RefEscapes]` attribute
as the implementation will be returning fields of the declaring type. The 
accessibility of the member will match the accessibility on the `fixed` field.

For example, the signature of the indexer for `CharBuffer.Data` will be the 
following:

```cs
[RefEscapes]
internal ref char <>DataIndexer(int index) => ...;
```

If the provided index is outside the declared bounds of the `fixed` array then
an `IndexOutOfRangeException` will be thrown. In the case a constant value is 
provided then it will be replaced with a direct reference to the appropriate 
element. Unless the constant is outside the declared bounds in which case a 
compile time error would occur.

The backing storage for the buffer will be generated using the 
`[InlineArray]` attribute. This is a mechanism discussed in [isuse 12320](https://github.com/dotnet/runtime/issues/12320) 
which allows specifically for the case of efficiently declaring sequence of 
fields of the same type.

This particular issue is still under active discussion and the expectation is
that the implementation of this feature will follow however that discussion
goes.

### Provide parameter escape annotations
**THIS SECTION NEEDS WORK**
One of the rules that causes repeated friction in low level code is the 
"Method Arguments must Match" rule. That rule states that in the case a a method
call has at least one `ref struct` passed by `ref / out` then none of the 
other parameters can have a *safe-to-escape* value narrower than that parameter.
By extension if there are two such parameters then the *safe-to-escape* of 
all parameters must be equal.

This rule exists to prevent scenarios like the following:

```cs
struct RS
{
    Span<int> _field;
    void Set(Span<int> p)
    {
        _field = p;
    }

    static void DangerousCode(ref RS p)
    {
        Span<int> span = stackalloc int[] { 42 };

        // Error: if allowed this would let the method return a pointer to 
        // the stack
        p.Set(span);
    }
}
```

This rule exists because the language must assume that these values can escape
to their maximum allowed lifetime. In many cases though the method 
implementations do not escape these values. Hence the friction caused here is 
unnecessary.

To remove this friction the language will provide the attribute 
`[DoesNotEscape]`. When applied to parameters the *safe-to-escape* scope of
the parameter will be considered the top scope of the declaring method. 
It cannot return outside of it. Likewise the attribute can be applied to 
instance members, instance properties or instance accessors and it will have
the same effect on the `this` parameter.

To account for this change the "Parameters" section of the span safety document
will be updated to include the following:

- If the parameter is marked with `[DoesNotEscape]`it is *safe-to-escape* to the
top scope of the containing method. Because this value cannot escape from the 
method it is not considered a part of the general *safe-to-escape* input set 
when calculating returns of this method.

**THAT RULE ABOVE NEEDS WORK**

```cs
struct RS
{
    Span<int> _field;
    void Set([DoesNotEscape] Span<int> p)
    {
        // Error: the *safe-to-escape* of p is the top scope of the method while
        // the *safe-to-escape* of 'this' is outside the method. Hence this is
        // illegal by the standard assignment rules
        _field = p; 
    }

    static RS M(ref RS rs1, [DoesNotEscape]RS rs2)
    {
        Span<int> span = stackalloc int[] { 42 };

        // Okay: The parameter here is not a part of the calculated "must match"
        // set because it can't be returned hence this is legal.
        p.Set(span);

        // Error: the *safe-to-escape* scope of 'rs2' is the top scope of this
        // method
        return rs2;
    }
}
```

Misc Notes:
- The  `DoesNotEscapeAttribute` will be defined in the 
`System.Runtime.CompilerServices` namespace.

### ref struct constraint
***THIS SECTION NEEDS WORK***

### ref struct interfaces
***THIS SECTION NEEDS WORK***

## Considerations

### Keywords vs. attributes
This design calls for using attributes to annotate the new lifetime rules for 
`struct` members. This also could've been done just as easily with
contextual keywords. For instance: `scoped` and `escapes` could have been 
used instead of `DoesNotEscape` and `RefEscapes`.

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

### Fun Samples

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
        this = default;
        _value = value;
    }

    public StackLinkedListNode(T value, ref StackLinkedListNode<T> next)
    {
        _value = value;
        ref _next = ref next;
    }
}
```

### Stephen's concerns about Span<T> compat

The question posed is essentially once we have `[RefEscapes]` what happens if we
attach that to the `Span<T>` indexer? That could potentially break compatibility
because the compiler today plays by the rules that the indexer returns are
always safe to escape to the heap.

In order to understand the impact here it's important to review the rules for
method invocation around *ref-safe-to-escape*. 

> An lvalue resulting from a ref-returning method invocation e1.M(e2, ...) is 
*ref-safe-to-escape* the smallest of the following scopes:
> 1. The entire enclosing method
> 2. The *ref-safe-to-escape* of all ref and out argument expressions (excluding 
the receiver)
> 3. For each in parameter of the method, if there is a corresponding expression 
> that is an lvalue, its *ref-safe-to-escape*, otherwise the nearest enclosing scope
> 4. the *safe-to-escape* of all argument expressions (including the receiver)

The span safety rule which impacts this scenario the most is the following is 
item 4 listed above. The span safety doc even mentions that it's critical for
`Span<T>` (although it fails to give sufficient examples to justify this).

Below are some compatibility samples for the `Span<T>` indexer that we need to
maintain here:

```cs
ref int Examples()
{
    Span<int> s1 = stackalloc int[5];
    // ERROR: illegal because the *safe-to-escape* scope of `s1` is the current
    // method scope hence that limits the *ref-safe-to-escape" to the current
    // method scope as well.
    return ref s1[0];

    // SUCCESS: legal because the *safe-to-escape* scope of `s2` is outside
    // the current method scope hence the *ref-safe-to-escape* is as well
    Span<int> s2 = default;
    return ref s2[0];
}
```

The initial proposed changes for `[RefEscapes]` were to change item 2 above to
the following:

- The *ref-safe-to-escape* of all ref and out argument expressions. This
includes the receiver if it's marked with `[RefEscapes]`.

That would impact the compatibility scenarios as follows assuming `[RefEscapes]`
was applied to the `Span<int>` indexer:

``` cs
ref int Examples()
{
    Span<int> s1 = stackalloc int[5];
    // ERROR: This is still an error but now point 2 and point 4 make it so.
    return ref s1[0];

    // ERROR: This is now made an error by point 2 above. 
    Span<int> s2 = default;
    return ref s2[0];
}
```

That new error is a critical compatibility break that we can't maintain. The 
rules need to be adjusted to account for this.

Taking a step back I think part of the issue the initial proposal didn't 
consider enough is that the rules are already support a `struct` returning 
it's fields by `ref` when those fields are nominally `ref` fields. Yes `ref`
fields aren't supported today but if you consider that  all uses of 
`ByReference<T>` today are effectively `ref` fields then it's clear this is 
true (else our safety model would simply be unsafe).

That means what we need to focus on is really two related features:

1. Changing the existing rules so they explicitly support returning a `ref` 
field by `ref`. This is effectively legal today hence the exercise is merely 
formalizing the rules around it now that developers can declare `ref` fields
2. Reworking the `[RefEscapes]` proposal such that it is focused on `ref` return
of non-ref fields. Technically it's okay to apply this to `ref` fields as well
since it's a restriction of the capabilities but emphasis should be on non-ref
fields.
