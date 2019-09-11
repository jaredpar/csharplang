Nullable Scenarios
---

# Properties

## Write null but read non-null
*My property wants to accept potentially `null` values during a `set` but will 
always produce a non-null value on `get`.*

Annotate the property with `[AllowNull]`

Non-Generic: 

``` csharp
class Widget {
    string _description = string.Empty;

    [AllowNull]
    string Description {
        get => _description
        set => _description = value ?? string.Empty;
    }

    static void Test(Widget w)
    {
        w.Description = null; // ok
        Console.WriteLine(w.Description.Substring(1)); // ok
    }
}
```

Generic:

``` csharp
class Box<T> {
    T _value;

    [AllowNull]
    T Value {
        get => _value;
        set {
            if (value != null) {
                _value = value;
            }
        }
    }

    static void TestConstrained<U>(Box<U> box) where U : class?
    {
        box.Value = null; // ok
        Console.WriteLine(box.Value.ToString()); // ok
    }

    static void TestUnconstrained<U>(Box<U> box, U value)
    {
        box.Value = default(U); // warn on default of unconstraind type param
        box.Value = value; // ok
        Console.WriteLine(box.Value.ToString()); // ok
    }
}
```


# Control Flow

## Null tests update nullable state
When a pure null test is done on a value then the compiler will consider the 
value non-null on the true branch and maybe null on the false branch.

``` csharp
class Example {
    void M(string? s) {
        s.ToString(); // warning possible null ref

        if (s != null) {
            s.ToString(); // ok
        }
    }
}
```

This is true whether or not the type in question is marked as nullable. Even a 
non-nullable reference type will be changed to have a potentially null value 
on the false branch of a null check.

``` csharp
class NonNullableExample {
    void M(string s) {
        s.ToString(); // ok

        if (s != null) {
            s.ToString(); // ok
        }
        else {
            s.ToString(); // warning possible null ref
        }
    }
}
```

## Logical 
Logical operators factor into the nullable state of the tested values

``` csharp
class Widget {
    string? Description; 
}
class Example {
    void M(Widget? w) {
        if (w != null && w.Description != null) {
            w.ToString(); // ok
            w.Description.ToString(); // ok
        }
    }
}
```

## Binary