- Feature Name: `macro_attributes`
- Start Date: 2016-02-11
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add support for user-defined attributes in the form of macros.

# Motivation
[motivation]: #motivation

Currently, Rust has two very useful language constructs that are *entirely* uncustomisable in stable code: attributes and (as a notable subset of these) derivations.

## Derivations

The utility of user-defined derivations is obvious.  Whilst one can automatically derive implementations of various standard traits, the same cannot be said for any user-defined traits.  One of the persistent issues with adoption of the `serde` crate is that the `Deserialize` and `Serialize` traits *cannot* be derived on stable compilers without the use of the extremely hacky and slow `syntex` crate.

This is to say nothing of various traits within the standard library itself that have relatively obvious, common implementations.  For example, there is the obvious "element wise" implementation for arithmetic traits.  In addition, newtype structs can benefit from obvious "forwarding" implementations of many traits.

It is possible to implement these derivations as macros, and invoke them manually, but this generally requires duplicating the original type definition for *each* invocation.  The `custom_derive` crate exists to automate this, but it prevents composition with other macros, and is susceptible to "macro recursion limit blowout" problems.

Having a stable mechanism for defining derivations will improve the usability of traits throughout the ecosystem.

## Attributes

The case for attributes is slightly more abstract, but still reasonably compelling.  A derivation carries the expectation that it generates something mechanically from an item definition, without modifying the item itself.  However, modifying the item can sometimes be useful.  Consider wrapping a function with logging or tracing code, or transforming an arithmetic expression to use wrapping or constant time operations.

Another potential use is to encode inert metadata into a crate's AST.  This could be useful for source processing tools such as `rustfmt` and `clippy`, which may benefit from users being able to add annotations to the source which are meaningless to the compiler itself.  In fact, this pattern *already* appears in `serde`: its compiler plugin recognises certain annotations which exist purely to carry additional information for the trait derivations.

## Stable Implementation

Both custom derivations and custom attributes *could* be implemented in stable macros.  As mentioned previously, the `custom_derive` crate already does so for derivations, and there is no fundamental reason why it could not be extended to handle attributes.  However, it *does* introduce some issues:

* `custom_derive!` is incompatible with `#[cfg]` attributes, meaning it cannot be used with conditional compilation.

* Macros which need to parse their input are *generally* incomposable, since a macro cannot destructure the output of another macro.

* `custom_derive!` (and any macro like it) is necessarily complicated, slow, and consumes a significant amount of the macro recursion limit.

## Stable Procedural Macros

There are plans afoot to stabilise a procedural macro interface for Rust.  Adding support for user-defined attributes and derivations *now* means that procedural macros can take advantage of the same support upon introduction.

# Detailed design
[design]: #detailed-design

In summary, the proposed changes are:

- Extend the syntax for meta items to include a "macro invocation" form.
- Permit and define the processing of attributes containing a top-level macro invocation meta item.
- Permit and define the processing of arguments to the `#[derive]` attribute which are macro invocation meta items.

## Meta Syntax

The rules for meta syntax should be extended with an additional variant:

```ebnf
meta_item : ident [ '=' literal
                  | '(' meta_seq ')'
                  | '!' delimited_tt ? ] ? ;
```

This allows for "macro" meta items, optionally followed by a single delimited token tree.

Technically, this means that constructs such as `#[cfg(panic!)]` would be syntactically valid, but would be *semantically* invalid.

### Examples

- `foo!`
- `bar!(baz)`
- `qux! { she sells sea shells from seven sheds in Swindon. }`
- `wimbol!(all(meta, items, are="also valid"), coincidentally)`

## Macro Attributes

Macro attributes are attributes which consist of a single, top-level macro meta item.  They are processed during macro expansion as follows:

- If the macro meta item *does not* have a delimited `tt`, an empty pair of parentheses is appended.

  This exists to simplify parsing on the macro side.

  For example, `#[foo!]` would be treated as though it had been written `#[foo!()]`.

- The item the attribute is attached to, and all attributes lexically *after* the macro attribute being expanded are converted into a sequence of non-NT token trees, named `item_tokens` below.

- The macro attribute is immediately replaced with the result of expanding `name! { tt item_tokens }`.  Attributes *before* the macro attribute are then attached to the resulting item(s), duplicating as necessary.

- The macro attribute may *only* expand to zero or more items.

This allows macro attributes to arbitrarily modify items during expansion.

### Inner Attributes

Inner attributes, if supported, should follow the same algorithm as outer attributes.  This can be done by promoting inner attributes to outer attributes.  In the following example, assume that the `foo!` attribute is being processed:

```rust
#[outer]
mod yule {
    #![inner]
    #![foo!]
    #![bar!]

    ...
}

// Interpreted as:

#[outer]
#[inner]
#[foo!]
#[bar!]
mod yule {
    ...
}

// Interpreted as:

#[outer]
#[inner]
foo! {
    ()
    #[bar!]
    mod yule {
        ...
    }
}
```

### Examples

Items can be modified by attributes:

```rust
// Assuming feature `s_is_not_pub` is *not* set.

macro_rules! as_item { ($i:item) => { $i } }

#[cfg(not(feature="s_is_not_pub"))]
macro_rules! maybe_pub {
    (() $(#[$attrs:meta])* struct $($tail:tt)*) => {
        as_item! { $(#[$attrs])* pub struct $($tail)* }
    };
}

#[cfg(feature="s_is_not_pub")]
macro_rules! maybe_pub {
    (() $($tail:tt)*) => {
        as_item! { $($tail)* }
    };
}

#[maybe_pub!]
struct S;

// Is interpreted as:

maybe_pub! {
    ()
    struct S;
}

// Expands to:

pub struct S;
```

Repetitive documentation thunks can be inserted:

