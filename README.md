StringFormatter
===============

A zero-allocation* string formatting library for .NET applications.

Motivation
----------

The built-in string formatting facilities in .NET are robust and quite usable. Unfortunately, they also perform a ridiculous amount of GC allocations.
Mostly these are short lived, and on the desktop GC they generally aren't noticeable. On more constrained systems however, they can be painful.
Additionally, if you're trying to track your GC usage via live reporting in your program, you might quickly notice that attempts to print out
the current GC state cause additional allocations, defeating the entire attempt at instrumentation.

Thus the existence of this library. It's not completely allocation free; there are several one-time setup costs. The steady state
though is entirely allocation-free. You can freely use the string formatting utilities in the main loop of a game without
it causing a steady churn of garbage.

Quick Start
-----------

At its simplest, you can make use of the static `StringBuffer.Format` convenience method. The `StringBuffer`
formatting methods accept all of the formatting features supported by the .NET BCL.

```csharp
string result = StringBuffer.Format("{0,-8:x} some text -- {1:C11} {2} more text here {3:G}", -15, 13.4512m, true, double.MaxValue);
// output:
// "-15      some text -- 13.4512 True more text here 1.79769313486232E+308"
```

#### Allocation Analysis

Let's look at the allocations performed by the previous example and compare them to the BCL's [string.Format](https://msdn.microsoft.com/en-us/library/zf3d0ccc(v=vs.110).aspx).

|    | Mine | BCL | Explanation
---|---|---|---
Parameters | 0 | 1+4 | Boxing value types plus `params[]` array allocation
static `Format()` cache | 1 | 1 | Allocating a new `StringBuffer` / `StringBuilder` (will be cached in both cases)
Constructor | 1 | 1 | Allocation of the backing `char[]` array.
Format specifiers | 0 | 3*3 | In the BCL, each specifier in the format string results in a new `StringBuilder` allocation, an underlying buffer allocation, and then a `ToString()` call.
Each argument | 0 | 4 | The BCL calls `ToString()` on each argument.
`ToString` | 1 | 1 | No way around it, if you want a `string` instance you need to allocate.

Tally them up, we get the following totals:

|    | Mine | BCL
---|---|---
First Time | 3 | 21
Each Additional | 1 | 19

At the steady state, `StringBuffer` requires 1 allocation per format call, regardless of the number of arguments. `StringBuilder` requires 2 + 5n, where n is the number of arguments.
There is an additional cost not mentioned in the above table: each type reallocates its internal buffer when the size of the resulting string grows too large.
If you set your capacity properly and `Clear()` your buffer between format operations (as the static `Format()` methods do) you can avoid this cost entirely.

Note: that single allocation performed by `StringBuffer`, calling `ToString()` on the result, can be avoided by using additional library features described below.

Features
--------

`StringBuffer` has a similar API to `StringBuilder`. You can create an instance and set a capacity and then reuse that buffer for many operations,
avoiding any allocations in the process. 

```csharp
var buffer = new StringBuffer(128);
buffer.Append(32.53);
buffer.Clear();
buffer.AppendFormat("{0}", "Foo");
var result = buffer.ToString();
```

`StringBuffer` is fully culture-aware. Unlike the BCL APIs which require you to pass the desired `CultureInfo` around all over the
place, `StringBuffer` caches the culture during initialization and all subsequent formatting calls use it automatically.
If for some reason you want to mix and match strings for different cultures in the same buffer, you'll have to manage that yourself.

(*) If you want to avoid even the one allocation incurred by calling `ToString()` on the result of the `StringBuffer`, you can make use
of the `CopyTo` methods. These provides methods to copy the internal data to either managed buffers or to an arbitrary char pointer.
You can allocate stack memory or native heap memory and avoid any GC overhead entirely on a per-string basis:

```csharp
buffer.Append("Hello");

var output = stackalloc char[buffer.Count];
buffer.CopyTo(output, 0, buffer.Count);
```

#### Limitations

Unlike in the BCL, each argument to `StringBuffer.AppendFormat` must either be one of the known built-in types or be a type implementing `IStringFormattable`.
This new interface is the analogue to the BCL's `IFormattable`. This restriction is part of how `StringBuffer` is able to avoid boxing arguments.

If you need to work with an existing type that you don't own, you can get around the restriction by using a *custom formatter*:

```csharp 
StringBuffer.SetCustomFormatter<MyType>(FormatMyType);

void FormatMyType(StringBuffer buffer, MyType value, StringView formatSpecifier) {
}
```

Once that call has been made, you may pass instances of `MyType` to any of the format methods.

Another limitation of `StringBuffer` is that there only exist `AppendFormat` methods taking up to 8 arguments. Adding additional ones is
trivial from a development perspective, but there does exist a statically compiled limit. Thus if you want to provide more, you
need to make use of the `AppendArgSet` method. This takes an instance of an `IArgSet`, which you must implement, and formats it according
to the given format string. Whether or not this results in allocations is up to your implementation.

The format specifier for each argument is passed to the format routines via a `StringView`, which is a pointer to stack allocated
temporary memory. There is an upper limit to the size of this memory, so format specifiers are capped at a hard upper length.
Currently that length is 32, though it's easily changed in the source. Most format strings never have specifiers nearly that long; if
you're doing something crazy with specifiers though that might become a concern.

Performance
-----------

I need to do more in-depth performance analysis and comparisons, but so far my implementation is roughly on par with the BCL
versions. Their formatting routines tend to be faster thanks to having hand-coded assembly routines in the CLR, but they
also allocate a lot more so it generally ends up being a wash.

There are a few cases where I know I'm significantly slower; for example, denormalized doubles aren't great. If your
applications needs to format millions of denormalized numbers per second, you might want to consider sticking with the BCL.

To Do
-----

There is still some work to be done:

- General library cleanup and documentation
- Flesh out the StringView type.
- Unit tests
- Improved error checking and exception messages
- Custom numeric format strings
- Enums
- DateTime and TimeSpan
- Switch to using UTF8 instead of UTF16? Might be nice.

Feedback
--------

If you have any comments, questions, or want to help out, feel free to get in touch or file an issue.
