- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Remove the implicit `T: Sized` restriction on `MaybeUninit<T>`.

# Motivation
[motivation]: #motivation

The ergonomics of maybe-uninit slices and arrays have been very thoroughly investigated, and countless revisions to the existing set of methods involving `[MaybeUninit<T>]` and `[MaybeUninit<T>; N]` have been made. Ultimately, it makes sense to have an uninitialized slice of data or an uninitialized array of data, but until this point, the concept of `MaybeUninit<[T]>` has not really been explored.

Allowing `MaybeUninit` to contain a potentially unsized values has a lot of ergonomic benefits. For example, an `IndexMut` implementation for `MaybeUninit<[T]>` could allow obtaining `&mut MaybeUninit<T>` references to individual elements of the slice, or `&mut MaybeUninit<[T]>` to particular sections of the slice. Since this implementation would never return actual references to data, and only references to `MaybeUninit`-wrapped data, it would still be sound.

Additionally, the compiler will be able to automatically deref `MaybeUninit<[T; N]>` to `MaybeUninit<[T]>` and provide access to all of these methods on arrays as well, while still letting us call `assume_init` after the array is written to. In effect, we can drastically reduce the API surface area by letting users use the regular methods of `MaybeUninit<T>` to interact with slices instead of having to have dedicated methods for every operation the user wishes to perform involving slices or arrays.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`MaybeUninit<T>` supports unsized types in addition to sized types. This means that, in addition to your ordinary, sized types like your simple structs and arrays, you can also have maybe-uninit slices or trait objects. The compiler will also dynamically unsize types inside a `MaybeUninit` container, meaning that you can call methods for `MaybeUninit<[T]>` on `MaybeUninit<[T; N]>`, or `MaybeUninit<dyn Trait>` on `MaybeUninit<T>` if `T: Trait`.

It's important to understand that `MaybeUninit` specifically refers to the *data* stored for a type, not the *metadata*. Most importantly, this means that we'll always be able to determine the size and alignment of the memory underneath any `MaybeUninit` reference, even if the actual data in that memory may not be initialized. In the case of slices, this metadata is the length of the slice, and in the case of trait objects, this metadata determines the concrete type\* for the trait object. Any value or reference to `MaybeUninit<T>` for an unsized `T` will include this metadata alongside the value or reference, so that anyone interacting with `MaybeUninit<T>` can determine its size and alignment when writing to it.

\*: Technically, it doesn't determine the concrete type, but the vtable for the trait, unless that trait is a subtrait of `Any`. However, this distinction doesn't really matter in this context, since the vtables for two different types will in general be different as well.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Update `pub struct MaybeUninit<T>` to `pub struct MaybeUninit<T: ?Sized>`. That's it. Since `ManuallyDrop<T>` already supports unsized types, nothing else needs to be done to make this work; the main barriers are whether we want to support this or not.

# Drawbacks
[drawbacks]: #drawbacks

The main downside of this approach is that because the entire unsized type is included inside the `MaybeUninit` wrapper, users may falsely assume that the *metadata* for the type is uninitialized in addition to the *data*, which is not the case. In the case of slices, this means that the length of the uninitialized slice will always be explicitly initialized, even if the data inside the slice is not. Additionally, allowing unsized types inside `MaybeUninit` opens up the door to putting trait objects or other unsized objects into `MaybeUninit`, which can be further confusing for users.

While the metadata for these types will be explicitly initialized, that does not mean that it has to be static. Types like `Box<MaybeUninit<[T]>>` open the door for dynamically resizing the uninitialized buffer while keeping it uninitialized. We can even work with trait objects in this way too, although this RFC does not lay out any explicit plans for such APIs. The goal here is to simply allow unsized types in `MaybeUninit` wrappers and to cement the idea that maybe-uninit *data* does not mean maybe-uninit *metadata*, and determining whether this outweights the benefits of unsized `MaybeUninit` is the purpose of this RFC.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The primary alternative is creating slice- and array-specific methods, which has been explored and implemented for `MaybeUninit` on nightly in depth. It's the opinion of this RFC that this is less ergonomic than simply allowing unsized `MaybeUninit`, but that's what we're discussing right now. Depending on who you ask, not supporting trait objects may be either a benefit or a downside to this approach.

Wrapped methods specifically involve creating separate methods that operate on arrays and slices. In effect, you create a `[MaybeUninit<T>; N]` (now doable via `[MaybeUninit::uninit(); N]` without any dedicated methods) and then provide several methods which let you operate on `&mut [MaybeUninit<T>]` references. The obvious downside to this approach is that we explicitly have to create slice-versions of methods like `write_slice`, `slice_assume_init_ref`, and `slice_assume_init_mut`.

Right now, these are all static methods of `MaybeUninit`, meaning that they must be called as ordinary functions (`MaybeUninit::write_slice(dest, src)`) instead of methods, but this could theoretically be improved by directly adding methods to `[MaybeUninit<T>]`, which is doable in the standard library. Either way, this effectively requires maintaining multiple copies of these methods for slices and non-slices, which can make documentation, discovery, and maintenance more difficult.

Given by the fact that the `MaybeUninit` type has been stable since Rust 1.36.0 (July 2019), and these equivalent slice methods have still not been stabilized after over three years, it feels reasonable to say that these methods are not expected to be stabilised as-is. For a more fair comparison, const generics was stabilised in Rust 1.51.0, which was only about a year and a half ago; it feels like, if these APIs were expected to be stabilised as-is, it would have been done by now.

# Prior art
[prior-art]: #prior-art

The strongest precedent for supporting this kind of behaviour is rust-lang/rfcs#1789, which allowed unsized values inside `Cell`. Trait objects were similarly discussed for that issue, but the end result was that even if coercing `Cell<T>` to `Cell<dyn Trait>` was technically possible, none of the actual methods implemented for `Cell` could actually do anything with a value of this type.

While the tracking issues for the various `MaybeUninit` methods are spread about, here's the best list I could put together:

* rust-lang/rust#96097 (`uninit_array` and `array_assume_init`)
* rust-lang/rust#63569 (`slice_assume_init_ref` and `slice_assume_init_mut`
* rust-lang/rust#93092 (`slice_as_bytes` and `slice_as_bytes_mut`)
* rust-lang/rust#79995 (`write_slice` and `write_slice_cloned`)

There's also an open ACP for revamping some of these methods as well:

* rust-lang/libs-team#122

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The primary unresolved question of this RFC is what methods should be exposed once unsized `MaybeUninit` is allowed. Right now, allowing unsized `MaybeUninit` is considered a big enough philosophical decision to warrant an RFC, but these specific methods can be hashed out through the usual libs changes processes, like opening PRs for libstd changes directly or creating API Change Proposals for the libs team.

# Future possibilities
[future-possibilities]: #future-possibilities

* `Box<MaybeUninit<[T]>>` could potentially replace the internal `RawVec<T>` type used in the standard library in a way that could potentially be exposed as a standard API.
* An `IndexMut` implementation for `MaybeUninit<[T]>` could help replace the existing methods for `MaybeUninit` slices.
