# F# RFC FS-1053 - voidptr, IsReadOnly structs, inref, outref, ByRefLike structs, byref extension members

.NET has a new feature `Span`. This RFC adds support for this in F#.

* [x] Approved in principle [suggestion](https://github.com/fsharp/fslang-suggestions/issues/648)
* [x] Implementation: [Near completion](https://github.com/Microsoft/visualfsharp/pull/4888)
* Discussion: https://github.com/fsharp/fslang-design/issues/287

# Summary
[summary]: #summary

The main aim of the RFC is to allow effective consumption of APIs using `Span`, `Memory` and ref-returns.  

Span is actually built from several features and a library:
* The `voidptr` type
* The `NativePtr.ofVoidPtr` and `NativePtr.toVoidPtr` functions
* The `inref` and `outref` types via capabilities on byref pointers
* `ByRefLike` structs (including `Span` and `ReadOnlySpan`)
* `IsReadOnly` structs
* implicit dereference of byref and inref-returns from methods
* `byref` extension members

# Links

* [C# 7.2 feature "ref structs"](https://blogs.msdn.microsoft.com/mazhou/2018/03/02/c-7-series-part-9-ref-structs/)
* [C# 7.2 "read only structs"](https://blogs.msdn.microsoft.com/mazhou/2017/11/21/c-7-series-part-6-read-only-structs/)
* One key part of the F# compiler code for ref structs is [here](https://github.com/Microsoft/visualfsharp/blob/16dd8f40fd79d46aa832c0a2417a9fd4dfc8327c/src/fsharp/TastOps.fs#L5582)
* [C# 7.1 "readonly references"](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/readonly-ref.md)
* [C# 7.2: Compile time enforcement of safety for ref-like types.](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.2/span-safety.md)
* Issue https://github.com/Microsoft/visualfsharp/pull/4576 deals with oddities in the warnings about struct mutation.

# Motivation
[motivation]: #motivation

The main aim of the RFC is to allow effective consumption of APIs using `Span`, `Memory` and ref-returns.  

Also, parity with .NET/C#, better codegen, performance, interop.

# Detailed design
[design]: #detailed-design

There are several elements to the design.

#### Add the `voidptr` type to represent `void*`

The `voidptr` type represents `void*` in F# code, i.e. an untyped unmanaged pointer.
This type should only be used when writing F# code that interoperates
with native code.  Use of this type in F# code may result in
unverifiable code being generated.  Conversions to and from the 
`nativeint` or other native pointer  type may be required.
Values of this type can be generated
by the functions in the `NativeInterop.NativePtr` module.

#### Add `NativePtr.toVoidPtr` and `NativePtr.ofVoidPtr`

We add these:
```fsharp
//namespace FSharp.Core

module NativePtr = 
    val toVoidPtr : address:nativeptr<'T> -> voidptr
    val ofVoidPtr : voidptr -> nativeptr<'T>
```

#### Add `byref<'T, 'Kind>` to represent pointers with different capabilities

We add this type:
```fsharp
//namespace FSharp.Core

type byref<'T, 'Kind> 
```
The following capabilities are defined:
```fsharp
//namespace FSharp.Core

module ByRefKinds = 
    /// Read capability
    type In
    
    /// Write capability
    type Out
    
    /// Read and write capabilities
    type InOut
```
For example, the type `byref<int, In>` represents a byref pointer with read capabilities.
The following abbreviations are defined:
```fsharp
//namespace FSharp.Core

type byref<'T> = byref<'T, ByRefKinds.InOut>
type inref<'T> = byref<'T, ByRefKinds.In>
type outref<'T> = byref<'T, ByRefKinds.Out>
```
Here `byref<'T>` is the existing F# byref type.

A special constraint solving rule allows `ByRefKinds.InOut :> ByRefKinds.In` and `ByRefKinds.InOut :> ByRefKinds.Out` in subsumption,
e.g. most particularly at method and function argument application.  This
means, for example, that a `byref<int>` can be passed where an `inref<int>` is expected (dropping the `Out` capability) and
a  `byref<int>` can be passed where an `outref<int>` is expected (dropping the `In` capability).

Capabilities assigned to pointers get solved in the usual way by type inference. Inference variables for capabilities
are introduced each point the operators `&expr` and `NativePtr.toByRef` are used. In both cases their listed return
type of `byref<T>` is expanded to `byref<T, ?Kind>` for a new type inference variable `?Kind`.

#### `inref<T>` for readonly references and input reference parameters

The type `inref<'T>` is defined as `byref<'T, ByRefKinds.In>` to indicate a read-only byref, e.g. one that acts as an input parameter. It's primary use is to pass structures more efficiently, without copying and without allowing mutation. For example:
```fsharp
let f (x: inref<System.DateTime>) = x.Days
```
Semantically `inref` means "the holder of the byref pointer may only use it to read". It doesn't imply that other threads or aliases don't have write access to that pointer.  And `inref<SomeStruct>` doesn't imply that the struct is immutable.

By the type inference rules above, a `byref<'T>` may be used where an `inref<'T>` is expected.

For methods and properties on F# value types, the F# value type `this` paramater
is given type `inref<'T>` if the value type is considered immutable (has no mutable fields and no mutable sub-structs).

#### `outref<T>` for output reference parameters

The type `outref<'T>` is defined as `byref<'T, ByRefKinds.Out>` to indicate a write-only byref that can act as an output parameter.
```fsharp
let f (x:outref<System.DateTime>) = x <- System.DateTime.Now
```

Semantically `outref` means "the holder of the byref pointer may only use it to write". It doesn't imply that other threads or aliases don't have read access to that pointer.

By the type inference rules above, a `byref<'T>` may be used where an `outref<'T>` is expected.

#### Implicit dereference of return byrefs

F# 4.1 added return byrefs. We adjust these so that they implicitly dereference when a call such as `span.[i]` is used, which calls `span.get_Item(i)` method on `Span`. To avoid the implicit dereference you must write `&span.[i]`.

For example:
```fsharp
let SafeSum(bytes: Span<byte>) =
    let mutable sum = 0
    for i in 0 .. bytes.Length - 1 do 
        sum <- sum + int bytes.[i]
    sum
```

This does not apply to module-defined functions returning byrefs.  It specifically doesn't apply to:

* the `&` operator itself

* `NativePtr.toByRef` which is an existing library function returning a byref

As noted above, in these cases the returned type `byref<T>` is expanded to `byref<T, ?Kind>` for a new type inference variable `?Kind`.

#### Implicit address-of when calling members with parameter type `inref<'T>` 

When calling a member with an argument of type `inref<T>` an implicit address-of-temporary-local operation is applied.
```fsharp
type C() = 
    static member Days(x: inref<System.DateTime>) = x.Days

let mutable now = System.DateTime.Now
C.Days(&now) // allowed

let now2 = System.DateTime.Now
C.Days(now2) // allowed
```
This only applies when calling members, not arbitrary let-bound functions.

#### `IsReadOnly` on structs

F# structs are normally readonly, it is quite hard to write a mutable struct. Knowing a struct is readonly gives more efficient code and fewer warnings. 

Example code:

```fsharp
[<IsReadOnly>]
type S(count1: int, count2: int) = 
    member x.Count1 = count1
    member x.Count2 = count2
```

`IsReadOnly` is not added to F# struct types automatically, you must add it manually.

Using `IsReadOnly` attribute on a struct which has a mutable field will give an error.

#### ByRefLike structs

"ByRefLike" structs are stack-bound types with rules like `byref<_>` and `Span<_>`.
They declare struct types that are never allocated on the heap. These are
useful for high-performance programming as you get a set of strong checks
about the lifetimes and non-capture of these values. They are also potetnially useful for correctness
in some situations where capture must be avoided.

* Can be used as function parameters, method parameters, local variables, method returns
* Cannot be static or instance members of a class or normal struct
* Cannot be captured by any closure construct (async methods or lambda expressions)
* Cannot be used as a generic parameter

Here is an example:
```fsharp
open System.Runtime.CompilerServices

[<IsByRefLike>]
type S(count1: int, count2: int) = 
    member x.Count1 = count1
    member x.Count2 = count2
```

ByRefLike structs can have byref members, e.g.
```fsharp
open System.Runtime.CompilerServices

[<IsByRefLike>]
type S(count1: byref<int>, count2: byref<int>) = 
    member x.Count1 = count1
    member x.Count2 = count2
```

### Interoperability

* A C# `ref` return value is given type `outref<'T>` 
* A C# `ref readonly` return value is given type `inref<'T>`  (i.e. readonly)
* A C# `in` parameter becomes a `inref<'T>` 
* A C# `out` parameter becomes a `outref<'T>` 

* Using `inref<T>` in argument position results in the automatic emit of an `[In]` attribute on the argument
* Using `inref<T>` in return position results in the automatic emit of an `modreq` attribute on the return item
* Using `inref<T>` in an abstract slot signature or implementation results in the automatic emit of an `modreq` attribute on an argument or return
* Using `outref<T>` in argument position results in the automatic emit of an `[Out]` attribute on the argument

### Ignoring Obsolete attribute on `ByRefLike`

Separately, C# attaches an `Obsolete` attribute to the `Span` and `Memory` types in order to give errors in down level compilers seeing these types, and presumably has special code to ignore it. We add a corresponding special case in the compiler to ignore the `Obsolete` attribute on `ByRefLike` structs.


#### `byref` extension members

"byref" extension methods allow extension methods to modify the struct that is passed in. Here is an example of a C#-style byref extension member in F#:
```fsharp
open System.Runtime.CompilerServices
[<Extension>]
type Ext = 
    [<Extension>]
    static member ExtDateTime2(dt: inref<DateTime>, x:int) = dt.AddDays(double x)
```
Here is an example of using the extension member:
```fsharp
let dt2 = DateTime.Now.ExtDateTime2(3)
```

#### `this` on immutable struct members becomes `inref<StructType>`

The `this` parameter on struct members is now `inref<StructType>` when the struct type has no mutable fields or sub-structures. 

This makes it easier to write performant struct code which doesn't copy values.

# Examples of using `Span` and `Memory`

```fsharp
let SafeSum (bytes: Span<byte>) =
    let mutable sum = 0
    for i in 0 .. bytes.Length - 1 do 
        sum <- sum + int bytes.[i]
    sum

let TestSafeSum() = 
    // managed memory
    let arrayMemory = Array.zeroCreate<byte>(100)
    let arraySpan = new Span<byte>(arrayMemory);
    SafeSum(arraySpan)|> printfn "res = %d"

    // native memory
    let nativeMemory = Marshal.AllocHGlobal(100);
    let nativeSpan = new Span<byte>(nativeMemory.ToPointer(), 100);
    SafeSum(nativeSpan)|> printfn "res = %d"
    Marshal.FreeHGlobal(nativeMemory);

    // stack memory
    let mem = NativePtr.stackalloc<byte>(100)
    let mem2 = mem |> NativePtr.toVoidPtr
    let stackSpan = Span<byte>(mem2, 100)
    SafeSum(stackSpan) |> printfn "res = %d"
```


# Drawbacks
[drawbacks]: #drawbacks

* The addition of pointer capabilities slightly increases the perceived complexity of the language and core library.

* The addition of `IsByRefLike` types increases the perceived complexity of the language and core library.

# Alternatives
[alternatives]: #alternatives

* Considered allowing implicit-dereference when calling arbitrary let-bound functions.  

  --> However we don't normally apply this kind of implicit rule for calling let-bound functions, so decide against it.

* Make `inref`, `byref` and `outref` completely separate types rather than algebraically related

  --> The type inference rules become much harder to specify and much more "special case".  The current rules just need a couple of extra equations added to the inference engine.

* Generalize the subsumption "InOut --> In" and "InOut --> Out" to be a more general feature of F# type inference for tagged/measure/... types.

   --> Thinking this over


# Compatibility
[compatibility]: #compatibility

The additions are essentially backwards compatible.  There is one place where this is not the case.

* "Evil struct replacement" now gives an error. e.g.

```fsharp
[<Struct>]
type S(x: int, y: int) = 
    member __.X = x
    member __.Y = y
    member this.Replace(s: S) = this <- S
```
Note that the struct is immutable, except for a `Replace` method.  The `this` parameter will now be considered `inref<S>` and
an error will be reported suggesting to add a mutable field if the struc is to be mutated.

Allowing this assignment was never intended in the F# design and I consider this as fixing a bug in the F# compiler now we have the
machinery to express read-only references.

# Notes

* F# structs are not always readonly/immutable.
  1. You can declare `val mutable`.  The compiler knows about that and uses it to infer that the struct is readonly/immutable.
  2. Mutable fields can be hidden by signature files
  3. Structs from .NET libraries are mostly assumed to be immutable, so defensive copies aren't taken
  4. You can do wholesale replacement of the contents of the struct through `this <- v` in a member  (see above)

* Note that F# doesn't compile
```
type DateTime with 
    member x.Foo() = ...
```
as a "byref this" extension member where `x` is a readonly reference.  Instead you have to define a C#-style byref extension method. This means a copy happens on invocation. If it did writing performant extension members for F# struct types would be very easy.  Perhaps we should allow somthing like this

```
type DateTime with 
    member (x : inref<T>).Foo() = ...
```
or
```
type DateTime with 
    [<SomeNewCallByRefAttribute>]
    member x.Foo() = ...
```
This makes it impossible to define byref extension properties in particular. 

* Note that `IsByRefLikeAttribute` is only available in .NET 4.7.2.

```fsharp
namespace System.Runtime.CompilerServices
    open System
    open System.Runtime.CompilerServices
    open System.Runtime.InteropServices
    [<AttributeUsage(AttributeTargets.All,AllowMultiple=false); Sealed>]
    type IsReadOnlyAttribute() =
        inherit System.Attribute()

    [<AttributeUsage(AttributeTargets.All,AllowMultiple=false); Sealed>]
    type IsByRefLikeAttribute() =
        inherit System.Attribute()
```

# Unresolved questions
[unresolved]: #unresolved-questions

None
* What happens if we have two overloads, e.g. in C#

```
void Deconstruct<T, U>(in this KeyValuePair<T, U> k, out T key, out U value) { .. }
void Deconstruct<T, U>(this KeyValuePair<T, U> k, out T key, out U value) { .. }
```

* How the overload resolution interop supposed to work between F# -> C# and C# -> F#? Original Source: See https://github.com/fsharp/fslang-suggestions/issues/648#issuecomment-390157352
