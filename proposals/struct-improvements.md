# Function Pointers

## Summary

Related Issues:

- https://github.com/dotnet/csharplang/issues/1147
- https://github.com/dotnet/csharplang/issues/992

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

```
int i = 42;
Span<int> s = ref i;
```

https://github.com/dotnet/csharplang/issues/992

Consider downward facing only to fix the issues described in this comment
https://github.com/dotnet/csharplang/issues/992#issuecomment-357045213

### ref struct constraint

### ref interfaces

## Open Issues

### Better name for scoped
The name `scoped` was chosen due to a lack of a better option. 

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

### ref interface vs. normal interfaces

## Future Considerations

### Future Consideration 1
