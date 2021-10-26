Completing our unsafe enforcement
===

## Summary
This proposal expands the enforcement of `unsafe` such that diagnostics are issued for APIs that are marked as `unsafe`, not just ones that contain pointers.

## Motivation
The design of `unsafe` meant that C# only enforces caller be in an `unsafe` context when they are calling an API that contains a pointer type. Specifically that means types like `int*`, `void*`, etc ...

Using pointer types as the mechanism of enforcement was always a bit flawed. For example the following APIs are equally dangerous, but only one is flagged as requiring `unsafe` to call:

```c#
unsafe int* Alloc(int size);
unsafe IntPtr Alloc2(int size);
```

Even with this limitation it was still easy for C# developers to spot unsafe APIs. The vast majority either used real pointer types or existed on the `Marshal` type (which is generally understood to be an unsafe type). Overall though developers could simply look at the types involved and know that if a real pointer, `IntPtr` or `UIntPtr` was used then the operation was unsafe. 

The introduction of `Span<T>` blurred the lines here. The vast majority of `Span<T>` uses are safe but there are several APIs that very unsafe like `MemoryMarshal.CreateSpan`. Developers cannot tell at a glance which of these are or are not safe. The libraries team does have this understanding but has no mechanism by which to educate developers. 

This proposal aims to give developers the tools to require their API be used in an `unsafe` context irrespective of the types used in the API itself. 

## Detailed design
The compiler will enforce that consumption of APIs or types marked as `unsafe` occur only in an `unsafe` context. 

```c#
unsafe void M() { 
    ... 
}

M(); // Error: `M` can only be used from an unsafe context
unsafe { 
    M(); // Okay 
}
```

This will cause issues for APIs that are presently using the `unsafe` modifier as a convenience mechansim for using `unsafe` inside the method body: 

```c#
unsafe void Process() {
    const int size = 42;
    Widget* p = stackalloc Widget[size];
    Read(p);
    for (int i = 0; i < size; i++) {
        Use(p->size);
    }
}
```

The `unsafe` modifier was not expressing any safety issue with invoking the API, instead it was just a convenience to use `unsafe` code inside the implementation. It's an implementation detail incorrectly being expressed as a method modifier.  There is no reason for callers of `Process` to be in an `unsafe` context here. Once this change is in effect APIs like `Process` will need to move to the following form:

```c#
void Process() {
    unsafe {
        const int size = 42;
        Widget* p = stackalloc Widget[size];
        Read(p);
        for (int i = 0; i < size; i++) {
            Use(p->size);
        }
    }
}
```

The rule here is straight forward. If the desire is callers need to use `unsafe` to invoke the API, or differently stated it creates a potential safety issue in the caller, then use `unsafe` as a modifier. If `unsafe` is an implementation detail of the method then use an `unsafe` block.

This requires only a small change to C#. The enforcement of `unsafe` moves to APIs that are marked as `unsafe` not whether or not they contain pointer types. Given that all APIs with pointer types required `unsafe` this will not remove any existing diagnostics. 

To maintain back compat this change will only take effect when using `/langversion:11` or higher. This will be a new warning diagnostic. The reason for a warning vs. an error is this is not deemed important enough to be a .NET 7 adoption blocker. Developers should be able to adopt .NET 7, and by extension C# 11, and silence this warning if it's too disruptive. They can then come back to this when it's convenient for them and fix it up. 

## Drawbacks
This will cause a fair mount of churn in the dotnet/runtime code base as well as in our customers code. It wasn't uncommon for `unsafe` to be used as a convenience mechanism for using `unsafe` code inside a method body. Any customer that did this will need to move to using `unsafe` as an implementation detail to have the correct API shape.

## Alternatives

### Write an analyzer
This is a long standing issue in .NET but it's never risen to the point of justifying a language change. The renewed emphasis for the desire to have `unsafe` enforcement can be traced back to a handful of new types in .NET: `MemoryMarshal` and `Unsafe` are the most prominent. An alternative is to write an analyzer that raises a diagnostic if these types are used outside an `unsafe` context.

### Unsafe implementations
This change will require a lot of white space changes as a part of moving to `unsafe` blocks. That is annoying. We could explore other ways of marking method implementations as `unsafe` that avoid such a shift. As an example imagine if we allowed the first line of a method to be `unsafe;` which effectively marks the entire method body as having an `unsafe` context. 

```c#
void Process {
    unsafe;

    const int size = 42;
    Widget* p = stackalloc Widget[size];
    Read(p);
    for (int i = 0; i < size; i++) {
        Use(p->size);
    }
}
```

That particular syntax may not be appealing but possible there is a better suggestion out there which is

## Unresolved questions


## Design meetings

