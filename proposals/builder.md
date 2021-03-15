# Interpolated string builder

## Goals
The key items this design intends to address:

1. Putting control of the interpolated builder on the method.
1. Passing additional arguments to the interpolated builder constructor
1. Passing additional arguments to the interpolated builder `GetResult` method
1. Allowing for interpolated builders to return non-string results
1. Allowing builders to early exit and avoid all formatting calls

This design also specifically excludes the following scenarios:

1. Non-C# support. The methods will poisoned in a way that prohibits non-C# 
languages from using them (same as we did for `Span<T>`). Other languages
can replicate the same code gen pattern as C# or change to recognize the 
builder pattern.

### Method definition
Methods can define a custom builder to handle interpolated string formatting 
by annotating themselves with the `[InterpolatedBuilder]` attribute. 

```c#
[InterpolatedBuilder(typeof(ValueStringBuilder))]
static extern string Format(string text);
```

This declaration causes C# to build the string using the builder 
`ValueStringBuilder` rather than the standard `string.Format` approach. Much 
like `string.Format` the use of `ValueStringBuilder` will be inlined into the 
calling method. 

The method is marked as `extern` because there is no need for an implementation
here. The entire interpolated string processing is done in the calling method
hence there is no code to emit here. This is just a place holder to tell the 
compiler how to build the string. 

Let's look at an example of using the `Format` method above and how it is 
translated into a builder. 

```c#
// User writes
return Format($"Hello {name}");

// is translated to 
var builder = new ValueStringBuilder(baseLength: 6, holeCount: 1);
builder.TryFormat("Hello ");
builder.TryFormat(name);
return builder.GetResult();
```

Notice that the `Format` method is never called here. It was completely erased 
as the code was inlined into the calling function. This is intentional as the 
`Format` method is just telling us "how" to format the `string`, it doesn't
actually do any work, that is the job of the builder. 

To pass arguments to the builder constructor the builder method simply needs
to add parameters before the `string text` argument. Arguments to these 
parameters are positionally mapped to the builder constructor. 

For example consider the following sample for passing a `Span<byte>` into the 
builder.

```c#
ref struct CustomBuilder
{
    public CustomBuilder(Span<char> span, int baseLength, int holeCount) { ... }
    public string GetResult() => ...;
}

[InterpolatedBuilder(typeof(CustomBuilder))]
static extern string Format(Span<char> span, string text);

string name = ...;
Span<char> span = stackalloc char[100];
var result = Format(span, "hello {name}");

// Translates into 
var builder = new CustomBuilder(span, baseLength: 6, holeCount 1);
builder.TryFormat("hello ");
builder.TryFormat(name);
var result = builder.GetResult();
```

Any number of parameters can be passed in this manner and they can be marked as
`in / out / ref`. The compiler will verify when compiling a method that has 
`[InterpolatedBuilder]` attribute that a compatible constructor exists for the 
parameters in the signature. 

When a `[InterpolatedBuilder]` attribute appears on an instance method then 
the receiver is included in the list of arguments passed to the builder 
constructor. It will appear as the first argument. 

```c#
ref struct CustomBuilder
{
    public CustomBuilder(Container container, Span<char> span, int baseLength, int holeCount) { ... }
    public string GetResult() => ...;
}

struct Container
{
    [InterpolatedBuilder(typeof(CustomBuilder))]
    public extern string Format(Span<char> span, string text);
}

var container = new Container()
string name = ...;
Span<char> span = stackalloc char[100];
var result = container.Format(span, "hello {name}");

// Translates into 
var builder = new CustomBuilder(container, span, baseLength: 6, holeCount 1);
builder.TryFormat("hello ");
builder.TryFormat(name);
var result = builder.GetResult();
```

Parameters that came after the `string text` argument must be marked with the 
`[GetResultArgument]`. These would be passed to the generated `GetResult` call.

Just as with constructor parameters there can be any number of these and they 
can have `in / out / ref` modifiers. The compiler will also validate the correct
`GetResult` signature exists when processing a method marked with 
`[InterpolatedBuilder]`.

```c#
ref struct CustomBuilder
{
    public CustomBuilder(Span<char> span, int baseLength, int holeCount) { ... }
    public string GetResult();
    public string GetResult(out int written);
}

[InterpolatedBuilder(typeof(CustomBuilder))]
static extern string Format(
    Span<char> span,
    string text,
    [GetResultArgument] out int written);

string name = ...;
Span<char> span = stackalloc char[100];
var result = Format(span, "hello {name}", out int written);

// Translates into 
var builder = new CustomBuilder(span, baseLength: 6, holeCount 1);
builder.TryFormat("hello ");
builder.TryFormat(name);
var result = builder.GetResult(out int written);
```

The return type of a `[InterpolatedBuilder]` method is not limited to `string`. 
It can be any type. The only limitation is that the chosen `GetResult` method
must have the same return type as the method attributed with
 `[InteroplatedBuilder]` 

Constructors on builder types can optionally end with an `out bool` parameter.
The value provided to this parameter will be generated by the compiler. In the 
case the value is `false` after completion of the constructor the compiler will
not make any further calls to the builder and the result of the expression will
be `default(T)` where `T` is the return type of the chosen `GetResult`. 

This allows for builders to have a quick out when they know that interpolation 
is not going to succeed. For example if logging is currently disabled:

```c#
class Logger 
{
    public struct Builder
    {
        public Builder(Logger logger, out bool enabled)
        {
            if (!logger.Enabled)
            {
                this = default;
                enabled = false;
                return;
            }

            ...
        }

        public void GetResult() => { ... }
    }

    public bool Enabled { get; set; }

    [InterpolatedBuilder(Builder)]
    public extern void Log(string text);
}

var logger = new Logger();
string name = ...;

// This call
logger.Log($"hello {name}");

// translates to 
var builder = new Logger.Builder(logger, out var enabled);
if (enabled) {
    builder.TryFormat("hello ");
    builder.TryFormat(name);
    builder.GetResult();
}
```

Restrictions for a method attributed with `[InterpolatedBuilder]`:

1. The last parameter not marked with `[GetResultArgument]` must be of type 
`string`
1. The builder must have a constructor which has one of the following signatures 
where `T1` through `TN` map to the type of the parameters which appear before 
`string text`. In the case this is defined on an instance method then `T1` is 
the type of the receiver:
    - `Builder(T1 t1, ... TN tn)`
    - `Builder(T1 t1, ... TN tn, int baseLength, int holeCount)`
    - `Builder(T1 t1, ... TN tn, int baseLength, int holeCount, out bool enabled)`
    - `Builder(T1 t1, ... TN tn, out bool enabled)`
2. The builder must have a `GetResult` method  which has the following signature
where `T1` through `TN` map to the type of the parameters which are marked with
`[GetResultArgument]` and return type `TReturn` which maps to the return type of
the attributed method: `TReturn GetResult(T1 t1, ... TN tn)`