```rust
macro_rules! maybe_illegal {
    (() $($tail:tt)*) => {
        as_item! {
            /// **Note**: this item may be illegal in jurisdictions with
            /// strong anti-"dressing dogs up in funny costumes" laws.
            $($tail)*
        }
    };
}

/**
Put a dog into a sailor outfit.
*/
#[maybe_illegal!]
fn sailor_fuku(dog: &mut Dog) { ... }

// Is interpreted as:

#[doc="
Put a dog into a sailor outfit.
"]
maybe_illegal! {
    ()
    fn sailor_fuku(dog: &mut Dog) { ... }
}

// Expands to:

#[doc="
Put a dog into a sailor outfit.
"]
#[doc="**Note**: this item may be illegal in jurisdictions with"]
#[doc="strong anti-\"dressing dogs up in funny costumes\" laws."]
fn sailor_fuku(dog: &mut Dog) { ... }
```

Wrap a function to log all invocations.

```rust
macro_rules! trace_fn {
    (
        ($level:ident)
        $(#[$($attrs:tt)*])*
        fn $name($($arg_names:ident: $arg_tys:ty),*) -> $res:ty {
            $($body:tt)*
        }
    ) => {
        $(#[$($attrs)*])
        fn $name($($arg_names: $arg_tys),*) -> $res {
            log!($level,
                concat!(stringify!($name),
                    "(",
                    $(stringify!($arg_names), ": {:?}, ",)*
                    ")"),
                $($arg_names,)*);
            let result = as_expr!({ $($body)* });
            log!($level,
                concat!(stringify!($name), "(..) -> {:?}"),
                result);
            result
        }
    };
}

/// All I need is a bumping beat.
#[trace_fn!(info)]
#[maybe_pub!]
fn do_your_thing(my: &Body) -> Result<(), Error> {
    my.make("swing")
}

// Is interpreted as:

#[doc="All I need is a bumping beat."]
trace_fn! {
    (info)
    #[maybe_pub!]
    fn do_your_thing(my: &Body) -> Result<(), Error> {
        my.make("swing")
    }
}

// Expands to:

#[doc="All I need is a bumping beat."]
// NOTE: Attribute above is *not* from expansion.
#[maybe_pub!]
fn do_your_thing(my: &Body) -> Result<(), Error> {
    log!(info, ...);
    let result = { my.make("swing") };
    log!(info, ...);
    result
}

// Is interpreted as:

#[doc="All I need is a bumping beat."]
maybe_pub! {
    ()
    fn do_your_thing(my: &Body) -> Result<(), Error> {
        log!(info, ...);
        let result = { my.make("swing") };
        log!(info, ...);
        result
    }
}

// Expands to:

#[doc="All I need is a bumping beat."]
pub fn do_your_thing(my: &Body) -> Result<(), Error> {
    log!(info, ...);
    let result = { my.make("swing") };
    log!(info, ...);
    result
}
```

## Macro Derivations

The `derive` attribute shall support macro meta items.  They are processed during `derive` expansion as follows:

- If the macro meta item *does not* have a delimited `tt`, an empty pair of parentheses is appended.

  For example, `#[derive(foo!, bar!(), baz!(..))]` is interpreted as though it was written `#[derive(foo!(), bar!(), baz!(..))]`.

- The item the attribute is attached to, excluding all attributes, is converted into a sequence of non-NT token trees, named `item_tokens` below.

- The expansion of `name! { tt item_tokens }` is emitted.

- The derivation macro may *only* expand to zero or more items.

### Examples

A macro to derive a conversion implementation for newtypes.

```rust
macro_rules! From {
    (() $(pub)* struct $name($base_ty:ty);) => {
        impl ::std::convert::From<$base_ty> for $name {
            fn from(v: $base_ty) -> $name {
                $name(v)
            }
        }
    };

    (($from_ty:ty) $(pub)* struct $name($base_ty:ty);) => {
        impl ::std::convert::From<$from_ty> for $name {
            fn from(v: $from_ty) -> $name {
                $name(v.into())
            }
        }
    };
}

/// Wraps an `u16`.
#[derive(From!, From!(u8))]
struct Wrapped(u16);

// Is interpreted as:

#[doc="Wraps an `u16`."]
struct Wrapped(u16);

From! { () struct Wrapped(u16); }
From! { (u8) struct Wrapped(u16); }

// Expands to:

#[doc="Wraps an `u16`."]
struct Wrapped(u16);

impl ::std::convert::From<u16> for Wrapped {
    fn from(v: u16) -> Wrapped {
        Wrapped(v)
    }
}

impl ::std::convert::From<u8> for Wrapped {
    fn from(v: u8) -> Wrapped {
        Wrapped(v.into())
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

- A meteorite is heading toward Earth and everyone would rather be with their families and loved ones in their final moments.  Disregard this RFC, acquire solace.

# Alternatives
[alternatives]: #alternatives

- *Restricting to compiler plugins* - make registering attributes and derivations *require* the use of a compiler plugin.  I cannot think of a compelling reason for preventing non-procedural macros from being usable in this context.

- *Namespacing/annotating macros* - require `#[attribute]` and `#[derivation]` attributes on macro definitions, in order to segregate them from "regular" macros.  This is similar to how, internally, derivations use name mangling (*e.g.* `#[derive(Clone)]` maps to `#[derive_Clone]`).

  My fear is that this will cause unnecessary confusion.

- *Drop the `!`* - perhaps combined with macro namespacing/annotating.  This provides a more "natural" syntax.

  The downside is that it means introducing new attributes or derivations in the standard library and/or compiler could potentially be a breaking change, due to name collisions.

# Unresolved questions
[unresolved]: #unresolved-questions

**TODO**: What parts of the design are still TBD?
