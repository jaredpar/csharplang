# Function Pointers

## Summary

Related Issues:

- https://github.com/dotnet/csharplang/issues/1147
- https://github.com/dotnet/csharplang/issues/992

## Motivation

## Detailed Design 

### scoped values

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

## Open Issues

### Better name for scoped
The name `scoped` was chosen due to a lack of a better option. 


## Future Considerations

### Future Consideration 1