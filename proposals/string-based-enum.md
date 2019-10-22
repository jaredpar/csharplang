# String based enums

## Summary

## Motivation
## Detailed Design 

### Extending params


## Open Issues

### ValueFormattableString breaking change
## Considerations

# Related Issues
This spec is related to the following issues: 

- https://github.com/dotnet/csharplang/issues/2849

# Raw Notes

- Unknown values are expected
    - The values can come from a variety of sources virtually all of which the 
    type author does not control
    - The different values cannot clash. The problem with using enums today is
    they are all numerically based under the hood. That means any strategy for 
    choosing a numeric value for the same is subject to collisions with other
    developers
- Types help because it can provide intellisense for the known values
- Use cases
    - Switch on unkwno and unknown values
    - Explicit conversion to and from string
